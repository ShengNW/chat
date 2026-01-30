# Router 与 WebDAV 集成说明

本文档说明当前 Chat 如何通过 UCAN 同时集成 Router 与 WebDAV，以及关键调用链路与配置点。

## 集成目标

- 通过一次钱包授权（UCAN Root），同时访问多个后端（Router + WebDAV）。
- Router 提供 OpenAI-compatible API；WebDAV 提供存储与配额服务。
- Next.js 同时承担前端与 API 代理层，统一跨域与鉴权。

## 调用链路

```mermaid
flowchart TB
  subgraph Browser["浏览器"]
    UI["Next.js 前端\n- 钱包连接\n- Root UCAN\n- Invocation UCAN"]
  end
  subgraph Proxy["Next.js API 代理"]
    AUTH["/api/v1/public/auth/*"]
    WEBDAVQ["/api/v1/public/webdav/quota"]
    WEBDAVSYNC["/api/webdav/*"]
  end
  subgraph Backends["后端服务"]
    ROUTER["Router\nOpenAI-compatible"]
    WEBDAV["WebDAV\nStorage/Quota"]
  end

  UI -->|"Authorization: Bearer UCAN"| AUTH
  UI -->|"Authorization: Bearer UCAN"| WEBDAVQ
  UI -->|"Authorization: Bearer UCAN"| WEBDAVSYNC
  AUTH --> ROUTER
  WEBDAVQ --> WEBDAV
  WEBDAVSYNC --> WEBDAV
```

## Router 集成

- **入口**：`app/client/platforms/openai.ts` 在请求 Router 相关接口时生成 Invocation UCAN。
- **请求头**：`Authorization: Bearer <UCAN>`。
- **代理路径**：`/api/v1/public/auth/*`，并限制允许的后端路径（白名单）。
- **受众 (audience)**：优先使用 `NEXT_PUBLIC_ROUTER_UCAN_AUD`，未设置时自动推导 `did:web:<router-host>`。

## WebDAV 集成

- **配额接口**：`app/plugins/webdav.ts` 使用 `authUcanFetch` 调用 `/api/v1/public/webdav/quota`。
- **同步接口**：`/api/webdav/*` 负责 WebDAV 文件同步，限制可用方法与目标路径，避免 SSRF。
- **请求头**：配额代理使用允许头白名单，仅透传必要头部。
- **受众 (audience)**：优先使用 `NEXT_PUBLIC_WEBDAV_UCAN_AUD`，未设置时自动推导 `did:web:<webdav-host>`。

## UCAN 会话与本地存储

- Root UCAN 与 Session 保存在 IndexedDB：`yeying-web3 / ucan-sessions`。
- 关键状态缓存于 `localStorage`：
  - `currentAccount`
  - `ucanRootExp`
  - `ucanRootIss`
- 每次请求按后端生成 Invocation UCAN，做到“一次授权，多后端访问”。

## 关键配置项

- `ROUTER_BACKEND_URL`: Router 后端地址（必填）
- `WEBDAV_BACKEND_URL`: WebDAV 后端地址（必填）
- `NEXT_PUBLIC_UCAN_RESOURCE`: 默认能力 resource（如 `profile`）
- `NEXT_PUBLIC_UCAN_ACTION`: 默认能力 action（如 `read`）
- `NEXT_PUBLIC_ROUTER_UCAN_AUD`: Router audience（可选）
- `NEXT_PUBLIC_WEBDAV_UCAN_AUD`: WebDAV audience（可选）

## 安全要点

- Router 代理使用路径白名单，拒绝非授权路由。
- WebDAV 同步代理限制方法与目标路径，避免 SSRF。
- 配额代理使用允许头白名单，避免透传敏感头。
- UCAN `aud` 必须与后端配置保持一致。

