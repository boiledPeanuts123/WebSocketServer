# WebSocket 实现分析（`WebSocketServer`）

## 1. 总体架构

- `MyWebSocketServer` 负责监听 TCP 连接，并在连接建立后创建 `BusinessSession`（业务层会话）。
- `BusinessSession` 继承 `MyWebSocketSession`，只重写业务回调（`onConnect`/`onMessage`），底层握手、帧编解码由 `MyWebSocketSession` 负责。
- `MyWebSocketSession` 的生命周期绑定在 `TcpConnection` 上：
  - 握手前：按 HTTP 请求头解析并升级。
  - 握手后：按 RFC6455 帧头读取并处理 `TEXT/BINARY/PING/PONG/CLOSE`。

## 2. 数据流与关键路径

### 2.1 连接建立

1. `MyWebSocketServer::onConnection` 在 `conn->connected()` 为真时创建 `BusinessSession`。
2. 将 `TcpConnection` 的消息回调设置为 `MyWebSocketSession::onRead`。
3. 将 session 放入服务器维护的 `m_listSessions` 链表，连接断开时在 `onClose` 中移除。

### 2.2 握手流程

在 `MyWebSocketSession::onRead` 中，`m_bUpdateToWebSocket == false` 时走握手逻辑：

1. 在 buffer 中查找 `\r\n\r\n` 作为 HTTP 请求结束标记。
2. 解析请求行（必须是 `GET ... HTTP/1.1`）。
3. 校验部分必需头（`Connection/Upgrade/Host/Origin/Sec-WebSocket-Key`）。
4. 调用 `WebSocketHandshake::generate` 生成 `Sec-WebSocket-Accept`。
5. 发送 `101 Switching Protocols` 响应并置 `m_bUpdateToWebSocket=true`，随后调用 `onConnect()`。

### 2.3 帧解析流程

`decodePackage` 处理 WebSocket 帧：

1. 解析 FIN/opcode/mask/payload length。
2. 支持 7-bit、16-bit（126）、64-bit（127）扩展长度。
3. 若掩码位存在，取 4 字节 mask key 对 payload 做 XOR 解掩码。
4. 对分片帧进行 `m_strParsedData` 累积；FIN 到达后调用 `processPackage`。

`processPackage` 当前行为：
- `CLOSE`：返回 false，触发关闭。
- `PING`：回 `PONG`。
- `TEXT/BINARY`：如果协商了压缩则 `inflate`，然后回调 `onMessage`（业务层默认 echo）。

### 2.4 发送流程

`send()` 会：

1. 如果 `m_bClientCompressed` 为真，先 deflate。
2. 构造服务器帧（不加 mask）。
3. 根据长度选择短帧 / 16 位扩展 / 64 位扩展。
4. 拼包后 `conn->send()`。

## 3. 优点

- **分层清晰**：传输层（TCP）/协议层（WebSocket）/业务层（BusinessSession）职责分离。
- **基础协议点覆盖**：握手、mask 解码、分片聚合、ping/pong 都有实现。
- **压缩支持**：实现了 `permessage-deflate` 的基础收发流程。

## 4. 主要问题与风险

1. **握手响应头拼写问题**：`Sec-Websocket-Accept`（小写 s）建议统一为 `Sec-WebSocket-Accept`，避免严格客户端兼容性问题。
2. **Header 校验大小写敏感**：当前 `Connection: Upgrade`、`Upgrade: websocket` 等使用严格大小写和精确值，实际 HTTP 头与 token 通常应大小写不敏感。
3. **请求头解析过于脆弱**：按 `:` 直接切分，遇到 header value 含 `:` 的情况会失败（例如某些 `Origin`/自定义头）。
4. **payloadLength 判定条件有逻辑错误**：`if (payloadLength <= 0 && payloadLength > 127)` 永远为 false（应为 `||` 或更合理边界检查）。
5. **未严格校验客户端必须 mask**：代码中有注释但未启用强校验，不符合 RFC（客户端发给服务端必须带 mask）。
6. **分片帧规则未完全校验**：仅做简单拼接，未严格校验首帧 opcode 与后续 continuation 的合法组合。
7. **线程模型与对象生命周期可读性一般**：`spSession.get()` 绑定成员函数回调依赖外部容器保活，虽然当前通过 `m_listSessions` 可保活，但后续维护容易误改出悬挂回调风险。
8. **时间头格式非 HTTP 标准**：`Date` 使用 `%Y%m%d%H%M%S`，不符合 RFC7231 推荐格式（应使用 IMF-fixdate）。
9. **`compress` 参数未生效**：`send(..., bool compress)` 入口参数基本未参与控制，实际依赖 `m_bClientCompressed`。

## 5. 改进建议（按优先级）

### P0（建议先做）
- 修复 `Sec-WebSocket-Accept` 响应头拼写。
- 修复 payload 长度的逻辑判断错误。
- 对 `Connection/Upgrade` 做大小写不敏感比较。
- 强制校验客户端帧 mask 位。

### P1
- 重构 HTTP 头解析，避免 value 中含 `:` 时误切分。
- 完善分片状态机（首帧、连续帧、FIN 结束一致性）。
- 将 close frame 按规范返回 close code/reason。

### P2
- 规范化 Date 响应头。
- 明确 `compress` 参数语义（按调用参数控制，还是按协商状态强制）。
- 增加协议层单测（握手、短帧、扩展长度、mask、分片、压缩）。

## 6. 结论

该实现已经具备可用的 WebSocket 服务端骨架，尤其适合“业务层快速接入”的场景；但在协议严谨性与兼容性方面仍有明显改进空间。若用于生产公网场景，建议至少完成 P0 项后再上线。
