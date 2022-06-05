---
title: 关于HTTPS
tags: [计算机网络,HTTPS,TLS,RSA,ECDHE,数字证书,CA机构]
date: 2022-06-06 01:00:51
---






## TLS

TLS如何解决HTTP风险？

- 信息加密：HTTP交互信息被加密，第三方无法窃取。
- 校验机制：校验信息传输过程是否有第三方篡改，被篡改会警告。
- 身份证书：证明网站是真正的网站。

<!--more-->

## TLS握手

![image-20220605233707220](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206052337267.png?token=AHMFLSEZUU6NABAXMY5DRSDCTTG56)

每个框是个记录（record），记录TLS收发数据地基本单位。多个record可合并为同一个TCP包发送，通常是四个消息，即2RTT时延。不同的密钥交换算法，TLS握手过程可能有区别。

双方在加密应用信息时使用对称加密密钥，对称加密密钥不能被泄露，为了保证对称密钥的安全，使用非对称加密来保护对称加密密钥的协商过程，这个工作就是密钥交换算法完成的。

## RSA

传统的TLS握手基本使用RSA算法。服务端保存TLS证书，证书文件里有一对公私钥，TLS握手过程把公钥传给客户端，私钥存在服务端不能被窃取。

RSA算法中，客户端产生随机密钥，通过服务端传来的公钥进行加密再传给服务端。非对称加密算法中，公钥加密的消息只能私钥解密，服务端解密后，双方得到相同的密钥，之后用这个相同的密钥加密消息。

### 第一次握手

客户发 **Client Hello**：TLS版本号，支持密码套件列表，**生成的随机数（Client Random）**，这个Client Random存在服务端，用来生成对称密钥。

### 第二次握手

服务端收到Client Hello，确认版本，选择密码套件，生成**随机数（Server Random）**。

返回**Server Hello**消息，如密码套件Cipher Suite: TLS_RSA_WITH_AES_128_GCM_SHA256。形式基本是密钥交换算法+签名算法+对称加密算法+摘要算法。WITH前有2个单词，如果只有一个（例RSA），那就都用RSA。握手后通信用AES对称加密，密钥长度128位，分组模式GCM。摘要算法SHA256，消息认证和产生随机数。

然后服务端发送**Server Certificate**证明自己身份，消息里包含数字证书。然后发送**Server Hello Done**，表示该给的东西都给了。

### 第三次握手

客户端验证证书后，可信就往下继续。客户端产生新的**随机数（pre-master）**，用RSA公钥加密该随机数，然后**Change Cipher Key Exchange**，服务端收到，用私钥解密，得到pre-master。

共享三个随机数，根据这三个生成**会话密钥（Master Secret）**。生成完客户端发送**Change Cipher Spec**，告诉服务端开始使用会话密钥发送消息。再发送**Encrypted Handshake Message (Finished)**，把之前的发送的数据做摘要，用Master Secret加密发送到服务端做验证。

### 第四次握手

服务端也发送**Change Cipher Spec**和**Encrypted Handshake Message **，握手完成。

### 缺陷

RSA密钥协商不支持前向保密。客户端发送的随机数是用服务端公钥解密的，因此如果服务端私钥被泄露了，过去所有截获的TLS通讯密文都会被破解。因此产生DH密钥交换算法，在这个过程中，双端生成各自的随机数，作为私钥，然后用公开的DH算法生成公钥，TLS握手过程中，双方交换公钥，然后双方根据各自持有的材料算出一个随机数，这个随机数双方是一样的，即作为后续对称加密的会话密钥。

DH密钥交换中，即使第三方截取公钥，不知道私钥情况下，也无法算出密钥。私钥不经过网络传输，而且在会话结束后，双方会将私钥删除，即使长期使用的用来加密证书的公私钥对（Apub/Apriv）被非法获取，以往监听到的加密数据也无法被破解，除非有能力拿到每次会话的DH私钥（几乎不可能）。

[(32 封私信 / 80 条消息) 如何理解前向安全性？和完美前向保密（perfect forward secrecy）区别？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/45203206)

DH算法性能不太行，一般会用ECDHE密钥协商算法。

## 数字证书和CA机构

数字证书包含：

- 公钥
- 持有者信息
- 证书认证机构（CA）的信息
- CA对这份文件的数字签名及使用的算法
- 证书有效期
- 额外信息

数字证书来认证公钥持有者的身份，防止第三方冒充。即是告诉客户端，该服务端是否是合法的，因为只有证书合法，才代表服务端身份是可信的。

服务端的证书是由CA（Certificate Authority，证书认证机构）签名的，签名的作用可以避免中间人在获取证书时对证书内容的篡改。

### 数字证书签发和验证流程

![image-20220605231019324](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206052310398.png?token=AHMFLSDCMDLTXLE4QV7XR5TCTTDZO)

CA签发过程在左边：

- 首先CA把持有者公钥、用途、颁发者、有效时间等信息打包，然后进行Hash计算，得到Hash值。

- 然后CA使用自己的私钥将Hash值加密，生成Certificate Signature，也就是CA对证书**签名**的过程。

- 最后将Certificate Signature添加在文件证书上，形成数字证书。

客户端校验在右边：

- 客户端通过同样的Hash算法获取证书Hash值H1
- 浏览器和操作系统中集成了CA的公钥，浏览器收到证书后可以使用CA公钥解密Certificate Signature内容，得到H2
- 比较H1和H2，相同则可信

### 证书链

还存在证书信任链的问题。我们向CA申请的证书一般都不是根证书签发的，而是由中间证书签发的。

