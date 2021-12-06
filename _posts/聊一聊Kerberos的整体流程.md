---
title: 聊一聊Kerberos的整体流程
tags: 编程基础
categories: 编程基础
abbrlink: 8a2f3607
date: 2021-11-30 16:50:49
---

不得不说，在认证方面Kerberos可谓是一头巨复杂的巨兽，很多企业想要驾驭好它绝非易事，首先 Kerberos是种网络身份验证协议,最初设计是用来保护雅典娜工程的网络服务器。

历史也有意识，Kerberos这个名字源于希腊神话，是一只三头犬的名字，它旨在通过使用密钥加密技术为Client/Server序提供强身份验证。
可以用于防止窃听、防止重放攻击、保护数据完整性等场合，是一种应用对称密钥体制进行密钥管理的系统。Kerberos的扩展产品也使用公开密钥加密方法进行认证。

Kerberos目前最新版本是5，1~3版本只在MIT内部发行，因为使用DES加密，早期被美国出口管制局列为军需品禁止出口，直到瑞典皇家工学院实现了Kerberos版本4，KTH-KRB。
后续也是这个团队实现了版本5: Heimdal，目前常见的Kerberos5实现之一。

在Kerberos中有一些名词，先理解一下它的意思：
* AS（Authentication Server）：认证服务器
* KDC（Key Distribution Center）：密钥分发中心
* TGT（Ticket Granting Ticket）：票据授权票据，票据的票据
* TGS（Ticket Granting Server）：票据授权服务器
* SS（Service Server）：特定服务提供端
* Principal：被认证的个体
* Ticket：票据，客户端用来证明身份真实性。包含：用户名，IP，时间戳，有效期，会话秘钥。

使用Kerberos时，一个客户端需要经过三个步骤来获取服务:
* 认证: 客户端向认证服务器发送一条报文，获取一个包含时间戳的TGT。此流程使用了对称加密。
* 授权: 客户端使用TGT向TGS请求指定Service的Ticket。此流程发生在某一个Kerberos领域中。
* 服务请求: 客户端向指定的Service出示服务Ticket鉴权通讯。小写字母c,d,e,g是客户端发出的消息，大写字母A,B,E,F,H是各个服务器发回的消息。

Kerberos协议在网络通信协定中属于显示层。其通信流程简单地说，用户先用共享密钥从某认证服务器得到一个身份证明。随后，用户使用这个身份证明与SS通信，而不使用共享密钥。

一般大家在使用Kerberos的时候，特别是在Java工程，最多就是拿着一个Keytab，然后使用：
```java
UserGroupInformation.setConfiguration(conf);
UserGroupInformation.loginUserFromKeytab(name, KeyTabPath);
```
就进行认证了，实际上背后发生的流程是非常复杂的，具体通信流程：

![image](kerberos001.png)

首先，用户使用客户端上的程序进行登录：
1. 输入用户ID和密码到客户端（或使用keytab登录）。
2. 客户端程序运行一个单向函数（大多数为Hash）把密码转换成密钥，这个就是客户端的“用户密钥”(user's secret key)。

然后客户端(Client)从认证服务器(AS)获取票据的票据（TGT），也就是进行客户端认证：

1. Client向AS发送1条明文消息，申请基于该用户所应享有的服务，例如“用户Sunny想请求服务”（Sunny是用户ID）。（注意：用户不向AS发送“用户密钥”(user's secret key)，也不发送密码）该AS能够从本地数据库中查询到该申请用户的密码，并通过相同途径转换成相同的“用户密钥”(user's secret key)。
2. AS检查该用户ID是否在于本地数据库中，如果用户存在则返回2条消息：
   * 【消息A】：Client/TGS会话密钥(Client/TGS Session Key)（该Session Key用在将来Client与TGS的通信（会话）上），通过 用户密钥(user's secret key) 进行加密
   * 【消息B】：票据授权票据(TGT)（TGT包括：消息A中的“Client/TGS会话密钥”(Client/TGS Session Key)，用户ID，用户网址，TGT有效期），通过TGS密钥(TGS's secret key) 进行加密
3. 一旦Client收到消息A和消息B，Client首先尝试用自己的“用户密钥”(user's secret key)解密消息A，如果用户输入的密码与AS数据库中的密码不符，则不能成功解密消息A。输入正确的密码并通过随之生成的"user's secret key"才能解密消息A，从而得到“Client/TGS会话密钥”(Client/TGS Session Key)。（注意：Client不能解密消息B，因为B是用TGS密钥(TGS's secret key)加密的）。拥有了“Client/TGS会话密钥”(Client/TGS Session Key)，Client就足以通过TGS进行认证了。

客户端认证结束后，进入服务端授权：
![image](kerberos002.png)

1. 当client需要申请特定服务时，其向TGS发送以下2条消息：
   * 【消息c】：即消息B的内容（TGS's secret key加密后的TGT），和想获取的服务的服务ID（注意：不是用户ID）
   * 【消息d】：认证符(Authenticator)（Authenticator包括：用户ID，时间戳），通过Client/TGS会话密钥(Client/TGS Session Key)进行加密
2. 收到消息c和消息d后，TGS首先检查KDC数据库中是否存在所需的服务，查找到之后，TGS用自己的“TGS密钥”(TGS's secret key)解密消息c中的消息B（也就是TGT），从而得到之前生成的“Client/TGS会话密钥”(Client/TGS Session Key)。TGS再用这个Session Key解密消息d得到包含用户ID和时间戳的Authenticator，并对TGT和Authenticator进行验证，验证通过之后返回2条消息：
   *【消息E】：client-server票据(client-to-server ticket)（该ticket包括：Client/SS会话密钥 (Client/Server Session Key），用户ID，用户网址，有效期），通过提供该服务的服务器密钥(service's secret key) 进行加密
   *【消息F】：Client/SS会话密钥( Client/Server Session Key) （该Session Key用在将来Client与Server Service的通信（会话）上），通过Client/TGS会话密钥(Client/TGS Session Key) 进行加密
3. Client收到这些消息后，用“Client/TGS会话密钥”(Client/TGS Session Key)解密消息F，得到“Client/SS会话密钥”(Client/Server Session Key)。（注意：Client不能解密消息E，因为E是用“服务器密钥”(service's secret key)加密的）。

最后是服务请求：

Client从SS获取服务。

1. 当获得“Client/SS会话密钥”(Client/Server Session Key)之后，Client就能够使用服务器提供的服务了。Client向指定服务器SS发出2条消息：
   * 【消息e】：即上一步中的消息E“client-server票据”(client-to-server ticket)，通过服务器密钥(service's secret key) 进行加密
   * 【消息g】：新的Authenticator（包括：用户ID，时间戳），通过Client/SS会话密钥(Client/Server Session Key) 进行加密
2. SS用自己的密钥(service's secret key)解密消息e从而得到TGS提供的Client/SS会话密钥(Client/Server Session Key)。再用这个会话密钥解密消息g得到Authenticator，（同TGS一样）对Ticket和Authenticator进行验证，验证通过则返回1条消息（确认函：确证身份真实，乐于提供服务）
   * 【消息H】：新时间戳（新时间戳是：Client发送的时间戳加1，v5已经取消这一做法），通过Client/SS会话密钥(Client/Server Session Key) 进行加密
3. Client通过Client/SS会话密钥(Client/Server Session Key)解密消息H，得到新时间戳并验证其是否正确。验证通过的话则客户端可以信赖服务器，并向服务器（SS）发送服务请求。
4. 服务器（SS）向客户端提供相应的服务。

于是整体便完成了Kerberos的整个流程。
