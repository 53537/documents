

# 证书

### 设置密码变量 `PASS`

```shell
PASS=confidential
```



### 签发 CA 证书

```shell
openssl req -new -x509 -days 1825 -keyout ca.key -out ca.pem -nodes \
            -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority"
```



### 导入 CA 证书到 Keystore

```shell
keytool -keystore truststore.p12 -deststoretype pkcs12 -storepass $PASS \
        -import -alias ca-root -file ca.pem -noprompt
```



### 为 Zookeeper、Kafka、Client 签发证书

*依次重复以下步骤，为 Zookeeper & Kafka 生成证书并导入相应的 Keystore*



#### 证书列表

- ***zookeeper***		Zookeeper Server 运行证书
- ***kafka***                Kafka Server 运行证书
- ***client***                第三方客户端连接 Kafka 所用证书(如 *Logstash、Filebeat*)



#### 设置名称变量 `NAME`

```shell
NAME=zookeeper
```



#### 创建新的证书

```shell
keytool -keystore $NAME.p12 -storepass $PASS -alias $NAME -validity 1825 \
		    -genkey -keyalg RSA -keysize 2048 -keypass $PASS -deststoretype pkcs12 \
		    -dname "C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=$NAME"
```



#### 生成证书签名请求

```shell
keytool -keystore $NAME.p12 -storepass $PASS -alias $NAME -certreq -file $NAME-cert-file
```



#### 使用 CA 签发证书

```shell
openssl x509 -req -CA ca.pem -CAkey ca.key -in $NAME-cert-file -out $NAME.pem \
             -days 1825 -CAcreateserial -extensions SAN \
		         -extfile <(printf "\n[SAN]\nsubjectAltName=DNS:*.zk-svc,IP:10.10.20.151,IP:10.10.20.152,IP:10.10.20.153")
```



#### 导入CA证书到 Keystore

```shell
keytool -keystore $NAME.p12 -deststoretype pkcs12 -storepass $PASS -import -alias ca-root -file ca.pem -noprompt
```



#### 导入证书到 Keystore

```shell
keytool -keystore $NAME.p12 -deststoretype pkcs12 -storepass $PASS -import -alias $NAME -file $NAME.pem
```



# Zookeeper

## 配置 ZooKeeper

#### 在 *zkServer.sh* 中设置`SERVER_JVMFLAGS` 变量

```shell
export SERVER_JVMFLAGS="
	-Dzookeeper.serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
	-Dzookeeper.ssl.keyStore.location=/opt/zookeeper/conf/zookeeper.p12
	-Dzookeeper.ssl.keyStore.password=confidential
	-Dzookeeper.ssl.trustStore.location=/opt/zookeeper/conf/zookeeper.p12
	-Dzookeeper.ssl.trustStore.password=confidential"
```



#### 在 *zkCli.sh* 中设置变量 `CLIENT_JVMFLAGS`

```shell
export CLIENT_JVMFLAGS="
	-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
	-Dzookeeper.client.secure=true
	-Dzookeeper.ssl.keyStore.location=/opt/zookeeper/conf/zookeeper.p12
	-Dzookeeper.ssl.keyStore.password=confidential
	-Dzookeeper.ssl.trustStore.location=/opt/zookeeper/conf/truststore.p12
	-Dzookeeper.ssl.trustStore.password=confidential"
```



#### Zookeeper 配置文件 zoo.cfg

```shell
dataDir=/data/zk
secureClientPort=2182
syncLimit=5
initLimit=10
tickTime=2000
maxClientCnxns=500
standaloneEnabled=false
autopurge.purgeInterval=24
autopurge.snapRetainCount=7
admin.enableServer=false
reconfigEnabled=true

4lw.commands.whitelist=ruok, cons, stat, mntr
metricsProvider.httpPort=7000
metricsProvider.exportJvmInfo=true
metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
authProvider.1=org.apache.zookeeper.server.auth.X509AuthenticationProvider
ssl.quorum.keyStore.location=/opt/zookeeper/conf/zookeeper.p12
ssl.quorum.keyStore.password=confidential
ssl.quorum.trustStore.location=/opt/zookeeper/conf/truststore.p12
ssl.quorum.trustStore.password=confidential
```

- **`secureClientPort`**
  
  仅支持 TLS 连接，如果需开启非加密端口，可通过配置 `clientPort` 实现
  
- **`authProvider.1`**

  通过客户端证书进行认证

- **`ssl.quorum.keyStore.location`**

  Keystore 路径

- **ssl.quorum.keyStore.password**

  Keystore 密码

- **`ssl.quorum.trustStore.location`**

  Truststore 路径

- **`ssl.quorum.trustStore.password`**

  Truststore 密码

  

## 启动 Zookeeper

```shell
/opt/zookeeper/bin/zkServer.sh --config /opt/zookeeper/conf start
```



## 验证

### 通过日志确认是否使用了 TLS

如果配置正确，将会看到如下日志信息

