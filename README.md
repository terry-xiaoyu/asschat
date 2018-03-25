# ASSChat
**ASSChat** - A Super Simple Chat protocol for Instant Messaging. It is based on MQTT v5.0.

### Why another chat protocol?
在 IM 领域，经常用的两个协议是 `XMPP` 和 `MQTT`。

我很喜欢 MQTT，网络消耗很小，并且它设置了三个 QoS 级别，这一点比起 XMPP 是一个很大的进步。但是 MQTT 协议里考虑 IoT 的应用场景居多，因为它本身就是为 IoT 设计的。

目前多数开源 MQTT Broker 都实现了 `MQTT v3.1.1`, 并且都附加了协议里没有定义的常用功能，比如 `ACL`, `共享订阅` 等。`MQTT v5` 对此做出了很多有用的改进，比如错误返回值更加丰富：现在如果 PUBLISH 消息的 ACL (Access Control List) 规则匹配失败，Broker 可以返回一个 `0x87 Not authorized` 的 PUBACK。但其为 IoT 设计的目的没有改变，所以在使用这些开源 MQTT Broker 做 IM 应用的时候，有些地方就需要自己来实现。IM 应用与 IoT 应用给人感觉上非常类似，都是消息传递，但其实两者非常不同。具体来说，主要有几个方面：

- **在线优先 vs. 离线优先：**

  在 `IoT` 应用里，设计的时候其实是 **"online first"** 的。我们尽量保持设备始终在线，当设备由于某些原因掉线时，持续尝试重连。当设备重新连上服务器后，服务器(视情况)可以把这个设备处于离线状态的时间段里，发给它的消息再次推送给它。另外，设备可以利用 `retained` 消息来间歇性地更新自己的状态，当另一个设备订阅那个表示 "设备的状态" 的主题时，最后一条(最新的)状态信息会立刻推送给它。这个方案里，我们潜在地假设，IoT 设备处于离线状态时，没能收到它订阅的消息问题不大；或者假设在设备处于离线状态时，发给这个设备的消息量不会很多。这符合 IoT 的应用场景。

  而在 `IM` 应用里，设计时应当 **"offline first"**。IM 应用里，用户的终端不会每时每刻都保持在线，特别是那些移动终端。我们需要首先考虑不在线的场景，因为一个 IM 应用可能在大部分时间里都保持离线状态。当终端离线时，我们需要过推送通知的方式，唤起应用。当应用重新上线之后，应用需要一定的机制，从服务器获取那些它处于离线状态的时间段里，发给他的消息。**我们默认每一条 IM 消息都应该顺利送达，哪怕延时一段时间。**

  MQTT 里一个设备连接到 broker 之后，broker 会用一个 `Session` 维护其在服务端的状态。这些状态包括：设备订阅了哪些主题及其 QoS、由于设备离线未发给设备的消息队列、由于未收到设备的 ACK 暂存在 broker 的消息队列。多数情况下，IoT 设备是设置 `Clean Session` 连接 broker 的，这样在连接断开时，broker 不会为其维护那些繁重的数据：订阅、离线消息。

  但在 IM 应用里，我们不能使用 `Clean Session`。broker 需要知道应用的某个账号订阅了哪些主题，以便知道一个消息是否应该发送给那个账号，从而决定是否为其缓存这条消息。设置 `Clean Session` 在 IM 应用里没有意义。


- **发布/订阅模式：**

  在 MQTT 协议里，所有的消息传递都是通过 `发布/订阅` 来完成的。QoS 的设计也跟这个模式密切相关。如果许多 IoT 设备想随时了解 client1 的位置变更信息，他们只需要订阅类似这样的主题: "client1/location/#"。这是一个很灵活的模式，但在 IM 应用里，我们仅靠 `发布/订阅` 的话，其实并不方便：

  IM 应用里常见的聊天场景有两种：`一对一聊天`，`群组聊天`。

  发布/订阅模式的长处在于灵活，发布者不需要了解有哪些设备关心他发布的消息，他只需要简单的发布。比如一个温度传感器只需要发布消息到 "client1/temperature"，感兴趣的其他设备自然就能收到温度更新。

  但 `一对一聊天` 的场景与这个模式完全相反，发布者指明了要将消息发往什么地方。比如：要发给 user1，我们需要发布消息到 "user/user1"；要发给 user2 则要发布到 "user/user2"。这时候发布订阅失去了它的优势，因为订阅 "user/user1" 一定就是 user1，订阅 "user/user2" 一定就是 user2，直接将消息发给他就好了，服务器那里没必要走订阅匹配流程。

  `群组聊天` 也不能完全发挥 `发布/订阅` 模式的优势。假设一个名为 asschat 的群组里，所有人都订阅了 "group/asschat/#" 这个主题，那么当一个人向 "group/asschat" 发布一条消息的时候，所有人包括他自己也会收到这条消息，这样他就需要辨别出自己发出的这条消息，然后忽略掉。问题的根本在于，"group/asschat/#" 这个主题不属于发消息的那个人，他自己也订阅了这个主题，而在 `发布/订阅` 模式里，订阅一个自己可能会向其发布消息的主题是很奇怪的，就比如前面的例子里，client1 不会订阅 "client1/temperature"，因为自己温度自己还是知道的，没必要去订阅。

  考虑 `监听` 的场景：如果 IM 应用里，存在监听设备，用以监听某用户的聊天。这种情况下需要特殊设计一下发消息用的 MQTT 主题。比如，我们可以规定用户 user1 为接收消息订阅的主题是 "user/user1/#", 而 user2 给 user1 发消息时，需要发布到 "user/user1/user2"，注意我们将 user2 的用户名也加入到了发布主题里。这样监听设备就可以通过订阅 "user/+/user2" 监听 user2 发出来的消息，通过订阅 "user/user2" 来监听其他人发给 user2 的消息。

  再来考虑 `截获` 的场景：假设一个 IM 应用里的所有消息都是严格收到管制的，包含某些敏感信息如 `公司机密` 需要限制用户发消息必须先发布到监听设备 "user/intercepter", 然后再由监听设备转发到 "user/user-id"。

  可以看到使用 `发布/订阅` 模式做监听与截获并不直观。

  上述场景里，`发布/订阅` 不太合适，我们需要另外一套机制来处理这些 IM 应用中常见的需求。

