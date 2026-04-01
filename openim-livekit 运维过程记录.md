# openim-livekit 运维过程记录

本文记录本次为满足 `openim-electron-demo` RTC 测试所执行的运维过程、验证命令和常见问题处理。

## 目标

- 让 LiveKit 在本机可用，满足：
  - `7880` 可访问（HTTP/WS）
  - `7881` 可监听（RTC TCP）
- 使 `openim-chat` 的 `/user/rtc/get_token` 下发的 `serverUrl` 可达

## 前置条件

- `openim-chat` 已启动，且 `config/chat-rpc-chat.yml` 中 `liveKit` 配置与 `openim-chat/livekit/livekit.yaml` 一致
- 本机已安装 Go，能在 `openim-livekit` 仓库执行 `go run`

## 本次实际过程

## 1) 初始检查

- 查看端口：
  - `ss -lntp | awk 'NR==1 || /:7880|:7881/'`
- 结果：未监听（当时 LiveKit 未运行）

## 2) Docker 方案尝试（失败）

- 尝试命令（示例）：
  - `docker run -d --name openim-livekit -p 7880:7880 -p 7881:7881 -p 50000-60000:50000-60000/udp -v /home/administrator/interview-quicker/openim/openim-chat/livekit/livekit.yaml:/livekit.yaml:ro livekit/livekit-server --config /livekit.yaml --bind 0.0.0.0 --node-ip=127.0.0.1`
- 现象：
  - 容器处于 `Created`，`docker start/rm/logs` 操作卡住
  - 端口仍未监听
- 结论：
  - 本机当前 Docker 路径不稳定，改走源码直启

## 3) 源码直启方案（成功）

- 工作目录：
  - `/home/administrator/interview-quicker/openim/openim-livekit`
- 启动命令：
  - `go run ./cmd/server --config "/home/administrator/interview-quicker/openim/openim-chat/livekit/livekit.yaml" --bind 0.0.0.0 --node-ip=127.0.0.1`
- 日志关键行（成功标志）：
  - `starting LiveKit server`
  - `portHttp: 7880`
  - `rtc.portTCP: 7881`

## 4) 启动后验证

- 端口检查：
  - `ss -lntp | awk 'NR==1 || /:7880|:7881/'`
- 健康检查：
  - `curl --noproxy '*' -sS -m 3 "http://127.0.0.1:7880"`
  - 预期返回：`OK`
- 与 openim-chat 联调检查：
  - `POST http://127.0.0.1:10008/user/rtc/get_token`（带 `chatToken`）
  - 预期：`errCode=0` 且返回 `serverUrl=ws://127.0.0.1:7880`

## 结果

- LiveKit 已可用，`7880/7881` 可达
- `openim-electron-demo` 的 RTC 测试前置条件满足

## 常用运维命令（复用）

- 启动（源码）：
  - `go run ./cmd/server --config "/home/administrator/interview-quicker/openim/openim-chat/livekit/livekit.yaml" --bind 0.0.0.0 --node-ip=127.0.0.1`
- 检查端口：
  - `ss -lntp | awk 'NR==1 || /:7880|:7881/'`
- 健康探测：
  - `curl --noproxy '*' -sS -m 3 "http://127.0.0.1:7880"`
- 停止（源码进程）：
  - `pkill -f "cmd/server --config .*livekit.yaml" || true`

## 备注

- 若后续 Docker 环境恢复稳定，可再切回容器化部署。
- 若部署在非本机，请同步更新：
  - `openim-chat/config/chat-rpc-chat.yml` 的 `liveKit.url`
  - `openim-electron-demo/.env` 的 `VITE_BASE_HOST`
