
# ![Status: WIP](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square) Bitswap 中文版

**Author(s)**:
- Adin Schmahmann
- David Dias
- Jeromy Johnson
- Juan Benet

**Maintainer(s)**:

* * *

**摘要**

Bitswap 是用于发送和接收数据块的交换协议。它负责两个基本功能:
1. 尝试从网络中获取客户端已请求的数据块。
2. 发送自己拥有的数据块给需要它的节点。

## 文档结构

- [Introduction](#introduction)
- [Bitswap Protocol Versions](#bitswap-protocol-versions)
  - [Bitswap 1.0.0](#bitswap-100)
  - [Bitswap 1.1.0](#bitswap-110)
  - [Bitswap 1.2.0](#bitswap-120)
- [Implementations](#implementations)

## 介绍

Bitswap 是消息型协议，与request-response不同。 所有的消息包含 wantlists, 和/或 块blocks。
收到wantlist时，Bitswap 对端应该完全处理和响应请求方，既包括块信息，也包括块数据本身。 
一旦收到块数据，请求方应该发出 `Cancel` 通知给其他请求过的对端，标志自己不再需要其他对端发送给块数据。

Bitswap 的目标是成为一个简洁协议。所以实现它时可以从各方面平衡它，包括吞吐、延迟、公平性、内存用量等等，以满足特定需求。

## Bitswap 协议版本

已有多个Bitswap 协议版本，随着时间也会有更多版本演进。我们简单回顾各版本变化。

- `/ipfs/bitswap/1.0.0` - 初始版本
- `/ipfs/bitswap/1.1.0` - 支持 CIDv1
- `/ipfs/bitswap/1.2.0` - 支持 Wantlist Have's 和 Have/DontHave 响应

## 块大小

实现 Bitswap 时必须支持发送接收单个小于等于2MiB的块文件。并不推荐处理大于2MiB的块，以保持对只支持2MiB的已有实现的兼容。

## Bitswap 1.0.0 版

### Bitswap 1.0.0: 交互模式

假设客户端 C 向从服务端 S 获取一些数据块：

1. C 通过流`s_want`发送它需要的块的消息到 S
    1. C 可以发送完整的清单wantlist，或更新一个已有的wantlist
    2. C 可以复用这个流来发送新需求 
2. S 通过 `s_receive` 响应块请求。 S 可以复用这个流继续响应后续的请求。
    1. S 应尊重 C 的wantlist请求中描述的块优先级，更高的 `priority` 值对应的块应更早被响应。
3. 当 C 不再需求之前请求的块，它应该发送一个 `Cancel` 消息给所有还没有响应对应块的对端节点。

### Bitswap 1.0.0: 消息

一条 Bitswap 消息应包含以下任意内容:

1. 发送方的wantlist。可以是发送方的完整wantlist，也可以只是需要接收方知道的变化部分。
2. 块数据。这些是具体要请求的文件数据 (例如，在接收方wantlist中的块但也被发送方所知道）。

#### Bitswap 1.0.0: 网络报文格式

报文格式简化为消息流。以下 protobuf 对象描述了消息格式。注意: 所有protobufs对象使用proto3语法定义。

```protobuf
message Message {
  message Wantlist {
    message Entry {
      bytes block = 1; // 块key, 如 CIDv0
      int32 priority = 2; // 归一化的优先级。默认为1
      bool cancel = 3;  // 是否撤销该块
    }

    repeated Entry entries = 1; // 块列表
    bool full = 2; // 是否是完整的清单，默认为false
  }

  Wantlist wantlist = 1;
  repeated bytes blocks = 2;
```

### Bitswap 1.0.0: 协议格式

所有在连接流上发送的协议消息需要在前面带上消息字节长度，采用无符号的变长整数定义
[multiformats unsigned-varint spec](https://github.com/multiformats/unsigned-varint)。

所有的消息大小需要在4MiB字节以内。

## Bitswap 1.1.0 版

Bitswap 1.1.0 在protobuf消息对象中引入了 'payload' 字段， 并废弃了已有的'blocks' 字段。
'payload' 字段是一个cid和块数据对的数组。包含cid前缀的目的是确保在接收端处理块数据时用了正确的编码和哈希函数。

其他部分与1.0.0版相同。

### Bitswap 1.1.0: 网络报文格式

```protobuf
message Message {
    message Entry {
      bytes block = 1; // 块CID
      int32 priority = 2; // 归一化的优先级，默认为1
      bool cancel = 3; // 是否取消该块
    }

    repeated Entry entries = 1; // 块清单
    bool full = 2; // 是否是完整的wantlist。默认为 false
  }

  message Block {
    bytes prefix = 1; // CID 前缀 (CID数据结构中除了multihash摘要之外的部分)
    bytes data = 2;
  }

  Wantlist wantlist = 1;
  repeated Block payload = 3; 
}
```

## Bitswap 1.2.0 版

Bitswap 1.2.0 extends the Bitswap 1.1.0 protocol with the three changes:
1. Being able to ask if a peer has the data, not just to send the data
2. A peer can respond that it does not have some data rather than just not responding
3. Nodes can indicate on messages how much data they have queued to send to the peer they are sending the message to

### Bitswap 1.2.0: Interaction Pattern

Given that a client C wants to fetch data from some server S:

1. C opens a stream `s_want` to S and sends a message for the blocks it wants
    1. C may either send a complete wantlist, or an update to an outstanding wantlist
    2. C may reuse this stream to send new wants
    3. For each of the items in the wantlist C may ask if S has the block (i.e. a Have request) or for S to send the block (i.e. a block request). C may also ask S to send back a DontHave message in the event it doesn't have the block
2. S responds back on a stream `s_receive`. S may reuse this stream to send back subsequent responses
    1. If C sends S a Have request for data S has (and is willing to give to C) it should respond with a Have, although it may instead respond with the block itself (e.g. if the block is very small)
    2. If C sends S a Have request for data S does not have (or has but is not willing to give to C) and C has requested for DontHave responses then S should respond with DontHave
    3. S may choose to include the number of bytes that are pending to be sent to C in the response message
    4. S should respect the relative priority of wantlist requests from C, with wants that have higher `priority` values being responded to first.
3. When C no longer needs a block it previously asked for it should send a Cancel message for that request to any peers that have not already responded about that particular block. It should particularly send Cancel messages for Block requests (as opposed to Have requests) that have not yet been answered.

### Bitswap 1.2.0: Wire Format

```protobuf
message Message {
  message Wantlist {
    enum WantType {
      Block = 0;
      Have = 1;
    }

    message Entry {
      bytes block = 1; // CID of the block
      int32 priority = 2; // the priority (normalized). default to 1
      bool cancel = 3; // whether this revokes an entry
      WantType wantType = 4; // Note: defaults to enum 0, ie Block
      bool sendDontHave = 5; // Note: defaults to false
    }

    repeated Entry entries = 1; // a list of wantlist entries
    bool full = 2; // whether this is the full wantlist. default to false
  }
  message Block {
    bytes prefix = 1; // CID prefix (all of the CID components except for the digest of the multihash)
    bytes data = 2;
  }

  enum BlockPresenceType {
    Have = 0;
    DontHave = 1;
  }
  message BlockPresence {
    bytes cid = 1;
    BlockPresenceType type = 2;
  }

  Wantlist wantlist = 1;
  repeated Block payload = 3;
  repeated BlockPresence blockPresences = 4;
  int32 pendingBytes = 5;
}
```

## Implementations

- <https://github.com/ipfs/go-bitswap>
- <https://github.com/ipfs/js-ipfs-bitswap>
