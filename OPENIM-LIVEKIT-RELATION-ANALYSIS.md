# openim-livekit 与 openim-electron-demo 关系分析

`openim-livekit` 本质上是 LiveKit Server（WebRTC SFU 媒体服务）源码仓库，不是 OpenIM 的业务后端。  
它主要负责音视频流转发与 RTC 会话，不负责 OpenIM 的登录、好友、消息、会话等 IM 业务。

## openim-livekit 在做什么

- 提供实时音视频/数据通道能力（WebRTC SFU）。
- 通过 API Key/Secret + JWT 做 RTC 鉴权。
- 默认关键端口是：
  - `7880`（信令/RoomService）
  - `7881`（RTC over TCP）
  - `50000-60000/udp`（媒体流）
- 该目录看起来是上游 LiveKit 源码（`go.mod` 是 `github.com/livekit/livekit-server`），`install-livekit.sh` 也是官方安装脚本风格。

## 和 openim-electron-demo 的关系

`openim-electron-demo` 的链路是“三段式”：

- IM 相关：走 `open-im-server + openim-chat`
- RTC token：向 `openim-chat` 请求 `/user/rtc/get_token`
- 真正入会：前端 `LiveKitRoom` 用返回的 `serverUrl + token` 连接 LiveKit

也就是说：

- `openim-livekit` 只解决“音视频媒体服务”这一段
- 不能单独替代 `open-im-server / openim-chat`

## 是否能满足 openim-electron-demo

结论：可以满足其 RTC 依赖，但前提条件必须满足；对整个 demo 来说是“部分满足”。

### 能满足的部分

- 一对一音视频通话的 LiveKit 服务端能力（README 也写的是开源版 1v1）

### 不能单独满足的部分

- 登录、IM 消息、会话、通讯录等仍依赖 OpenIM 后端（`10001/10002/10008`）

### 必须对齐的条件

- `openim-chat` 的 `liveKit.url/key/secret` 与 LiveKit 配置一致
- LiveKit 实际可达（至少 `7880/7881`，以及 UDP 端口范围）
- Web 端需 `localhost` 或 HTTPS（浏览器策略）

仓库内 `ADAPTATION-ASSESSMENT.md` 也和这个判断一致：  
登录+IM 可测，RTC token 可拿到，但当时 LiveKit 端口未监听，所以 RTC 实连不满足。

---

后续可只做“分析版核对清单”（不改代码），逐项确认当前环境卡在：端口、防火墙、key/secret 不匹配，还是 `serverUrl` 不可达。
