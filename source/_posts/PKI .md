---
title: PKI
date: 2022-07-21 14:22:14
tags:
    - pki


---

<!--more-->

# PKI

​	学习了PKI两周左右，做一下总结。

## 1.**介绍**

​	PKI，即公钥基础设施（Public key infrastructure），主要用于**证书的颁发**、证书的管理、证书的存储、**证书的撤销**等。主要作用与证书颁发与证书撤销。

## 2.相关名词

​	介绍PKI之前，先介绍几个相关名词

### 	2.1 对称加密与非对称加密

1. 对称加密简单来说就是加密和解密都使用同一个密钥，比较好理解。
2. 非对称加密就是通过公钥与私钥进行加解密。

非对称加解密方式：

```
①公钥加密+私钥解密（普通加密）
②私钥加密+公钥解密（数字签名/验签）
```

​	非对称加解密也就只有这两种方式，**值得注意的是公钥与私钥是无法相互推理出的**。

#### 2.1.1 公钥私钥产生方式：

```
系统随机产生的一个随机数，然后经过特定函数（RSA等）生成公钥私钥，然后这个随机数就会被丢弃
```

#### 2.1.2 优缺点

|          |  对称加密  | 非对称加密 |
| -------- | :--------: | :--------: |
| 加密效率 |     快     |     慢     |
| 安全性   |    一般    |     高     |
| 加密方式 | 适合点对点 | 适合点对面 |

​	所以一般情况，先用对称加密的公钥A加密明文形成密文A，再使用非对称加密的私钥B加密对称加密的公钥A形成密文B，再用非堆成加密的公钥B解密密文B得到对称加密的公钥A，再用公钥A解密密文B得到明文。

### 2.2 X.509证书格式

​	X.509一般使用PEM（Privary Enhanced Mail）格式来存储证书相关文件。证书文件名称后缀一般为.crt或者.cer。

​	PEM格式采用文本方式来进行存储，一般包括首尾标记和内容块，内容块采用Base64进行编码。

#### 2.2.1 编码格式总结

​	X.509 DER(Distinguished Encoding Rules)编码，后缀为：.der、.cer、.crt。

​	X.509 Base64编码（PEM格式）后缀为：.pem、.cer、.crt。

**X.509证书格式一般包括：**

1. （用户）公钥
2. 数字签名
3. 证书有效期
4. 用户信息（服务器的话就是域名）
5. 版本
6. 等等

其中前四个是一定有的，其余的都是可添加的。

### 2.3 散列函数

​		经过MD5、SHA-1、SHA-2、SHA-256等哈希算法得到的哈希值称为指纹。

#### 2.3.1 散列特点

1. 固定大小（过滤作用）：不管输入长度为多少，生成的散列值长度固定
2. 雪崩效应：修改了一个字节也会得到完全彻底不一样的散列值
3. 单向不可逆
4. 冲突避免

### 2.4 数字签名

​	通过非对称加密的私钥加密处理就是数字签名。

#### 2.4.1签发与验签过程

​	有个很经典的图，讲了大概流程：

