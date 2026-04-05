# openim-livekit：音视频转发、职责与依赖

本文档说明 `openim-livekit`（LiveKit Server 源码树）如何转发音视频、承担哪些职责，以及主要技术与运行依赖；并补充 **RTP/RTCP/SRTP** 概念与 **「订阅推送中心」** 类比的准确边界。可与同目录下《openim-livekit 与 openim-electron-demo 关系分析.md》《OpenIM + LiveKit 一页纸排障决策树.md》对照阅读。

---

## 1. 定位与职责

- **本质**：开源 **LiveKit Server**（`go.mod` 模块名为 `github.com/livekit/livekit-server`），提供 **WebRTC SFU** 能力与数据通道。
- **不负责**：OpenIM 的登录、好友、消息、会话等 IM 业务；那些仍由 `open-im-server`、`openim-chat` 等承担。
- **负责**：房间与参与者、信令、媒体 RTP 的接收与按订阅关系转发、JWT/API Key 维度的 RTC 鉴权等。
- **典型端口**（部署时需放行）：
  - `7880`：信令 / RoomService（WebSocket 等）
  - `7881`：RTC over TCP
  - `50000–60000/udp`：媒体（UDP 范围以实际配置为准）

---

## 2. 音视频如何转发（SFU 模型）

LiveKit 采用 **SFU（Selective Forwarding Unit，选择性转发单元）**：

1. **信令**：客户端使用由业务侧签发的 **JWT** 连接服务，完成进房、SDP 交换，建立 **PeerConnection**。
2. **上行（发布）**：发布端将已编码的 **RTP**（如音频 Opus、视频 VP8/VP9/H.264 等）通过 WebRTC 送入服务端。
3. **服务端处理**：基于 **Pion WebRTC** 接收 RTP；在 `pkg/sfu` 中完成缓冲、**Simulcast/SVC 层选择**、带宽估计、RTCP/PLI 等控制面逻辑。**默认路径不是 MCU**：不做多路混音、也不对每路流做通用转码（录制/转推等属于 Egress/Ingress 等独立组件能力）。
4. **下行（订阅）**：为每个订阅者维护 **DownTrack**，将选中的 RTP（含必要的 RTP 头/序号/层相关处理）**写入该订阅者的 PeerConnection**。

房间侧在有人发布轨道后，会通知房间内其他已就绪参与者发起订阅（逻辑入口示例：`pkg/rtc/room.go` 中 `onTrackPublished` → `SubscribeToTrack`）。媒体面接收与发送在 SFU 中通过 `WebRTCReceiver`、`downtrack` 等与 Pion 衔接（例如 `pkg/sfu/receiver.go`、`pkg/sfu/downtrack.go` 中的 `TrackSender` / `WriteRTP` 等接口语义）。

**一句话**：音视频以 **RTP 在 SFU 上选路并转发**（外加层选择、拥塞与 RTCP 等），与 IM 消息链路分离。

---

## 3. RTP、RTCP、SRTP 是什么（和本仓库的关系）

**RTP（Real-time Transport Protocol）** 是 IETF 定义的 **实时媒体封包协议**（常见依据 **RFC 3550**），用于传输 **已编码** 的音视频数据：把码流切成包，带上 **序号（sequence）**、**时间戳（timestamp）**、**负载类型（payload type）** 等，让接收端能 **排序、抗抖动、按时间播放**，并可与别路流做 ** lip-sync** 等同步。

- **和 UDP 的关系**：RTP **通常运行在 UDP 之上**（低延迟、无连接）；WebRTC / LiveKit 媒体面 **多数是 UDP 上的 SRTP**（见下）。理论上 RTP 也可叠在其他传输上，但本栈里按 **UDP 媒体端口** 理解最贴切。
- **RTCP（RTP Control Protocol）**：与 RTP **成对使用**，传 **统计与控制**（丢包、抖动、发送报告等），供 **拥塞控制、码率/层决策、PLI/FIR** 等使用；SFU 里大量逻辑与 RTCP 反馈相关。
- **SRTP（Secure RTP）**：**加密的 RTP**（以及对应的 SRTCP）。WebRTC 里媒体几乎总是 **SRTP**，不是明文 RTP。
- **RTP 不负责编解码**：包里是 **压缩后的媒体分片**；编码/解码在终端（或 MCU）；**SFU 默认不解码**，主要在 **转发与改写 RTP 头/选层** 等层面工作。

