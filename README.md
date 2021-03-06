# ASSChat
**ASSChat** - A Super Simple Chat specification for Instant Messaging. It is fully based on MQTT v5.0.

## Why ASSChat?
在 IM 领域，经常用的两个协议是 `XMPP` 和 `MQTT`。

我很喜欢 MQTT，网络消耗很小，并且它设置了三个 QoS 级别，这一点比起 XMPP 是一个很大的进步。

在使用各类开源的 MQTT Broker 开发 IM 应用时，我们需要基于 MQTT 基础协议，设计一套上层聊天协议，并根据这套协议对 MQTT broker 做一点定制工作(通过开发插件)。这套上层协议就是 **ASSChat**.

我们的 ASSChat 基于 MQTT v5.0，因为 MQTT v3.1 不够完善。很多现实中的应用场景并没有包含在协议里。 v5.0 里做了许多有用的改进，比如：
- 返回值更加丰富，比如现在 PUBLISH 消息的 ACL (Access Control List) 规则如果匹配失败，Broker 可以返回一个 `0x87 Not authorized` 的 PUBACK。之前 PUBACK 里没有返回值。类似的 DISCONNECT 消息也可以设置错误码。
- 订阅主题时，可以设置 SUBSCRIBE 消息里的 `No Local` Option，阻止消息被转发到当前连接。这个让我们的订阅设计更加灵活：我们可以让两个用户订阅同一个主题以实现聊天，客户端向这个共同的主题发消息时，不用担心收到自己发出的消息。


## IoT 应用与 IM 应用的不同：
MQTT 协议里考虑 IoT 的应用场景居多，因为它本身就是为 IoT 设计的。

IM 应用与 IoT 应用给人感觉上非常类似，都是消息传递，但其实两者略有不同。具体来说，主要体现在两者对待 `离线` 的态度上：

在 `IoT` 应用里，设计的时候其实是 **"Online First"** 的。我们尽量保持设备始终在线，当设备由于某些原因掉线时，持续尝试重连。当设备重新连上服务器后，服务器(视情况)可以把这个设备处于离线状态的时间段里，发给它的消息再次推送给它。这个方案里，**IoT 场景里，我们潜在地假设，IoT 设备处于离线状态时，没能收到它订阅的消息问题不大；或者假设在设备处于离线状态时，发给这个设备的消息量不会很多.** 这符合 IoT 的应用场景。

而在 `IM` 应用里，设计时应当 **"Offline First"**。IM 应用里，我们需要首先考虑不在线的场景，因为一个 IM 应用可能在大部分时间里都保持离线状态。当终端离线时，我们需要过推送通知的方式(比如 [APN](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html))，唤起应用。当应用重新上线之后，应用需要一定的机制，从服务器下载那些它处于离线状态的时间段里，发给他的消息。**IM 应用里，我们默认每一条 IM 消息都应该顺利送达，哪怕延时一段时间。**

MQTT 里一个设备连接到 broker 之后，broker 会用一个 `Session` 维护其在服务端的状态。这些状态包括：设备订阅了哪些主题及其 QoS、由于设备离线未发给设备的离线消息队列、由于未收到设备的 ACK 暂存在 broker 的消息队列等。多数情况下，IoT 设备是设置 `Clean Session = True` 连接 broker 的，这样在连接断开时，broker 不会为其维护那些繁重的数据：订阅、离线消息等。

MQTT 协议里的 `Clean Session` 是在 Client Connect 时设置的，也就是说 Broker 要么保存这个 Client 的所有订阅，要么一个也不保存。这不符合我们 IM 应用里的需求: 有些订阅信息的确需要持久化保存，比如那些作为聊天通道的订阅。服务器需要知道聊天消息应该投递到哪些终端去，如果终端不在线就帮他们缓存下来，以便终端在未来的某时刻来获取。但其他的订阅可能并不需要持久化保存，比如一个用户 A 订阅了另一个用户 B 的在线状态，A 离线的时候，他其实不再需要继续接收 B 上线、下线的通知。所以这个关于用户在线状态的订阅信息是不需要持久化保存的。

