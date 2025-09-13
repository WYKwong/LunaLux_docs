# S3 CORS 设置与本地开发上传回退方案

## 目标
本地开发或多域名场景可能触发 S3 CORS 阻拦，导致前端使用预签名 URL 上传失败。为提升可用性，我们提供了服务端中转回退端点。

## 推荐 S3 CORS 配置（示例）
在目标 Bucket 的 CORS 规则中加入（按需收紧）：

```xml
<CORSConfiguration>
  <CORSRule>
    <AllowedOrigin>http://localhost:3000</AllowedOrigin>
    <AllowedOrigin>http://localhost:3001</AllowedOrigin>
    <AllowedOrigin>http://localhost:3002</AllowedOrigin>
    <AllowedHeader>*</AllowedHeader>
    <AllowedMethod>PUT</AllowedMethod>
    <AllowedMethod>GET</AllowedMethod>
    <AllowedMethod>HEAD</AllowedMethod>
    <ExposeHeader>ETag</ExposeHeader>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
  </CORSRule>
</CORSConfiguration>
```

注意：生产环境应写入明确来源（域名/协议），避免使用 `*`。

## 回退端点
- `POST /api/character/upload-direct`
  - 请求：`multipart/form-data`，字段 `file`（必填）、`filename`（可选）
  - 鉴权：与其他角色 API 一致，需有效会话头。
  - 行为：服务器接收文件后，通过 SDK 直接写入 S3，返回 `{ key }`。

## 前端行为
`function/character/publish.ts`：
1. 尝试 `/api/character/presign-upload` + 预签名 PUT 上传；
2. 若失败（常见于本地缺少 CORS 设置），自动回退到 `/api/character/upload-direct`；
3. 随后调用 `/api/character/finalize-publish` 完成记录写入与缩略图生成（仅保存名称、语言、图片URL；不保存元数据）。


