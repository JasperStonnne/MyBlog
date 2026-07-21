---
title: 协同编辑器架构分析
slug: collab-editor-architecture
description: 记录一个 Yjs + Go + PostgreSQL 协同 Markdown 编辑器各模块的已有实现与待完善之处
date: 2026-05-21T00:00:00+08:00
draft: true
image: cover.svg
tags:
  - 协同编辑
  - Yjs
  - CRDT
  - WebSocket
categories:
  - 项目记录
---

## 项目概况

一个完整的实时协同 Markdown 编辑器，技术栈：前端 TipTap + Yjs，后端 Go（Gin），数据库 PostgreSQL。

实现目标包括：多人协同编辑、自建图床、通知推送、JWT 鉴权、四级权限系统。

## 1. Markdown 同步后端

**已有：**

- 前端 Yjs + y-websocket，TipTap 绑定 Y.Doc。
- 后端 Go 做 CRDT-blind relay（`hub.go` + `manager.go`），不解码 CRDT，只广播二进制帧。
- PostgreSQL `documents.ydoc_state` 存 CRDT 快照，`content` 列存纯 Markdown 供只读/导出。
- 保存流程：`Y.encodeStateAsUpdate(doc)` -> REST `/documents/:id` -> 写库 -> 通知 Hub 清空 update log。

**缺口：**

REST 保存路径本质是 last-writer-wins，后续需要改为 `mergeUpdates(old, new)` 合并策略，并追加 `document_versions` 历史记录。

## 2. 自建图床

**已有：**

- `backend/internal/models/media.go`、`handlers/media.go`、`service/media.go` 上传接口。
- 文件落本地 `backend/uploads/`，`mime_type` 字段已存。
- 前端 `frontend/src/extensions/image.ts` 是 TipTap 图片扩展。

**缺口：**

- 视频插入扩展可能未实现。
- drag-drop -> 上传 API -> 插入节点的串联需要验证。
- 本地磁盘存储缺抽象接口，换 S3/MinIO 时改动大。
- 需要补文件去重、MIME sniffing、路径穿越防御、上传后重命名为 UUID 等逻辑。

## 3. 通知系统

**已有：**

- `models/notification.go` 定义了 `NotifDocumentUpdated`、`ConflictDetected`、`ConflictResolved`。
- `handlers/notification.go`、`service/notification.go` 和 WebSocket 推送通道。

**缺口：**

- 缺 `document_subscriptions(user_id, doc_id)` 表，现在通知按文档权限持有者推，而不是主动订阅者。
- 前端触达可以先做 WebSocket 推送 + `new Notification()`；完整版再考虑 Web Push + service worker。

## 4. 登录鉴权

**已有：**

- JWT access + refresh pair（`backend/internal/auth/jwt.go`），HS256 签名。
- 密码存储待确认是否使用 bcrypt。

**待确认/完善：**

- `frontend/src/store/auth.ts` 中 token 存储位置。
- 推荐：access token 存内存，refresh token 存 HttpOnly Cookie。
- 需要继续分析 XSS 与 CSRF 风险。

## 5. 权限系统

**已有：**

- 四级权限：`manage`、`edit`、`readable`、`none`。
- `PermissionGroup` 支持 group 粒度授权。
- `handlers/document.go:SetPermission` 已暴露接口。
- Workspace 层有 membership。

**缺口：**

- 题目要求三级身份 manager、组长、组员，现在只有 owner vs member 二元，需扩枚举或用 `PermissionGroup` 建模“组长”标志位。
- 需要确认 `handlers/admin.go` 是否暴露 `POST/DELETE /groups`、`POST /groups/:id/members`。
- `auth/middleware.go` 需要有统一的 `RequirePermission(docID, PermEdit)` 守卫，避免各 handler 手写权限检查。
