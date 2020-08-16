#### socks5协议详解

```Sequence
participant client as a
participant socks5_server as b
participant internet as c
a->b: 1.request
b-->a: 2.response
a->b: 3.info
b--a: 4.establish-connection
a->b: 5.communication
b->c: 6.communication
```

1. 客户端连接上代理服务器之后需要发送请求告知服务器目前的socks协议版本以及支持的认证方式
2. 代理服务器收到请求后根据其设定的认证方式返回给客户端
3. 如果代理服务器不需要认证，客户端将直接向代理服务器发起真实请求
4. 代理服务器收到该请求之后连接客户端请求的目标服务器
5. 代理服务器开始转发客户端与目标服务器之间的流量


#### 认证过程

##### 客户端请求
Version| Method_Counts | Methods
:-: | :-: | :-:|
1byte | 1 byte | 1~255 byte(由Method_Counts决定) |

* VERSION SOCKS协议版本，目前固定0x05
* METHODS_COUNT 客户端支持的认证方法数量
* METHODS… 客户端支持的认证方法，每个方法占用1个字节

METHOD:
* 0x00 不需要认证（常用）
* 0x01 GSSAPI认证
* 0x02 账号密码认证（常用）
* 0x03 - 0x7F IANA分配
* 0x80 - 0xFE 私有方法保留
* 0xFF 无支持的认证方法

##### 服务端响应

###### 无需认证
Version| Method_Counts 
:-: | :-: |
1byte| 1byte
0x05| 0x00

* VERSION SOCKS协议版本，目前固定0x05
* METHOD 本次连接所用的认证方法，上例中为无需认证


###### 账号密码认证
Version| Method_Counts 
:-: | :-: |
1byte| 1byte
0x05| 0x02

##### 客户端发送账号密码

```
服务端返回的认证方法为0x02(账号密码认证)时，客户端会发送账号密码数据给代理服务器
```

Version| UserNameLength | UserName | PasswordLength | Password |
:-: | :-: | :-: | :-: |:-: |
1byte| 1byte | 1~255 byte | 1byte | 1~255 byte |
0x01| 0x01 | 0x0a| 0x01 | 0x0a

* VERSION 认证子协商版本（与SOCKS协议版本的0x05无关系）
* USERNAME_LENGTH 用户名长度
* USERNAME 用户名字节数组，长度为USERNAME_LENGTH
* PASSWORD_LENGTH 密码长度
* PASSWORD 密码字节数组，长度为PASSWORD_LENGTH

##### 服务端响应账号密码认证结果

Version| Status 
:-: | :-: |
1byte| 1byte
0x01| 0x00

* VERSION 认证子协商版本，与客户端VERSION字段一致
* STATUS 认证结果
  * 0x00 认证成功
  * 大于0x00 认证失败

#### 命令过程

```
认证成功后，客户端会发送连接命令给代理服务器，代理服务器会连接目标服务器，并返回连接结果
```

##### 用户请求

Version| Command | RSV | ADDRESS_TYPE | DST.ADDR | DST.PORT
:-: | :-: | :-: | :-: |:-: | :-: |
1byte| 1byte | 1byte | 1byte | 1~255 byte | 2byte

* VERSION SOCKS协议版本，固定0x05
* COMMAND 命令
  * 0x01 CONNECT 连接上游服务器
  * 0x02 BIND 绑定，客户端会接收来自代理服务器的链接，著名的FTP被动模式
  * 0x03 UDP ASSOCIATE UDP中继
* RSV 保留字段
* ADDRESS_TYPE 目标服务器地址类型
  * 0x01 IP V4地址
  * 0x03 域名地址(没有打错，就是没有0x02)，域名地址的第1个字节为域名长度，剩下字节为域名名称字节数组
  * 0x04 IP V6地址
* DST.ADDR 目标服务器地址
* DST.PORT 目标服务器端口

##### 响应
Version| RESPONSE | RSV | ADDRESS_TYPE | DST.ADDR | DST.PORT
:-: | :-: | :-: | :-: |:-: | :-: |
1byte| 1byte | 1byte | 1byte | 1~255 byte | 2byte

* VERSION SOCKS协议版本，固定0x05
* RESPONSE 响应命令
  * 0x00 代理服务器连接目标服务器成功
  * 0x01 代理服务器故障
  * 0x02 代理服务器规则集不允许连接
  * 0x03 网络无法访问
  * 0x04 目标服务器无法访问（主机名无效）
  * 0x05 连接目标服务器被拒绝
  * 0x06 TTL已过期
  * 0x07 不支持的命令
  * 0x08 不支持的目标服务器地址类型
  * 0x09 - 0xFF 未分配
* RSV 保留字段
* BND.ADDR 代理服务器连接目标服务器成功后的代理服务器IP
* BND.PORT 代理服务器连接目标服务器成功后的代理服务器端口

#### 实例

```Sequence
participant client as a
participant socks5_server as b
participant internet as c
a->b: 1.request 0x05 0x02 0x00 0x02
b-->a: 2.response 0x05 0x02
a->b: 3.info 0x01 ulen user plen pass
b-->a: 4.establish-connection 0x01 0x00
a->b: 5.communication 0x05 0x01 0x01 0x01 0x7f 0x00 0x00 0x01 0x00 0x50
b-->a: 6.communication 0x05 0x00 0x01 0x01 0x7f 0x00 0x00 0x01 0x00 0xaa 0xaa
```

##### 客户端请求代理服务器连接目标服务器
```
0x05 0x01 0x01 0x01 0x7f 0x00 0x00 0x01 0x00 0x50
```
* 0x05 SOCKS协议版本
* 0x01 CONNECT命令
* 0x01 RSV保留字段
* 0x01 地址类型为IPV4
* 0x7f 0x00 0x00 0x01 目标服务器IP为127.0.0.1
* 0x00 0x50 目标服务器端口为80

##### 代理服务器连接目标主机，并返回结果给客户端
```
0x05 0x00 0x01 0x01 0x7f 0x00 0x00 0x01 0x00 0xaa 0xaa
```
* 0x05 SOCKS5协议版本
* 0x00 连接成功
* 0x01 RSV保留字段
* 0x01 地址类型为IPV4
* 0x7f 0x00 0x00 0x01 代理服务器连接目标服务器成功后的代理服务器IP, 127.0.0.1
* 0xaa 0xaa 代理服务器连接目标服务器成功后的代理服务器端口（代理服务器使用该端口与目标服务器通信），本例端口号为43690



#### 代码框架
```golang
├── auth.go //鉴权方式相应的接口
├── auth_test.go // 上述接口测试
├── credentials.go // 提供用户名和用户密码授权
├── credentials_test.go // 上述接口测试
├── LICENSE
├── README.md
├── request.go // 处理客户端请求和回复响应
├── request_test.go 
├── resolver.go // 解析域名
├── resolver_test.go
├── ruleset.go // 设定socks5的规则
├── ruleset_test.go 
├── socks5.go // 提供服务等
└── socks5_test.go
```