![img](https://s2.loli.net/2022/08/07/45JG6w2NjfAD8xl.webp)

### 2.5 CA

​	CA是Certificate Authority的缩写，也叫“证书授权中心”。CA 是负责签发证书、认证证书、管理已颁发证书的机关。它要制定政策和具体步骤来验证、识别用户身份，并对用户证书进行签名，以确保证书持有者的身份和公钥的拥有权。

### 2.6 RA

​	简单理解，CA是签发证书的，但是RA是审核证书里面的信息的，只有RA审核证书里的信息没有问题，CA才会去签发这张证书

### 2.7 CRL与OCSP

​	CRL（certificate revocation list）证书吊销列表

```
	包含了证书颁发机构 (CA) 已经吊销的证书的序列号及其吊销日期。 CRL 文件中还包含证书颁发机构信息、吊销列表失效时间和下一次更新时间，以及采用的签名算法等。证书吊销列表最短的有效期为一个小时，一般为 1 天，甚至一个月不等，由各个证书颁发机构在设置其证书颁发系统时设置。
```

​	OCSP(Online Certificate Status Protocol)  证书状态在线查询协议

```
 	IETF 颁布的用于实时查询数字证书在某一时间是否有效的标准。
```

​		一般 CA 都只是 每隔一定时间 ( 几天或几个月 ) 才发布新的吊销列表，可以看出： CRL 是 不能及时反映证书的实际状态的。而 OCSP 就能满足实时在线查询证书状态的要求。

## 3. PKI工作流程

​	针对一个使用PKI的网络，配置PKI的目的就是为指定的实体向CA申请一个本地证书，并由设备对证书的有效性进行验证。下面是PKI的工作过程：

(1)   实体向RA提出证书申请；

(2)   RA审核实体身份，将实体身份信息和公开密钥以数字签名的方式发送给CA；

(3)   CA验证数字签名，同意实体的申请，颁发证书；

(4)   RA接收CA返回的证书，发送到LDAP服务器（或其他形式的发布点）以提供目录浏览服务，并通知实体证书发行成功；

(5)   实体获取证书，利用该证书可以与其他实体使用加密、数字签名进行安全通信；

(6)   实体希望撤消自己的证书时，向CA提交申请。CA批准实体撤消证书，并更新CRL，发布到LDAP服务器（或其他形式的发布点）。

![image-20220706230615562](https://s2.loli.net/2022/07/06/sOr31JWXdxcVb4C.png)

## 4.CA授权过程



### 4.1 证书校验

客户端验证服务端下发的证书，主要包括以下几个方面：

- 1、校验证书是否是`受信任`的`CA根证书`颁发机构颁发；
- 2、校验证书是否在上级证书的`吊销列表`；
- 3、校验证书`是否过期`；
- 4、校验证书`域名`是否`一致`。

## 5.用户与用户之间交互



## 6.离线申请

1. 用户产生RSA密钥对，申请本地证书携带公钥（私钥自己保存）
2. 配置PKI用户，实现申请本地证书时携带PKI用户信息来标识PKI用户身份
3. PKI用户离线申请本地证书，生成本地证书请求文件
4. **通过带外的方式（邮件或其他方式）发送本地证书请求文件来申请本地证书，并通过带外方式下载本地证书。**
5. 安装本地证书，使得设备可以使用证书来保护通信。

## 7.证书链

​	打开一个支持 HTTPS 的网站，可以点击地址栏的小锁，查看这个网站的证书信息，可以看到证书路径：

![img](https://s2.loli.net/2022/08/07/xERGSsqQOt9wUMN.webp)

这里有 3 级证书，它们被分为：

- **end-user**：即 [www.google.com](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.google.com)，是该网站使用 HTTPS 安装的数字证书
- **intermediates**：即 Google Internet Authority G3，是给 end-user 签发证书的中间 CA 的证书，中间 CA 可能不止一个
- **root**：即 Google Trust Services - GlobalSign Root CA-R2，是根 CA 的证书，它给中间 CA 签发证书

### 7.1 证书链的形成

1. root 证书：由根 CA 自己对自己签发的
2. intermediates 证书：根 CA 生成一对公钥、私钥，并用私钥将中间 CA 的信息和公钥进行“加密”生成签名，并封装得到 intermediates 证书。上一级 CA 也是按照这个逻辑给下一级 CA 进行签发证书。
3. end-user 证书：最后的 CA 生成公钥私钥，并私钥将用户信息、公钥进行“加密”得到 end-user 证书

证书这一级一级的关系就形成了证书链。

### 7.2中间证书

GoDaddy 是这样描述的：

> ​	中间证书用作我们的根证书的替代品。我们使用中间证书作为代理，因为我们必须将我们的根证书置于多个安全层之后，确保其密钥绝对不可访问。
>
> ​	但是，由于根证书本身签署了中间证书，中间证书可用于签署我们的客户安装和维护“信任链”的 SSL。

### 7.3 证书链的作用

​	可以保证中间 CA 公开的公钥安全，并且如果中间 CA 的私钥泄漏了，那用根 CA 再签发一个就好了，不会影响到这个根 CA 的所有证书。

### 7.4 根证书可信度

​	根证书大部分是内嵌在软件中**（如浏览器、操作系统）**，所以理论上是可信的，当然也可以自己下载安装证书。[Apple 官网](https://links.jianshu.com/go?to=https%3A%2F%2Fsupport.apple.com%2Fen-us%2FHT202858) 可以看到 macOs 所支持的根证书。在 Windows 上可以运行 certmgr.msc，结果如：

![img](https://s2.loli.net/2022/08/07/C7pcznTSOlrtLxH.webp)

### 7.5 证书链校验

证书签发：

- `根证书`CA机构 使用自己的`私钥`对`中间证书`进行签名，授权`中间机构`证书；
- `中间机构`使用自己的`私钥`对`服务器证书`进行签名，从而授权`服务器证书`。

证书校验：

- `客户端`通过`服务器证书` 中`签发机构信息`，获取到`中间证书公钥`；利用`中间证书公钥`进行`服务器证书`的签名验证。
  a、中间证书公钥解密 服务器签名，得到证书摘要信息；
  b、摘要算法计算 服务器证书 摘要信息；
  c、然后对比两个摘要信息。
- `客户端`通过`中间证书`中`签发机构信息`，客户端本地查找到`根证书公钥`；利用`根证书公钥`进行`中间证书`的签名验证。

![证书链](https://s2.loli.net/2022/08/07/UlEQ16PLH8YIBqW.png)