```
INFO  [main:ServerCnxnFactory@169] - Using org.apache.zookeeper.server.NettyServerCnxnFactory as server connection factory
...
...
INFO  [main:QuorumPeer@1999] - Using TLS encrypted quorum communication
...
...
INFO  [ListenerHandler...:3888:QuorumCnxManager$Listener$ListenerHandler@1127] - Creating TLS-only quorum server socket
```



### 使用 `openssl s_client` 进行测试连接

```shell
openssl s_client -debug -connect 10.10.20.161:2281 -tls1_2
```

#### Output

```
CONNECTED(00000003)
write to 0x19fd550 [0x1a18c13] (289 bytes => 289 (0x121))
0000 - 16 03 01 01 1c 01 00 01-18 03 03 a5 e5 ec c2 4b   ...............K
...
0110 - 03 01 03 02 03 03 02 01-02 02 02 03 00 0f 00 01   ................
0120 - 01                                                .
read from 0x19fd550 [0x1a146c3] (5 bytes => 5 (0x5))
0000 - 16 03 03 09 54                                    ....T
read from 0x19fd550 [0x1a146c8] (2388 bytes => 2388 (0x954))
0000 - 02 00 00 4d 03 03 5e d4-c1 f9 7b 12 2b 93 7d 7f   ...M..^...{.+.}.
...
0950 - 0e                                                .
0954 - <SPACES/NULS>
depth=1 C = CN, ST = Shanghai, L = Shanghai, O = Services, CN = Services Security Authority
verify error:num=19:self signed certificate in certificate chain
write to 0x19fd550 [0x1a1e160] (12 bytes => 12 (0xC))
0000 - 16 03 03 00 07 0b 00 00-03                        .........
000c - <SPACES/NULS>
write to 0x19fd550 [0x1a1e160] (75 bytes => 75 (0x4B))
0000 - 16 03 03 00 46 10 00 00-42 41 04 c5 d7 c8 ef a1   ....F...BA......
0010 - 8f 51 45 a7 d7 4c be 5f-da 46 e2 b7 f5 ea 38 19   .QE..L._.F....8.
0020 - 68 fc ec 89 4c 5b 1f 92-83 63 31 a8 20 e4 33 dd   h...L[...c1. .3.
0030 - 2a f1 7b 32 e6 2a ca f0-74 78 1d 3a cd e4 c0 10   *.{2.*..tx.:....
0040 - 72 b0 9b ec f0 d1 c2 22-75 36 c2                  r......"u6.
write to 0x19fd550 [0x1a1e160] (6 bytes => 6 (0x6))
0000 - 14 03 03 00 01 01                                 ......
write to 0x19fd550 [0x1a1e160] (45 bytes => 45 (0x2D))
0000 - 16 03 03 00 28 76 85 3d-c3 8c 8b 5c f1 b8 89 0c   ....(v.=...\....
0010 - 81 d3 47 08 63 59 f3 aa-b0 a4 b8 72 2c e5 52 ca   ..G.cY.....r,.R.
0020 - 7e ed d2 56 03 bb e2 16-29 e8 d4 88 18            ~..V....)....
read from 0x19fd550 [0x1a146c3] (5 bytes => 5 (0x5))
0000 - 15 03 03 00 02                                    .....
read from 0x19fd550 [0x1a146c8] (2 bytes => 2 (0x2))
0000 - 02 2a                                             .*
139634466551696:error:14094412:SSL routines:ssl3_read_bytes:sslv3 alert bad certificate:s3_pkt.c:1493:SSL alert number 42
139634466551696:error:1409E0E5:SSL routines:ssl3_write_bytes:ssl handshake failure:s3_pkt.c:659:
---
Certificate chain
 0 s:/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=zookeeper
   i:/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority
 1 s:/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority
   i:/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDXDCCAkSgAwIBAgIJANyktokIuT7ZMA0GCSqGSIb3DQEBCwUAMGwxCzAJBgNV
...
-----END CERTIFICATE-----
subject=/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=zookeeper
issuer=/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority
---
Acceptable client certificate CA names
/C=CN/ST=Shanghai/L=Shanghai/O=Services/CN=Services Security Authority
Client Certificate Types: RSA sign, DSA sign, ECDSA sign
Requested Signature Algorithms: ECDSA+SHA512:RSA+SHA512:ECDSA+SHA384:RSA+SHA384:ECDSA+SHA256:RSA+SHA256:DSA+SHA256:ECDSA+SHA224:RSA+SHA224:DSA+SHA224:ECDSA+SHA1:RSA+SHA1:DSA+SHA1
Shared Requested Signature Algorithms: ECDSA+SHA512:RSA+SHA512:ECDSA+SHA384:RSA+SHA384:ECDSA+SHA256:RSA+SHA256:DSA+SHA256:ECDSA+SHA224:RSA+SHA224:DSA+SHA224:ECDSA+SHA1:RSA+SHA1:DSA+SHA1
Peer signing digest: SHA512
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 2400 bytes and written 138 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 5ED4C1F94151453EE22640F7D18C7E35EFCB26BC0EB354C7B06E99D6E3F839B8
    Session-ID-ctx:
    Master-Key: 5C00D2A13EE851DA7D16A6AD2B2807C19458FB9A145ED1588F54A0512E32FE1B37686AB1AF2ACC23B400A4B492AAA0BB
    Key-Arg   : None
    Krb5 Principal: None
    PSK identity: None
    PSK identity hint: None
    Start Time: 1591001593
    Timeout   : 7200 (sec)
    Verify return code: 19 (self signed certificate in certificate chain)
---
```





