# OpenIM + LiveKit 一页纸排障决策树

目标：`openim-electron-demo` 一对一音视频通话打通  
分层：客户端 → chat 发 token → livekit 连接 → 媒体传输

## 0. 先看现象属于哪一类

- A：登录/消息都不正常
- B：登录/消息正常，但“发起通话失败”
- C：能发起，接听后一直转圈/连不上
- D：已连上但听不到/看不到
- E：偶发断线、跨网络差、弱网差

## 1) 如果是 A：IM 基础层先修好

看点

- `VITE_WS_URL(10001)`, `VITE_API_URL(10002)`, `VITE_CHAT_URL(10008)` 是否可达
- `open-im-server / openim-chat` 是否 healthy

结论

- IM 不通时，RTC 不用查，先修 IM。

## 2) 如果是 B：token 层（chat）排查

触发信号

- 点击呼叫后马上失败，或提示获取 RTC 信息失败

检查

- `POST /user/rtc/get_token` 是否成功
- 返回是否有 `serverUrl`、`token`

常见报错 → 定位

- `401/403`：chat token 无效或过期（客户端登录态）
- `5xx`：chat 服务内部异常（配置或依赖）
- 返回空 `serverUrl/token`：chat 到 livekit 配置层问题

重点核对

- `chat-rpc-chat.yml` 里 `liveKit.url/key/secret`
- 与 `livekit.yaml` 的 `keys` 完全一致（大小写、空格都别错）

## 3) 如果是 C：连接层（LiveKit 信令）排查

触发信号

- 接听后 `connecting...` 很久，最后超时/断开

常见报错 → 定位

- `connection failed / websocket close`：`serverUrl` 不可达或端口/代理问题
- `unauthorized`：key/secret 不匹配（token 验签失败）
- `not found room`（少见）：room 生命周期/参数不一致

优先项

- `serverUrl` 是否是客户端可达地址（跨机别用 `127.0.0.1`）
- TCP `7880/7881` 是否开通
- 若走 `wss`，证书链与域名是否正确

## 4) 如果是 D：媒体层（ICE/UDP）排查

触发信号

- 已 `connected`，但无声音/黑屏/只有单向媒体

常见原因

- UDP 端口没放行：`7882/udp`、`50000-60000/udp`
- NAT/公网映射问题（`use_external_ip`、`node-ip`）
- 编解码兼容边缘问题（当前默认 VP9+VP8 备份通常够用）
- 设备权限问题（麦克风/摄像头授权）

快速判断

- 双方同内网能通，跨网不通 → 90% 是 UDP/NAT/TURN 问题

## 5) 如果是 E：稳定性层排查

触发信号

- 偶发卡顿、断断续续、切网后恢复差

排查方向

- 丢包/抖动高：网络质量与 UDP 路径
- 仅某地区差：跨地域 RTT，高延迟链路
- 高并发退化：LiveKit 资源、端口、带宽瓶颈
- 需要更强穿透：补 TURN（生产常见）

## 报错到层级速查表（最实用）

- 登录失败/拉不到会话 → IM层
- `get_token` 失败 → chat-token层
- `unauthorized`（LiveKit） → key/secret层
- 连接超时/WS断开 → 可达性/端口/代理层
- `connected` 但无媒体 → ICE/UDP/NAT层
- 同网好，跨网差 → NAT/TURN层

## 最小闭环验收（5步）

- 登录 + 收发消息正常
- `get_token` 成功（拿到 `serverUrl + token`）
- 双端可进入 `connected`
- 双向音频正常，再验证视频
- 挂断后状态正确回收

如果你愿意，我可以再给你一版“命令行检查模板”（每层 1-2 条命令）——你复制执行后，结果直接就能映射到这棵树上。