**特点归纳**：面向实时、可容忍少量丢包以换低延迟；靠 **序号 + 时间戳** 处理乱序与抖动；可靠性由 **WebRTC 的上层机制**（NACK、FEC、PLC 等）与 **RTCP 反馈** 共同补齐。

---

## 4. 和「订阅推送中心」的类比（准确边界）

从 **业务直觉** 上，可以把 LiveKit SFU 看成一种 **媒体面的发布 / 订阅中心**：

- **发布**：一端把某路 **音视频轨道** 以 RTP（SRTP）形式送进 SFU。
- **订阅**：其他参与者声明要收哪路 track，SFU 为每个订阅者维护下行路径（如 **DownTrack**），把 **选中的 RTP 流转发** 到对应 PeerConnection，相当于对订阅者 **「推送」媒体包**。

需要收窄的说法（避免和 IM「消息推送」混淆）：

| 类比成立的部分 | 建议区分的部分 |
|----------------|----------------|
| 按 **订阅关系** 决定谁收到哪路媒体 | 不是通用 **消息总线**（不推聊天文本、业务 JSON） |
| 媒体 **常见是 UDP 上的 SRTP** | 整链路 **不只有 UDP**：信令在 **WebSocket（如 7880）**；ICE 可能走 **TCP（如 7881）** 或 **TURN（UDP/TCP/TLS）** |
| 多订阅者共享同一发布源（SFU 复制转发） | 推送的是 **实时 RTP 码流**，不是队列里的离线消息 |

**一句话**：可以理解为 **「谁订阅谁收」的实时媒体转发中心**；载体以 **SRTP/RTP（多为 UDP）** 为主，**信令与部分 RTC 路径仍可能走 TCP**。

---

## 5. 与 OpenIM / Electron Demo 的衔接（摘要）

- **Token**：业务后端（如 `openim-chat`）提供 RTC Token；客户端使用 **`serverUrl` + `token`** 连接 LiveKit。
- **配置对齐**：`liveKit.url`、`key`、`secret` 须与 LiveKit 服务端配置一致，且网络可达（含 UDP）。
- 更完整的“谁依赖谁、哪些能单独测”见：**《openim-livekit 与 openim-electron-demo 关系分析.md》**；排障流程见：**《OpenIM + LiveKit 一页纸排障决策树.md》**。

---

## 6. 主要依赖（Go 模块）

| 类别 | 代表依赖 | 作用简述 |
|------|----------|----------|
| WebRTC 栈 | `github.com/pion/webrtc/v4` 及 ICE、DTLS、SRTP、RTP、RTCP、SCTP、TURN 等相关 `pion/*` 包 | 建连、传输与 RTP 收发 |
| LiveKit 协议与工具 | `github.com/livekit/protocol`、`github.com/livekit/psrpc`、`github.com/livekit/mediatransportutil` | 信令/协议、RPC、媒体传输辅助 |
| 状态与扩展部署 | `github.com/redis/go-redis/v9` | 多节点等场景下的状态协调（按部署配置使用） |
| 可观测与日志 | `github.com/prometheus/client_golang`、`go.uber.org/zap` | 指标与结构化日志 |
| 其他 | `github.com/gorilla/websocket`、`github.com/google/wire` 等 | 信令通道、依赖注入等 |

完整列表以仓库根目录 **`go.mod`** / **`go.sum`** 为准。

---

## 7. 运行与镜像

- **Dockerfile**：多阶段构建，`CGO_ENABLED=0` 编译 **`./cmd/server`** 得到 `livekit-server` 单二进制，最终镜像基于 **Alpine**，无额外 FFmpeg 层——与“SFU 核心以转发为主”的定位一致。
- **本地/生产**：亦可直接运行二进制；生产参数与 Redis、端口、TURN 等见 [LiveKit 自托管文档](https://docs.livekit.io/home/self-hosting/)。

---

## 8. 相关路径速查

| 内容 | 路径 |
|------|------|
| 服务入口 | `cmd/server` |
| 房间与订阅逻辑 | `pkg/rtc/`（如 `room.go`、`participant.go`） |
| SFU 媒体转发核心 | `pkg/sfu/` |

---

*文档版本：与仓库当前结构一致；若升级 LiveKit 大版本，请以官方 Release 说明与 `go.mod` 为准同步本文。*