## ASSChat Spec 设计原则
- 完全基于 MQTT v5 基础协议。
- 不持久化 Session 和 Subscriptions，以保持 MQTT Broker 的轻量。
- 基于 MQTT PUBLISH 消息定制 `Channel` 协议。`Channel` 是用来聊天的通道。
- 用户之间使用 `Channel` 聊天，使用 `发布/订阅` 作其他辅助功能。
- 在 ClientID 的语义基础之上，增加一套命名约定。ClientID 由 UserID 以及扩展字段组成，以实现用户的多终端设备同时登陆。

## Channel 设计概要
`Channel` 是 IM 用户聊天的通道，通过为 Channel 内的所有 Users 订阅相同的 MQTT Topic 来实现聊天，我们称这个 Topic 为 `Channel Topic`。

如果 User 有多个终端的话，我们为他的每个终端都订阅 `Channel Topic`，这样我们就实现了多终端之间的消息同步。

订阅 `Channel Topic` 时使用 `No Local` option。

`Channel Topic` 的订阅和发布都使用 `QoS1`。

Channel 内的消息需要持久化，而 Channel 之外的 PUBLISH 消息不持久化。

- 创建 Channel 时，服务器为 Channel 里的所有在线的 Users 订阅 Channel Topic。
- 当 Channel Topic 收到消息时:
  服务器将 Channel 内消息当做 `聊天历史记录` 持久化保存下来。
  Channel 里在线的 Users 可以实时收到通过 Channel Topic 发来的消息。
- 当某 User 上线时:
  1. User 获取自己所属的所有 Channel 的 ChID。
  2. 服务器为其重新订阅所有 Channel Topic。
  3. 对于每个 ChID，User 从服务器下载自己未曾收到过的 Channel 消息。

- 删除 Channel 时，一并为 Channel 里的在线 Users 取消订阅 Channel Topic。

### Channel 的定义：
- Channel 由多个 `MQTT UserID` 组成。
- Channel 由 `Channel ID (CHID)` 标识。

### Channel 的表示法：
- 一个包含了 user1, user2 (注意是 UserID 不是 ClientID), 并且 Channel ID 为 ch1 的 Channel 可以表示为： ch1 = [user1, user2]。

### Channel User Properties：
Channel 里的 User 可以有如下属性：
基本属性:
- UserID: UserID, MQTT CONNECT 消息中的 username 字段.
- Mode: 3 bit 的数字，各 bit 位又高到低分别表示: read, write, push. 分别表示是否允许从 Channel 接收消息，是否允许向 Channel 发消息，是否允许接收来自本 Channel 的推送通知。举例俩说，数字 7 (111) 表示所有三个都允许; 而数字 3 (010) 表示不允许用户接收来自 Channel 的实时消息和推送通知，但允许他发送消息到 Channel.

`Mode` 可以通过 MQTT Broker 的 ACL 功能来实现。

还可以自定义属性，比如 User Name 等。

### Channel Properties:
一个 Channel 可以有如下属性：

基本属性:
- ChID: Channel ID。
- Users: Channel 里的 Users。
- Persistent: 是否持久化保存聊天消息。

还可以自定义其他属性，比如 Channel Name 等。

## Channel 消息类型

### Channel Msg
即时通信消息。Client 发给 Broker 的消息格式可以自定义，服务器需要在消息体内附加 `msg_id` 等额外字段之后再进行分发。

| Direction                                      | Method  | Topic       |
| ---------------------------------------------- | ------- | ----------- |
| Both `Broker -> Client` and `Client -> Broker` | PUBLISH | ch/{{ChID}} |

##### Payload Example:

`Client -> Broker`:
```JSON
// The format of payload is application specific
//   not specified in ASSChat.
// Following is an exmaple which sends a text message with an
// image.
{
  "msg": "hello?",
  "type": "image",
  "file": "https://example/image.jpg"
}
```

