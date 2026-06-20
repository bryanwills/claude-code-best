# cloud-artifacts

独立的 Cloudflare Worker + R2 服务，用于托管 HTML artifact。

服务端（CLI / RCS 后台）通过单一 bearer token 上传 HTML，得到一个公开可访问的 CDN URL。**POST 上传和 GET 访问都走 Worker**，URL 形如 `https://<worker-domain>/<ttl-prefix>/<id>.html`，文件到期由 R2 lifecycle rule 自动删除。

## 架构

```
客户端  --POST /upload-->  Worker (鉴权 + 校验 + R2 put)  --返回 {id, url, expiresAt}
客户端  --GET html------->  Worker (R2 get → text/html)   --直出 html，带 Cache-Control: max-age=86400
                           ↑
                           R2 lifecycle rule: 7d/ 删 7 天，30d/ 删 30 天
```

- Worker 处理 `POST /upload` 和 `GET /<prefix>/<id>.html`
- R2 key 形如 `7d/<id>.html` 或 `30d/<id>.html`
- URL 形如 `https://<worker-domain>/7d/<id>.html`，hash 本身是秘密（21 字符 nanoId，126 bit 熵）

## 部署

前置：本机已 `npx wrangler login` 登录目标 Cloudflare 账号，且账号下有一个用于 Worker custom domain 的域名（zone）。

```bash
cd packages/cloud-artifacts
bun install                          # 在 monorepo 根执行也行（workspace 自动识别）

cp .dev.vars.example .dev.vars       # 填本地 dev 用的 TOKEN（仅 wrangler dev 读）
bun run setup                        # 创建 bucket + 加 lifecycle rule + 设生产 TOKEN secret

# 绑定 Worker custom domain（在 Cloudflare dashboard）：
#   Workers & Pages > cloud-artifacts > Settings > Domains & Routes > Add > Custom Domain
#   填入你的 domain（如 artifacts.example.com）

# 改 wrangler.toml 中 [vars] PUBLIC_URL 为上一步的 domain（如 https://artifacts.example.com）

bun run deploy
```

## API

### `POST /upload`

| Header / Query | 必填 | 说明 |
|----------------|------|------|
| `Authorization: Bearer <TOKEN>` | 是 | 与 Worker secret `TOKEN` 完全相等 |
| `Content-Type: text/html` | 是 | 不接受其他类型 |
| `?ttl=7\|30` | 否 | 默认 7，**只允许 7 或 30**（与 R2 lifecycle prefix 对应） |
| `?hash=<custom-id>` | 否 | 自定义 ID，校验 `^[A-Za-z0-9_-]{1,128}$`；指定时覆盖同 ID 旧版本 |
| body | 是 | 原始 HTML（`--data-binary @file.html`），≤10MB |

成功 200：

```json
{
  "id": "V1StGXR8_Z5jdHi6B-myT",
  "url": "https://<worker-domain>/7d/V1StGXR8_Z5jdHi6B-myT.html",
  "expiresAt": "2026-06-27T10:00:00.000Z"
}
```

错误（统一 `{ "error": "<code>" }`）：

| 状态码 | error code | 触发条件 |
|--------|------------|----------|
| 400 | `invalid_ttl` | `ttl` 非 7 或 30 |
| 400 | `invalid_hash` | `hash` 不匹配 `^[A-Za-z0-9_-]{1,128}$` |
| 401 | `unauthorized` | 缺 Authorization / token 不匹配 |
| 404 | `not_found` | 非 `/upload` 路径或 GET 路径不匹配 `/<7d\|30d>/<id>.html` |
| 413 | `payload_too_large` | body > 10MB |
| 415 | `unsupported_media_type` | Content-Type 非 `text/html` |

### `GET /<ttl-prefix>/<id>.html`

由 Worker 处理：解析路径 → R2 get → 返回 `text/html; charset=utf-8` + `Cache-Control: public, max-age=86400`。任何人拿到 URL 都可访问，hash 即秘密。

`ttl-prefix` 只能是 `7d` 或 `30d`（其他路径返回 404）。

## 示例

```bash
# 上传（默认随机 ID + 7 天）
echo '<h1>hello</h1>' > /tmp/t.html
curl -X POST "https://<worker-domain>/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: text/html" \
  --data-binary @/tmp/t.html
# -> {"id":"V1StGXR8_Z5jdHi6B-myT","url":"https://<worker-domain>/7d/V1StGXR8_Z5jdHi6B-myT.html","expiresAt":"..."}

# 自定义 hash + 30 天（再次上传同 hash 覆盖）
curl -X POST "https://<worker-domain>/upload?ttl=30&hash=my-report" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: text/html" \
  --data-binary @/tmp/report.html

# 访问（公开 URL，走 Worker → R2）
curl "https://<worker-domain>/7d/V1StGXR8_Z5jdHi6B-myT.html"
```

## 覆盖语义

指定 `?hash=` 时：

1. 校验 hash 字符集（`^[A-Za-z0-9_-]{1,128}$`）
2. 删除 `7d/<hash>.html` 和 `30d/<hash>.html` 两个 key（R2 delete 不存在的 key 不报错，零成本）
3. 按 `?ttl=` 写入新 key
4. 返回新的 `expiresAt`

不指定 `?hash=` 时：用 `nanoid(21)` 随机 ID，几乎不可能碰撞，不做碰撞检查。

## TTL 落地

R2 不支持 per-object TTL，本服务用 prefix + R2 lifecycle rule 模拟：

- bucket 配两条 rule：prefix `7d/` 删 7 天前对象、prefix `30d/` 删 30 天前对象
- 由 `scripts/setup.sh` 调 `wrangler r2 bucket lifecycle add` 自动配置
- Worker 完全不参与过期处理，零额外代码
- 因此 `?ttl=` 只能取 `7` 或 `30`，对应这两个 prefix（其他值会写到无 lifecycle 的 prefix → 永久存储，故拒绝）

## 本地开发

```bash
cp .dev.vars.example .dev.vars
# 编辑 .dev.vars 填 TOKEN

bun run dev                          # wrangler dev，启动本地 Miniflare + 本地 R2 模拟
curl -X POST "http://localhost:8787/upload" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: text/html" \
  --data-binary @/tmp/t.html
```

## 测试

`scripts/test.sh` 覆盖 7 个错误用例 + 3 个成功用例 + R2 写入验证：

```bash
WORKER_URL=https://<worker-domain> \
TOKEN=<your-token> \
bash scripts/test.sh
```

## 依赖

- `wrangler` ^4 — Cloudflare Workers CLI
- `nanoid` ^5 — ID 生成（纯 ESM，Worker 兼容）

## 不被主 CLI 引用

这是独立 Cloudflare Worker 服务，类似 `packages/remote-control-server/` 的定位。Monorepo 根 `package.json` 的 `workspaces: ["packages/*", ...]` 自动识别本包，但主 CLI 不会 import 它。
