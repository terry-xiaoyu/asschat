# ASSChat
**ASSChat** - A Super Simple Chat specification for Instant Messaging. It is fully based on MQTT v5.0.

### Why ASSChat?
在 IM 领域，经常用的两个协议是 `XMPP` 和 `MQTT`。

我很喜欢 MQTT，网络消耗很小，并且它设置了三个 QoS 级别，这一点比起 XMPP 是一个很大的进步。

在使用各类开源的 MQTT Broker 开发 IM 应用时，我们需要基于 MQTT 基础协议，设计一套上层聊天协议，并根据这套协议对 MQTT broker 做一点定制工作(通过开发插件)。这套上层协议就是 **ASSChat**.

我们的 ASSChat 基于 MQTT v5.0，因为 MQTT v3.1 不够完善。很多现实中的应用场景并没有包含在协议里。 v5.0 里做了许多有用的改进，比如：
- 返回值更加丰富，比如现在 PUBLISH 消息的 ACL (Access Control List) 规则如果匹配失败，Broker 可以返回一个 `0x87 Not authorized` 的 PUBACK。之前 PUBACK 里没有返回值。类似的 DISCONNECT 消息也可以设置错误码。
- 订阅主题时，可以设置 SUBSCRIBE 消息里的 `No Local` Option，阻止消息被转发到当前连接。这个让我们的订阅设计更加灵活。


### IoT 应用与 IM 应用的不同：
MQTT 协议里考虑 IoT 的应用场景居多，因为它本身就是为 IoT 设计的。

IM 应用与 IoT 应用给人感觉上非常类似，都是消息传递，但其实两者略有不同。具体来说，主要是两者对待 `离线状态` 的方式不同：

在 `IoT` 应用里，设计的时候其实是 **"Online First"** 的。我们尽量保持设备始终在线，当设备由于某些原因掉线时，持续尝试重连。当设备重新连上服务器后，服务器(视情况)可以把这个设备处于离线状态的时间段里，发给它的消息再次推送给它。另外，设备可以利用 `retained` 消息来间歇性地更新自己的状态，当另一个设备订阅那个表示 "设备的状态" 的主题时，最后一条(最新的)状态信息会立刻推送给它。这个方案里，**我们潜在地假设，IoT 设备处于离线状态时，没能收到它订阅的消息问题不大；或者假设在设备处于离线状态时，发给这个设备的消息量不会很多.** 这符合 IoT 的应用场景。

而在 `IM` 应用里，设计时应当 **"Offline First"**。IM 应用里，我们需要首先考虑不在线的场景，因为一个 IM 应用可能在大部分时间里都保持离线状态。当终端离线时，我们需要过推送通知的方式，唤起应用。当应用重新上线之后，应用需要一定的机制，从服务器获取那些它处于离线状态的时间段里，发给他的消息。**我们默认每一条 IM 消息都应该顺利送达，哪怕延时一段时间。**

MQTT 里一个设备连接到 broker 之后，broker 会用一个 `Session` 维护其在服务端的状态。这些状态包括：设备订阅了哪些主题及其 QoS、由于设备离线未发给设备的消息队列、由于未收到设备的 ACK 暂存在 broker 的消息队列。多数情况下，IoT 设备是设置 `Clean Session = True` 连接 broker 的，这样在连接断开时，broker 不会为其维护那些繁重的数据：订阅、离线消息。

MQTT 协议里的 `Clean Session` 是在 Client Connect 时设置的，也就是说 Broker 要么保存这个 Client 的所有订阅，要么一个也不保存。这不符合我们 IM 应用里的需求: 有些订阅信息的确需要持久化保存，比如那些作为聊天通道的订阅。服务器需要知道聊天消息应该投递到哪些终端去，如果终端不在线就帮他们缓存下来，以便终端在未来的某时刻来获取。但其他的订阅可能并不需要持久化保存，比如一个用户 A 订阅了另一个用户 B 的在线状态，A 离线的时候，他其实不再需要继续接收 B 上线、下线的通知。所以这个关于用户在线状态的订阅信息是不需要持久化保存的。

### ASSChat Spec 设计原则
- 完全基于 MQTT v5 基础协议。
- 不持久化 Session 和 订阅信息以保持 MQTT Broker 的轻量。
- 基于 MQTT PUBLISH 消息定制 `Channel` 协议。`Channel` 是用来聊天的通道。
- 在 ClientID 的语义基础之上，增加一套命名约定。

### Channel 设计概要
`Channel` 是 IM 用户聊天的通道，需要有建立、删除 Channel，发送消息到 Channel 等过程。
Channel 通过为 Channel 内的所有 Users 订阅相同的 MQTT Topic 来实现，附加离线消息的处理:
- 创建 Channel 时，服务器为 Channel 里的所有在线的 Users 订阅 Channel Topic。
- 当 Channel Topic 收到消息时:  
  服务器将 Channel 内消息保存到 `聊天历史记录` 数据库。
  Channel 里在线的 Users 可以实时收到通过 Channel Topic 发来的消息。