`Broker -> Client`:
```JSON
// Broker will wrap the payload in `body` field and then send it to target:
{"msg_id": 1231, // monotonically increasing msg-id in current channel
 "type": "ch_msg", // msg type
 "body": {
   "msg": "hello?",
   "type": "image",
   "file": "https://example/image.jpg"
 },
 "push": { // optional, for push notifications
   "sound": "bingbong.aiff",
   "@": "except(all(), 'user1')"
 },
 "from": "user1/pc", // sender's client-id
 "timestamp": 1522434238021  // server's timestamp in ms
}
```

##### '@' 字段支持的表达式:
| operator | Meaning                    | Arguments      |
| -------- | -------------------------- | -------------- |
| all      | all users in channel       |                |
| except   | exclude a user from a list | UserList, User |

Examples:
Let's say there're 3 users in a channel, user1, user2, user3.
- "@": "'user1', 'user1'" // @user1 @user2
- "@": "all()" // @user1 @user2 @user3
- "@": "except(all(), 'user1')" // @user2 @user3

### Channel Cmd
如果 Request 里设置了 `"want_reply": true`，服务器需要将回复发布到 `ass/c/{{client-id}}` 主题。Client 上线时，Broker 自动会为其订阅 `ass/c/{{client-id}}`，用以接收 Channel Cmd 消息回复和 Channel Notification.

Request 里可以设置 `"notify_change": true`，来广播 Channel 改动通知给所有 Users。

##### Create/Update Channel

  **Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_create |

  Payload Example:
  ```JSON
  {"req_id": "5b18b249bc09", // request id
   "users": [
     {"user_id": "user1", "mode": 7 }, // userid, mode
     {"user_id": "user2", "mode": 7 }
   ],
   "persis": true, // persistent msgs in channel
   "want_reply": true, // want a response being publish to "ass/c/{{client-id}}"
   "notify_change": true // sent channel change notifications to all users.
 }
  ```

  **Response:**

  | Direction          | Method  | Topic           |
  | ------------------ | ------- | --------------- |
  | `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  { "req_id": "5b18b249bc09", // which request we're replying to
    "type": "ch_create", // cmd type
    "rcode": 201, // create
    "rmsg":"channel created", // response detail
    "ch_id": "ch1" // the channel-id created
  }
  ```

##### Search Channels

  **Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_search |

  Payload Example:

  ```JSON
  {"req_id": "3264e19627e8",
   "query": "or(eq('ch_id', 'ch1'), expr('ch_name', 'Awesome.*Group') )"
  }
  ```
  **Supported query operators:**

| Operators | Meaning                            | Arguments                    |
| --------- | ---------------------------------- | ---------------------------- |
| eq        | equals                             | channel_property, value      |
| gt        | greater than                       | channel_property, value      |
| gte       | gte	greater than or equal          | channel_property, value      |
| lt        | less than                          | channel_property, value      |
| lte       | less than or equal                 | channel_property, value      |
| neq       | not equal to                       | channel_property, value      |
| expr      | regular expression                 | channel_property, pattern    |
| cs        | contains                           | channel_property, [value]    |
| or        | combine 2 query clauses with 'or'  | query_clause1, query_clause2 |
| and       | combine 2 query clauses with 'and' | query_clause1, query_clause2 |

  **Query examples:**
  ```
  1. channels with ch_id = "ch1":
  "eq('ch_id', 'ch1')"

  2. channels with ch_name pattern match "^A.*Team$":
  "expr('ch_name', '^A.*Team$')"

  3. channels contains users whoes user_id = 'user1' or 'user2':
  "cs('users', ['user1', 'user2'])"
  ```

  **Response:**

  | Direction          | Method  | Topic           |
  | ------------------ | ------- | --------------- |
  | `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  { "req_id": "3264e19627e8",
    "type": "ch_search",
    "rcode": 200,
    "rmsg": "ok",
    "query_result": [
      { "ch_id": "ch1",
        "ch_name": "Awesome Elixir Group!"
        "users": [
          {"user_id": "user1", "mode": 7}, // userid, mode
          {"user_id": "user2", "mode": 7}
        ],
        "persis": true
      },
      ...
    ]
  }
  ```