### 使用 zkCli.sh 连接到 Zookeeper

```shell
./bin/zkCli.sh -server 10.10.20.161:2281
```



#### 如果配置正确，将会看到如下输出

##### On Client side:

```
2020-06-01 14:43:57,462 [myid:] - INFO  [main:X509Util@77] - Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
2020-06-01 14:43:57,572 [myid:] - INFO  [main:ClientCnxnSocket@239] - jute.maxbuffer value is 1048575 Bytes
2020-06-01 14:43:57,578 [myid:] - INFO  [main:ClientCnxn@1703] - zookeeper.request.timeout value is 0. feature enabled=false
Welcome to ZooKeeper!
2020-06-01 14:43:57,584 [myid:10.10.20.161:2281] - INFO  [main-SendThread(10.10.20.161:2281):ClientCnxn$SendThread@1154] - Opening socket connection to server mq-1/10.10.20.161:2281.
2020-06-01 14:43:57,584 [myid:10.10.20.161:2281] - INFO  [main-SendThread(10.10.20.161:2281):ClientCnxn$SendThread@1156] - SASL config status: Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
[zk: 10.10.20.161:2281(CONNECTING) 0] 2020-06-01 14:43:57,961 [myid:10.10.20.161:2281] - INFO  [nioEventLoopGroup-2-1:ClientCnxnSocketNetty$ZKClientPipelineFactory@454] - SSL handler added for channel: [id: 0x68fd6785]
2020-06-01 14:43:57,966 [myid:10.10.20.161:2281] - INFO  [nioEventLoopGroup-2-1:ClientCnxn$SendThread@986] - Socket connection established, initiating session, client: /10.10.20.161:32640, server: mq-1/10.10.20.161:2281
2020-06-01 14:43:57,967 [myid:10.10.20.161:2281] - INFO  [nioEventLoopGroup-2-1:ClientCnxnSocketNetty$1@184] - channel is connected: [id: 0x68fd6785, L:/10.10.20.161:32640 - R:mq-1/10.10.20.161:2281]
2020-06-01 14:43:58,781 [myid:10.10.20.161:2281] - INFO  [nioEventLoopGroup-2-1:ClientCnxn$SendThread@1420] - Session establishment complete on server mq-1/10.10.20.161:2281, session id = 0xb33bfd10000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null

[zk: 10.10.20.161:2281(CONNECTED) 0]
```



##### On Server side:

```
2020-06-01 14:43:58,739 [myid:0] - INFO  [nioEventLoopGroup-7-1:X509AuthenticationProvider@166] - Authenticated Id 'C=CN/ST\=Shanghai/L\=Shanghai/O\=Services/CN\=zookeeper' for Scheme 'x509'
2020-06-01 14:43:58,757 [myid:0] - WARN  [QuorumPeer[myid=0](plain=0.0.0.0:2181)(secure=0.0.0.0:2281):Follower@170] - Got zxid 0x300000001 expected 0x1
2020-06-01 14:43:58,757 [myid:0] - INFO  [SyncThread:0:FileTxnLog@284] - Creating new log file: log.300000001
2020-06-01 14:43:58,764 [myid:0] - INFO  [CommitProcessor:0:LearnerSessionTracker@116] - Committing global session 0xb33bfd10000
```



## Troubleshooting

### Error

#### Error Message: *Caused by: javax.net.ssl.SSLHandshakeException: no cipher suites in common*

- 可能使用了太高版本的Java，比如 *OpenJDK 14*

- 或者没有正确的设置变量，在文件 *zkServer.sh* 中



#### Error Message: *DerInputStream.getLength(): lengthTag=109, too big*

- 在 keytool 导入证书时使用  `-deststoretype pkcs12` 参数



## References

[1]. [Encrypt Communication Between Kafka And ZooKeeper With TLS](https://juplo.de/encrypt-communication-between-kafka-and-zookeeper-with-tls/)

[2]. [ZooKeeper SSL User Guide](https://cwiki.apache.org/confluence/display/ZOOKEEPER/ZooKeeper+SSL+User+Guide)

[3.] [ZooKeeper Administrator's Guide](https://zookeeper.apache.org/doc/r3.6.1/zookeeperAdmin.html#Quorum+TLS)