- 当某 User 上线时:  
  1. User 获取自己所有的 Channel。
  2. User 从 `聊天历史记录` 获取每个 Channel 内，自己未曾收到过的消息。
  3. 服务器为其重新订阅所有 Channel Topic。
- 删除 Channel 时，一并为 Channel 里的在线 Users 取消订阅 Channel Topic。

##### Channel 的定义：
- Channel 由多个 `MQTT User` 组成。
- Channel 由 `Channel ID (CHID)` 标识。

##### Channel 的表示法：
- 一个包含了 user1, user2 (注意是 UserID 不是 ClientID), 并且 Channel ID 为 $ch1 的 Channel 可以表示为： $ch1 = [user1, user2]。

##### Channel User Properties：
Channel 里的 User 可以有如下属性：
基本属性:
- UserID: MQTT CONNECT 消息中的 username 字段.
- Mode: 可以是 read、write, 或者 read-write. 分别表示允许向 Channel 发消息、允许从 Channel 接收消息，或者两者都允许。
mode 可以通过 MQTT Broker 的 ACL 功能来实现。

还可以自定义属性，比如 UserName 等。

##### Channel Properties:
一个 Channel 可以有如下属性：

基本属性:
- ChID: Channel ID。
- Users: Channel 里的 Users。
- Persistent: 是否持久化保存聊天消息。

还可以自定义其他属性，比如 Channel Name 等。

### Channel 消息类型

##### Channel Msg
即时通信消息。

Direction: Both `Broker -> Client` and `Client -> Broker`.
Method: PUBLISH
Topic: ch/channel-id
Payload: {"id": msg-id, "body": msg-body, "from": from-client-id}

##### Channel Cmd
- Create Channel Request   
  Direction: `Client -> Broker`. Client 发送创建/删除 Channel 请求给服务器。  
  Method: PUBLISH  
  Topic: ass/s/ch-create  
  Payload Exmaple:  
  ```JSON
  {"seq_num": 641,
   "users": [
     {"user_id": "user1", "mode": "rw"},
     {"user_id": "user2", "mode": "w"}
   ],
   "with_notif": true,
   "reply_topic": "ass/c/aN8XTQb34t"}
  ```
  `reply_topic` 是登录时随机生成，的格式为: "ass/c/random-generated-id".
  Client 在每次登录时要主动订阅这个 `reply_topic。`

- Create Channel Response  
  Direction: `Broker -> Client`. 服务器 发送创建/删除 结果给 Client。  
  Method: PUBLISH  
  Topic: `reply_topic` of the `Create Channel Request`  
  Payload:   
  ```JSON
  {"seq_num": 642, "rcode": 201, "msg":"channel created", "ch_id": id }
  ```

- Delete Channel Request
- Delete Channel Response

- Add Channel Users  
- Remove Channel Users  

- Channel Notification  
  Direction: `Broker -> Client`. Broker 发送 Channel 改动通知给 Client，通知类型可以有如下几种:  
  - Channel Created: Channel 被创建。CHID 属性描述了创建的 UserID，OP 属性描述了创建动作的发起人。
  - Channel Deleted: Channel 被删除。CHID 属性描述了删除的 UserID，OP 属性描述了创建动作的发起人。
  - Channel Users In: 新的 User 被加入, CHID 属性描述了 UserID，Users 属性描述了新加入的 Users，OP 属性描述了添加 User 动作的发起人。Broker 可以连续发多个 Channel Users In 消息。
  - Channel Users Out: User 被删除, CHID 属性描述了 UserID，Users 属性描述了被移除的 Users，OP 属性描述了移除 User 动作的发起人。Broker 可以连续发多个 Channel Users Out 消息。
  - User Defined Notification：用户自定义的 Channel 通知，Type 属性描述了自定义通知的类型，CHID，User，OP 等属性可选。消息体描述了通知的内容。

  Client 收到 Channel Notification 后要回复一个 Channel Cmd ACK。

#### Update Online Status Periodically


### 约定
- ChannelID 命名规范：  
  CHID 使用 `$` 开头，包含字母、数字、下划线、短线，最大长度 32： `^\$[0-9a-zA-Z_\-]{1,31}$`.  
  举例：$ch1, $ch-1, $342

- UserID 命名规范：  
  包含字母、数字、下划线、短线、`@`、点。最大长度 32：`^[0-9a-zA-Z_\-@\.]{1,32}$`.  
  举例：Scarlett, 98712, Shawn_123

- ClientID 命名规范：
  ClientID 是以斜线 `/` 分割的字符串, 并且分割的第一段被认为是 UserID。  
  ClientID 包含字母、数字、下划线、短线、斜线、`@`、点，且斜线分隔符不能作为开始和结尾字符。最大长度 32：`^(?!/)(?!.*?/$)[0-9a-zA-Z_\-@\./]{1,32}$`.  
  举例：9761/mobile, Shawn@app1/pc, app2-7843/pc/1  
  上面的例子里，UserID 分别为 9761, Shawn@app1 和 app2-7843。

UserID 相同的两个 Client 应当订阅相同的 Channel Topic。