##### Delete Channels

  **Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_delete |

  Payload Example:
  ```JSON
  { "req_id": "1db236757400",
    "ch_id": "ch1",
    "want_reply": true,
    "notify_change": true
  }
  ```
  **Response:**

  | Direction          | Method  | Topic           |
  | ------------------ | ------- | --------------- |
  | `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  { "req_id": "1db236757400",
    "type": "ch_delete",
    "rcode": 200,
    "rmsg":"channel deleted",
    "ch_id": "ch1"
  }
  ```

##### Add/Update Channel Users

  **Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_users_update |

  Payload Example:
  ```JSON
  { "req_id": "e082938117d1",
    "ch_id": "ch1",
    "users": [
      {"user_id": "user3", "mode": 7}
    ],
    "want_reply": true,
    "notify_change": true
  }
  ```
  **Response:**

  | Direction          | Method  | Topic           |
  | ------------------ | ------- | --------------- |
  | `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  { "req_id": "e082938117d1",
    "type": "ch_users_update",
    "rcode": 201,
    "rmsg":"channel user added",
    "ch_id": "ch1"
  }
  ```

##### Search Users in Channel

**Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_users_search |

Payload Example:

```JSON
{"req_id": "b8720f15806f",
 "query": "eq('ch_id', 'ch1')"
}
```

See `ch_search` section for all supported query operators.

**Response:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

Payload:
```JSON
{ "req_id": "b8720f15806f",
  "type": "ch_users_search",
  "rcode": 200,
  "rmsg": "ok",
  "query_result": [
    { "user_id": "user1",
      "user_name": "Shawn",
      "mode": 7
    },
    { "user_id": "user2",
      "user_name": "Scarlett",
      "mode": 7
    },
    ...
  ]
}
```

##### Remove Channel Users

  **Request:**

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Client -> Broker` | PUBLISH | ass/s/ch_users_delete |

  Payload Exmaple:
  ```JSON
  { "req_id": "caa2e0e34d69",
    "ch_id": "ch1",
    "users": [
      "user3", "user1"
    ],
    "want_reply": true,
    "notify_change": true
  }
  ```
  **Response:**

  | Direction          | Method  | Topic           |
  | ------------------ | ------- | --------------- |
  | `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  { "req_id": "caa2e0e34d69",
    "type": "ch_users_delete",
    "rcode": 200,
    "rmsg":"channel user deleted",
    "ch_id": "ch1"
  }
  ```

##### Channel Notification

| Direction          | Method  | Topic               |
| ------------------ | ------- | ------------------- |
| `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

  Payload:
  ```JSON
  {"msg_id": 1297,
   "type": "ch_notif",
   "notif": {}, // notification object
   "from": "user1/pc", // the one who initiate the notification
   "timestamp": 1522434238021
  }
  ```

  上面的 `"notif"` 类型可以有如下几种:
  - Channel Created: Channel 被创建。
    ```JSON
    "notif": {
      "type": "CH_CREATED",
      "ch_id": "ch1"
    }
    ```
  - Channel Deleted: Channel 被删除。
    ```JSON
    "notif": {
      "type": "CH_DELETED",
      "ch_id": "ch1"
    }
    ```
  - Channel Users In: 新的 User 被加入。
    ```JSON
    "notif": {
      "type": "CH_USERS_IN",
      "ch_id": "ch1",
      "users": ["user1", "user2"]
    }
    ```
  - Channel Users Out: User 被删除。
    ```JSON
    "notif": {
      "type": "CH_USERS_IN",
      "ch_id": "ch1",
      "users": ["user3"]
    }
    ```
  - User Defined Notification：用户自定义的 Channel 通知。除了 type 和 ch_id 字段外，用户可以定义其他字段。
    ```JSON
    "notif": {
      "type": "{{User Defined Notification Type}}",
      "ch_id": "ch1",
      "{{User Defined Key}}": "{{User Defined Value}}"
    }
    ```

