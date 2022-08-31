# ![](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square) IPNS - Inter-Planetary Naming System

**Authors(s)**:
- Vasco Santos ([@vasco-santos](https://github.com/vasco-santos))
- Steven Allen ([@Stebalien](https://github.com/Stebalien))

-----

**摘要**

IPFS 依托内容寻址数据，相应它是不变的: 修改一个对象也会导致哈希值变化，地址也相应的变化了，导致它完全是另一个数据。尽管如此，可变数据也有很多用户场景。IPNS 可提供等价功能。

通盘考虑，IPFS 命名层负责如下工作:
- 指向数据的可变指针
- 有可读性的命名

# 内容列表

- [介绍](#介绍)
- [IPNS记录](#ipns记录)
- [协议](#协议)
- [概况](#概况)
- [API规范](#API规范)
- [IPFS集成](#IPFS集成)

## 介绍

每次修改一个文件，它的内容寻址会变化。结果导致使用该文件的人需要更换地址。由此很不实用, IPNS被创建以解决该问题。

IPNS 基于了 [SFS](http://en.wikipedia.org/wiki/Self-certifying_File_System)。它包括 PKI 命名空间，其名称是public key的哈希。
由此，控制 private key 的人就可以完全控制文件地址。相应的，记录是用 private key 来签发，然后通过网络分发(在IPFS中通过routing系统)。 
这是一个来通过Internet但不依靠中心化和认证机构就能分配可变文件名的平等方式。

## IPNS记录

一个IPNS记录是包含如下字段的数据结构:

- 1. **Value** (bytes)
  - 它可以包含任意路径，包括指向其他IPNS记录的路径，`dnslink` 路径 (eg. `/ipns/example.com`) 或 IPFS 路径 (eg. `/ipfs/Qm...`)
- 2. **Validity** (bytes)
  - 记录过期方式采用有纳秒精度的[RFC3339](https://www.ietf.org/rfc/rfc3339.txt) 
  - 注意: 目前过期时间是唯一可用于证明有效性的字段。
- 3. **Validity Type** (uint64)
   - 允许我们定义记录的有效性条件。
   - 目前只支持用 `validityType = 0` 打开有效期认证。
- 4. **Signature** (bytes)
  - 连接值，包括有效性字段、类型。
  - 提过private key签名连接值。
  - 注意: 一旦我们增加有效性类型，需要修改签名。更多信息在 [ipfs/notes#249](https://github.com/ipfs/notes/issues/249)
- 5. **Sequence** (uint64)
  - 表达当前记录的版本 (从0开始)
- 6. **Public Key** (bytes)
  - 前面记录的公钥（Public key）
  - 注意: 如果不能从Peer ID中解析出的话就**必须**包含公钥 (参考 [libp2p/specs#100](https://github.com/libp2p/specs/pull/100/files)).
- 7. **ttl** (uint64)
  - 一个提示记录可以缓存多久的线索，例如在 DHT 中，用来检查记录是否需要更新。

这些记录在本地缓存，也分布在网络中以让每个人都访问到。为保持这样的数据结构，我们用 [Protocol Buffers](https://github.com/google/protobuf), 它是语言无关、平台无关、有扩展机制的序列化数据结构。 
```
message IpnsEntry {
	enum ValidityType {
		// 设置该记录生命周期结束的时间
		EOL = 0;
	}
	required bytes value = 1;
	required bytes signature = 2;

	optional ValidityType validityType = 3;
	optional bytes validity = 4;

	optional uint64 sequence = 5;

	optional uint64 ttl = 6;

	optional bytes pubKey = 7;
}
```

## 协议

Taking into consideration a p2p network, each peer should be able to publish IPNS records to the network, as well as to resolve the IPNS records published by other peers.

When a node intends to publish a record to the network, an IPNS record needs to be created first. The node needs to have a previously generated asymmetric key pair to create the record according to the datastructure previously specified. It is important pointing out that the record needs to be uniquely identified in the network. As a result, the record identifier should be a hash of the public key used to sign the record.

As an IPNS record may be updated during its lifetime, a versioning related logic is needed during the publish process. As a consequence, the record must be stored locally, in order to enable the publisher to understand which is the most recent record published. Accordingly, before creating the record, the node must verify if a previous version of the record exists, and update the sequence value for the new record being created.

Once the record is created, it is ready to be spread through the network. This way, a peer can use whatever routing system it supports to make the record accessible to the remaining peers of the network.

On the other side, each peer must be able to get a record published by another node. It only needs to have the unique identifier used to publish the record to the network. Taking into account the routing system being used, we may obtain a set of occurrences of the record from the network. In this case, records can be compared using the sequence number, in order to obtain the most recent one.

As soon as the node has the most recent record, the signature and the validity must be verified, in order to conclude that the record is still valid and not compromised.

Finally, the network nodes may also republish their records, so that the records in the network continue to be valid to the other nodes.

## 概况

![](img/ipns-overview.png)

## API规范

  - [API_CORE](https://github.com/ipfs/specs/blob/master/API_CORE.md)

## Implementations

  - [js-ipfs](https://github.com/ipfs/js-ipfs/tree/master/packages/ipfs-core/src/ipns)
  - [go-namesys](https://github.com/ipfs/go-namesys)

## IPFS集成

#### Local record

This record is stored in the peer's repo datastore and contains the **latest** version of the IPNS record published by the provided key. This record is useful for republishing, as well as tracking the sequence number.

**Key format:** `/ipns/base32(<HASH>)`

Note: Base32 according to the [RFC4648](https://tools.ietf.org/html/rfc4648).

#### Routing record

The routing record is spread across the network according to the available routing systems.

**Key format:** `/ipns/BINARY_ID`

The two routing systems currently available in IPFS are the `DHT` and `pubsub`. As the `pubsub` topics must be `utf-8` for interoperability among different implementations