- **ClientID vs. UserID：**

  MQTT 里 ClientID 是终端的唯一标识，订阅也是基于 ClientID 的。但 MQTT 对于 ClientID 的命名规则没有定义。在 IM 应用里，我们关心的是 UserID。如果支持多终端登陆的话，一个 UserID 需要对应多个 ClientID。举例来说，用户 user1 有两个终端需要同时登陆， 我们需要分别为这两个终端分配不同的 ClientID："mobile:user1" 和 "pc:user1"，这几个终端不仅仅需要接收同样的消息，在其中一个发消息的时候，还需要同步到其他终端上去。这些过程应该由协议来规范。

- **添加好友**
  IM 应用里，添加好友是非常常见的功能，非好友关系不能聊天。为了实现添加好友的功能，MQTT 里常见的做法有两个：
  - 用户 user1 接收消息需要订阅 "user/user1", user2 向 user1 发消息需要发布到 "user/user1"。默认的 ACL 规则禁止发布，当 user1 与 user2 添加好友后，开放 user2 发布到 "user/user1" 的 ACL 权限。
  - 用户默认不订阅任何主题。当 user1 与 user2 添加好友后，为 user1 和 user2 同时订阅 "conversation/<convs-id>"。双方通过 "conversation/<convs-id>" 来聊天。
  
  第二中方式相对优雅，好处是不必维护繁杂的 ACL 规则列表，并且将添加好友和创建群组的语义统一了起来，添加好友就是创建一个仅有两人的群组。但这种方式面临着跟群聊一样的问题：发消息的人会收到一条自己发来的消息。

### ASSChat Protocol 设计原则
- 完全兼容 MQTT v5 基础协议，使用 MQTT 协议里定义的各种 `Control Packets`。
- 在 MQTT 基础协议之上，增加一套新的 Control Packet Type: `Channel`。
- 在 ClientID 的语义基础之上，增加一套约定。

ASSChat 基于 MQTT 协议 v5，意味着可以完全复用 MQTT 协议里的 连接建立过程和断开、以及连接保活过程。ASSChat 仍然可以使用 PUB/SUB 过程，但不作为聊天通道，仅做为实现 IM 应用里的辅助功能，比如检查用户是否在线等。

### Channel
`Channel` 是 IM 用户聊天的通道，需要有建立、删除 Channel，发送消息到 Channel 等过程。

##### Channel 的定义：
- Channel 由 `Channel ID (CHID)` 标识。
- Channel 由多个 `Channel Endpoint (CHEP)` 组成，`Channel Endpoint` 可以是 `user` 或 其他 `Channel`。
- 一个 Channel 里至少包含两个 `Channel Endpoint`。
- 当一个 `Channel Endpoint` 发来消息时，Channel 向所有除了它之外的 `Channel Endpoints` 推送这条消息。

##### Channel 的表示法：
一个包含了 user1, user2 (注意是 UserID 不是 ClientID), 并且 Channel ID 为 ChID1 的 Channel 可以表示为： ChID1 = [user1, user2]。

一个包含了 user5, ChID1, ChID2 的 Channel 可以表示为：ChID3 = [user5, ChID1, ChID2]。

##### Channel 的推送方式：

假设有三个 Channel：ChID1 = [user1, user2], ChID2 = [user3, user4], ChID3 = [user5, ChID1, ChID2].

那么：
- 如果 user1 向 ChID1 发了一条消息，消息会被推送给 user2。同样如果 user3 向 ChID2 发消息会被推送到 user4。但如果 user3 发消息给 ChID1 的话，会被 ChID1 拒绝。
- 如果 user5 向 ChID3 发送消息会被推送到 ChID1 和 ChID2, 但 user1 发消息到 ChID1, 或者 user3 发消息到 ChID2 都无法把消息推送到 ChID3, 也就无法推送到 user5。
- 若更改 ChID1 = [user1, user2, ChID3], ChID2 = [user3, user4, ChID3], 则三个 Channel **互通**。意思是 user5 发送的消息会经由 ChID1 推送给 user1、user2，经由 ChID2 推送给 user3、user4，反之亦然。


##### Channel Notifications

##### ACL in Channel

### 约定