## Online Status
ASSChat 使用 `status/{{client-id}}` 作为记录终端在线状态的 Topic.

Client 上线时，服务器发布 `Retained` 消息到 `status/{{client-id}}`, 消息体为 `"online"`。

Client 下线时，服务器发布 `Retained` 消息到 `status/{{client-id}}`，消息体为 `"offline"` 。

当一个 Client 想要知道某个用户 "user1" 的在线状态时，需要主动订阅 `"status/user1/#"`.

## 推送通知

Client 上线时，需要向服务器上传其 device token；当 Client 不想再收到推送通知时 (比如退出手机 app，或者设置手机消息免打扰)，需要从服务器删除 device token 记录。

如果一个 Channel 内的某个 User 有离线消息，Channel 会向这个 User 的所有 注册了 Device Token 的 Clients 发送推送通知，除非这个 User 的 Mode 属性的 `push` bit 位为 `0` (可用来实现 Channel 内推送通知免打扰)。

但如果那个离线消息体内设置了 `push.@`, Channel 将向 `@` 列表里的所有用户的所有 Clients 强行发送推送通知，而不管这些 User 是否设置了 `push` Mode 为 0，也不管这些 Clients 下线与否。

### Device token 注册
**Request:**

| Direction          | Method  | Topic                   |
| ------------------ | ------- | ----------------------- |
| `Client -> Broker` | PUBLISH | ass/s/push_token_reg |

Payload Exmaple:
```JSON
{ "req_id": "aa05f0d6f476",
  "push_target": "APNS", // can be APNS, FCM or other supported push services.
  "device_token": "{{the device token string}}",
  "want_reply": false
}
```
**Response:** (only if `want_reply` set to true)

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

Payload:
```JSON
{ "req_id": "aa05f0d6f476",
  "type": "push_token_reg",
  "rcode": 201,
  "rmsg":"device token registered"
}
```

### Device token 取消注册
**Request:**

| Direction          | Method  | Topic                  |
| ------------------ | ------- | ---------------------- |
| `Client -> Broker` | PUBLISH | ass/s/push_token_unreg |

Payload Exmaple:
```JSON
{ "req_id": "46e657cf3ea3",
  "want_reply": false
}
```
**Response:** (only if `want_reply` set to true)

| Direction          | Method  | Topic           |
| ------------------ | ------- | --------------- |
| `Broker -> Client` | PUBLISH | ass/c/{{client-id}} |

Payload:
```JSON
{ "req_id": "46e657cf3ea3",
  "type": "push_token_unreg",
  "rcode": 200,
  "rmsg":"device token unregistered"
}
```

### 通知标题和内容
推送通知的 Title 以及 Body 与应用相关，ASSChat 不做规定。Sound 由消息体里 `push.sound` 设定，默认为系统 default 铃声。

## 约定
- UserID 命名规范：
  包含字母、数字、下划线、短线、`@`、点。最大长度 32：`^[0-9a-zA-Z_\-@\.]{1,32}$`.
  举例：Scarlett, 98712, Shawn_123

- ClientID 命名规范：
  ClientID 是以斜线 `/` 分割的字符串, 并且分割的第一段被认为是 UserID。
  ClientID 包含字母、数字、下划线、短线、斜线、`@`、点，且斜线分隔符不能作为开始和结尾字符。最大长度 32：`^(?!/)(?!.*?/$)[0-9a-zA-Z_\-@\./]{1,32}$`.
  举例：9761/mobile, Shawn@app1/pc, app2-7843/pc/1
  上面的例子里，UserID 分别为 9761, Shawn@app1 和 app2-7843。

UserID 相同的两个 Client 应当订阅相同的 Channel Topic。
