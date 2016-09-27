通讯协议
=======

客户端和服务端均采用相同的协议格式与网关进行通讯，消息由4个字节包长度信息和变长的包体组成：

```
+--------+--------+
| Length | Packet |
+--------+--------+
  4 byte   Length
```

每个消息的包体可以再分解为4个字节的虚拟连接ID和变长的消息内容两部分：

```
+---------+-------------+
| Conn ID |   Message   |
+---------+-------------+
   4 byte    Length - 4
```

`Conn ID`是虚拟连接的唯一标识，网关通过识别`Conn ID`来转发消息。

当`Conn ID`为0时，`Message`的内容被网关进一步解析并作为指令进行处理。

网关指令固定以一个字节的指令类型字段开头，指令参数根据指令类型而定：

```
+--------+------------------+
|  Type  |       Args       |
+--------+------------------+
  1 byte    Length - 4 - 1
```

目前支持的指令如下：

| **Type** | **用途** | **Args** | **说明** |
| ---- | ---- | ---- | ---- |
| 0 | New | 无 | 客户端或服务端发送給网关，申请新建虚拟连接ID |
| 1 | Open | Conn ID | 由网关发送给虚拟连接创建者，分配新虚拟连接ID |
| 2 | Dial | Conn ID + Remote ID | 将虚拟连接关联到服务端或客户端 |
| 3 | Refuse | Conn ID | 网关下发此消息告诉连接创建者虚拟连接初始化失败 |
| 4 | Accept | Conn ID + Remote ID | 由网关发送给虚拟连接两端，虚拟连接初始化完成可用于通讯 |
| 5 | Close | Conn ID | 客户端、服务端、网关都有可能发送次消息 |
| 6 | Ping | 无 | 网关下发给客户端和服务端，收到后应立即响应 |

特殊说明：

+ 协议不只允许客户端主动连接服务端，也允许服务端主动连接客户端。
+ 新建虚拟连接的时候，网关会把虚拟连接信息发送给两端。
+ 网关发送`Accept`指令给客户端时，`Remote ID`为服务端ID。
+ 网关发送`Accept`指令给服务端时，`Remote ID`为客户端ID。

完整的消息示例：

+ 客户端创建虚拟连接：

	```
	+------------+-------------+---------+
	| Length = 5 | Conn ID = 0 | CMD = 1 |
	+------------+-------------+---------+
        4 byte        4 byte      1 byte
	```

+ 网关分配虚拟连接ID：

	```
	+------------+-------------+---------+----------------+
	| Length = 9 | Conn ID = 0 | CMD = 1 | Conn ID = 9527 |
	+------------+-------------+---------+----------------+
        4 byte        4 byte      1 byte       4 byte
	```

+ 将虚拟连接关联到到服务端：

	```
	+-------------+-------------+---------+----------------+----------------+
	| Length = 13 | Conn ID = 0 | CMD = 2 | Conn ID = 9527 |  Server ID = 1 |
	+-------------+-------------+---------+----------------+----------------+
         4 byte        4 byte      1 byte       4 byte           4 byte
	```

+ 网关发送虚拟连接信息给服务端：

	```
	+-------------+-------------+---------+----------------+---------------+
	| Length = 13 | Conn ID = 0 | CMD = 4 | Conn ID = 9527 | Client ID = 1 |
	+-------------+-------------+---------+----------------+---------------+
	    4 byte        4 byte       1 byte       4 byte           4 byte
	```

+ 网关发送虚拟连接信息给客户端：

	```
	+-------------+-------------+---------+----------------+---------------+
	| Length = 13 | Conn ID = 0 | CMD = 4 | Conn ID = 9527 | Server ID = 1 |
	+-------------+-------------+---------+----------------+---------------+
	    4 byte        4 byte       1 byte       4 byte           4 byte
	```

+ 客户端通过虚拟连接发送"Hello"到服务端：

	```
	+------------+----------------+-------------------+
	| Length = 9 | Conn ID = 9527 | Message = "Hello" |
	+------------+----------------+-------------------+
        4 byte         4 byte            5 byte
	```

+ 客户端关闭虚拟连接：

	```
	+------------+-------------+---------+----------------+
	| Length = 9 | Conn ID = 0 | CMD = 5 | Conn ID = 9527 |
	+------------+-------------+---------+----------------+
        4 byte        4 byte      1 byte       4 byte
	```

握手协议
=======

服务端在连接网关时需要先进行握手来验证服务端的合法性。

握手过程如下：

0. 合法的服务端应持有正确的网关秘钥。
1. 网关在接受到新的服务端连接之后，向新连接发送一个`uint64`范围内的随机数作为挑战码。
2. 服务端收到挑战码后，拿出秘钥，计算 `MD5(挑战码 + 秘钥)`，得到验证码。
3. 服务端将验证码和自身节点ID一起发送给网关。
4. 网关收到消息后同样计算 `MD5(挑战码 + 秘钥)`，跟收到的验证码比对是否一致。
5. 验证码比对一致，网关将新连接登记为对应节点ID的连接。

握手下行数据格式：

```
+----------------+
| Challenge Code |
+----------------+
      8 byte
```

握手上行数据格式：

```
+-----------+-----------+
|    MD5    | Server ID |
+-----------+-----------+
   16 byte      4 byte
```

使用示例
=======

示例1 - 以客户端身份连接到网关：

```go
client, err := DialClient(
	GatewayAddr,   // 网关地址
	MyMsgFormat,   // 消息格式
	MyMemPool,     // 内存池
	MaxPacketSize, // 包体积限制
	BufferSize,    // 预读所用的缓冲区大小
	SendChanSize,  // 异步发送用的chan缓冲区大小
)
```

示例2 - 以服务端身份连接到网关：

```go
server, err := DialServer(
	GatewayAddr,   // 网关地址
	MyMsgFormat,   // 消息格式
	MyMemPool,     // 内存池
	ServerID,      // 服务端ID
	AuthKey,       // 身份验证用的Key
	AuthTimeout,   // 身份验证IO等待超时时间
	MaxPacketSize, // 包体积限制
	BufferSize,    // 预读所用的缓冲区大小
	SendChanSize,  // 异步发送用的chan缓冲区大小
)
```

示例3 - 创建一个虚拟连接：

```go
conn, connID, err := client.Dial(ServerID)
```

示例4 - 接收一个虚拟连接：

```go
conn, connID, clientID, err := server.Accept()
```

示例5 - 以JSON格式发送一个消息：

```go
var my MyMessage
buf, err := json.Marshal(&msg)
conn.Send(&buf)
```

例6 - 以JSON格式接收一个消息：

```go
var my MyMessage
buf, err := conn.Receive()
json.Unmarshal(*(buf.(*[]byte)), &my)
```