![image-20220605231910601](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206052319653.png?token=AHMFLSCTZNEEYJ4S5U6ZPZ3CTTE2W)

多级关系：

- 客户端收到\*.bilibili.com的证书后，发现签发者不是根证书，就无法根据本地已有的根证书中的公钥去验证\*.bilibili.com证书是否可信。于是客户端根据签发者找到证书的颁发机构是GlobalSign RSA OV SSL CA 2018，然后向CA请该求中间证书。
- 请求到证书后发现GlobalSign RSA OV SSL CA 2018证书是由GlobalSign签发的，发现也不是根证书，于是再向CA请求中间证书
- 然后发现GlobalSign是由GlobalSign Root CA - R1签发，再往上也没有签发机构，说明是根证书，即自签证书。浏览器检查这个证书是否已经预载在根证书清单上，如果有就可以通过根证书中的公钥验证下级的证书，如果验证通过，就认为该中间证书是可信的。然后再验证下一层证书可信性，最终验证到\*.bilibili.com，如果验证通过，就可以信任该证书。

这个过程中，一开始客户端只信任根证书GlobalSign Root CA - R1，GlobalSign Root CA - R1信任GlobalSign证书，GlobalSign证书信任GlobalSign RSA OV SSL CA 2018，GlobalSign RSA OV SSL CA 2018信任\*.bilibili.com证书。

再往上，用户信任操作系统和浏览器的软件商，所以软件预载的根证书都被信任。

证书信任链流程：

![image-20220605232948160](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206052329198.png?token=AHMFLSAR7UE3ITIBUAHCWVTCTTGCQ)

为什么要搞证书链这么麻烦的流程？这么多中间层？

是为了保证根证书绝对安全性，将根证书隔离地约严格越好，不然根证书失守，整个信任链就都有问题了。



## DH算法

### 数学基础

如果对于一个整数b和质数p的一个原根a，可以找到唯一的指数i使得：
$$
a^i (mod \quad p) =b
$$
成立，那么指数i称为b的以a为底的模p的离散对数，b是真数。

底数a和模数p是公共参数，知道了对数i，可以很容易地算出真数；但反过来只知道真数b，则很难推出指数i。

### DH算法

A端和B端协商用DH算法，基于离散对数，双方确定模数和底数为参数，用P、G代称。

然后双方各自产生随机整数作为私钥，A的私钥Apriv，B的私钥Bpriv。

可计算公钥
$$
Apub = G^{Apriv}(mod\quad P)
$$

$$
Bpub = G^{Bpriv}(mod\quad P)
$$

因此碍于计算机的计算能力，只知道pub计算出priv是非常困难的。

交换DH公钥后，A端有：P、G、Apriv、Apub、Bpub，B端有：P、G、Bpriv、Bpub、Apub。

A执行运算$K=Bpub^{Apriv}(mod\quad P)$，B执行$K=Apub^{Bpriv}(mod\quad P)$

都是K的原因是离散对数具有幂运算交换不变性。
$$
K = Bpub^{Apriv}(mod\quad P)=(G^{Bpriv}(mod\quad P))^{Apriv}(mod\quad P)=G^{Apriv\cdot Bpriv}(mod\quad P)
$$
非法的第三方可能能够得到：P、G、Apub、Bpub很难计算出会话密钥。

### DHE算法

根据私钥生成方法，DH算法有两种实现：

- static DH，基本废弃
- DHE，现在常用

static DH有一方的私钥是静态的，一般是服务端固定，客户端私钥随机生成。时间延长，第三方可能可以破解出服务端私钥，于是之前截获的加密数据会被破解（破解出的服务端私钥+截取的客户端pub=会话密钥），因此staticDH不具有前向安全性。

因此干脆让双方的私钥在每次交换时都是随机生成的、临时的。也就是DHE(ephemeral临时)。每次的通信过程的私钥都无关系，保证前向安全。

### ECDHE

DHE计算性能不佳，碍于大量乘法计算。ECDHE算法在DHE算法基础上利用ECC椭圆曲线特性，用更少的计算量计算出公钥，及会话密钥。

过程：

- 双端确定用哪种椭圆曲线，和曲线上的基点G，这两个参数公开；
- 双端随机生成一个随机数作为私钥d，与基点G相乘得到公钥Q（Q=dG），A的公私钥是Q1和d1，B的是Q2和d2；
- 交换公钥，A计算点（x1,y1) = d1Q2，B计算点(x2,y2) = d2Q1，椭圆曲线上满足乘法交换和结合律，d1Q2=d1d2G=d2d1G=d2Q1，因此双方x坐标是相同的，作为共享密钥。

ECDHE在TLS第四次握手前，客户端就已经发送了加密的HTTP数据，相比RSA握手省1RTT，被称为TLS False Start，在连接没完全建立前就发送了应用数据。

- TLS第一次握手：Client Hello，发送TLS版本号、密码套件列表、Client Random
- TLS第二次握手：Server Hello，选择的密码套件，Server Random；发送Certificate，有证书信息；发送Server Key Exchange，选择名为name_curve的椭圆曲线，也就选好了基点G，生成随机数作为服务端私钥，根据基点G和私钥计算服务器公钥，公开给客户端，RSA签名算法给公钥做个签名；Server Hello Done
- TLS第三次握手：客户端校验证书合法，生成随机数作为私钥，根据前面信息生成客户端椭圆曲线公钥，发送Client Key Exchange；用Client Random + Server Random + x生成最终的会话密钥；算完后发送Change Cipher Spec；发送Encrypted Handshake Message，摘要信息；
- TLS第四次握手：服务端发Change Cipher Spec，Encrypted Handshake Message。

