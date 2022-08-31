
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

Bitswap 1.2.0 版通过3个改变扩展了 1.1.0 协议:
1. 可以询问对端是否有某个数据块，而不只是请求获取它。
2. 对端可以明确回复自己没有那些块，而不只是不响应。
3. 节点可以在回复的消息中标示自己缓存了多少数据待发送。

### Bitswap 1.2.0: 交互模式

假设客户端 C 想要从服务端 S 请求数据:

1. C 打开连接到 S 的流 `s_want`，发送包括自己需要的块的一个消息
    1. C 可以发送完整的 wantlist，也可以针对已有的 wantlist 变化部分
    2. C 可以复用流来发送新的需求
    3. 对 wantlist 中每一个块，C 可以询问 S 是否拥有这个块 block (如用 Have 请求) ，或请 S 发送这个块 (如用块请求)。C 也可以请 S 回应 一个 DontHave 消息表示对端没有这个块
2. S 用 `s_receive` 流来响应。 S 可以复用这个流以响应后续的请求
    1. 如果 C 向 S 发送一个 Have 请求，S 有这个数据 (它也愿意发给 C) 就响应一个 Have 回复，当然它也可以直接回应块数据本身 (如果块并不大的话)
    2. 如果 C 向 S 发送一个 Have 请求，S 不拥有这个数据 (或它有但不想发给 C) 且 C 要求进行 DontHave 回复，那么 S 应该回复 DontHave
    3. S 可以选择将等待发送的字节数量也放在响应消息中
    4. S 应该尊重 C 的请求清单中描述的相对优先级，更高 `priority` 值的块应该被优先响应
3. 当 C 不再需要一个之前请求的块时，它应该发送一个块的 Cancel 消息到所有请求过但还没响应的对端。它特别应该为没响应的块发送这个 Cancel 请求 (与 Have 请求相反) 

### Bitswap 1.2.0: 网络报文格式

```protobuf
message Message {
  message Wantlist {
    enum WantType {
      Block = 0;
      Have = 1;
    }

    message Entry {
      bytes block = 1; // 块CID
      int32 priority = 2; // 归一化的优先级，默认为1
      bool cancel = 3; // 是否取消这个块
      WantType wantType = 4; // Note: 默认为 0, 即 Block
      bool sendDontHave = 5; // Note: 默认为 false
    }

    repeated Entry entries = 1; // 块清单 wantlist
    bool full = 2; // 是否为完整清单 wantlist. 默认为 false
  }
  message Block {
    bytes prefix = 1; // CID 前缀 (CID 数据结构除multihash摘要以为部分)
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

## Implementations 协议实现

- <https://github.com/ipfs/go-bitswap>
- <https://github.com/ipfs/js-ipfs-bitswap>
