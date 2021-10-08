---
title: OpenLdap相关知识的记录
tags: 大数据
categories: 大数据
abbrlink: 9c831021
date: 2021-06-08 23:09:21
---

### 什么是OpenLdap

以前只是简单的使用OpenLdap，不求甚解，最近需要深度的以来OpenLdap来做用户中心，进行权限相关的处理，并且需要打通多个系统集成，作为默认服务启动，于是开始仔细的研究OpenLdap的相关知识了。

在用OpenLdap的时候大家一定会看到一些缩写，例如DC、UID、OU、CN、SN等等，这些词是构成Ldap的核心要素。

OpenLdap本质上也是Ldap协议的一种实现，LDAP具有两个标准，分别是X.500和LDAP。OpenLDAP是基于X.500标准的，而且去除了X.500复杂的功能并且可以根据自我需求定制额外扩展功能，但与X.500也有不同之处，例如OpenLDAP支持TCP/IP协议等，目前TCP/IP是Internet上访问互联网的协议，但OpenLDAP目录服务不支持通用数据库的大量更新操作所需要的复杂的事务管理或回滚策略等。

OpenLDAP可以直接运行在更简单和更通用的TCP/IP或其他可靠的传输协议层上，避免了在OSI会话层和表示层的开销，使连接的建立和包的处理更简单、更快，对于互联网和企业网应用更理想。

OpenLDAP目录中的信息是以树状的层次结构来存储数据（这很类同于DNS），最顶层即根部称作“基准DN”，形如“dc=mydomain,dc=org”或者“o=mydomain.org”，前一种方式更为灵活也是Windows AD中使用的方式。在根目录的下面有很多的文件和目录，为了把这些大量的数据从逻辑上分开，OpenLDAP像其它的目录服务协议一样使用OU（Organization Unit，组织单元），可以用来表示公司内部机构，如部门等，也可以用来表示设备、人员等。同时OU还可以有子OU，用来表示更为细致的分类。

OpenLDAP中每一条记录都有一个唯一的区别于其它记录的名字DN（Distinguished Name）,其处在“叶子”位置的部分称作RDN(用户条目的相对标识名)。如dn:cn=tom,ou=animals,dc=ilanni,dc=com中cn即为RDN，而RDN在一个OU中必须是唯一的。

OpenLDAP默认以Berkeley DB作为后端数据库，BerkeleyDB数据库主要以散列的数据类型进行数据存储，如以键值对的方式进行存储。

BerkeleyDB是一类特殊的面向查询进行优化、面向读取进行优化的数据库，主要用于搜索、浏览、更新查询操作，一般对于一次写入数据、多次查询和搜索有很好的效果。BerkeleyDB不支持事务型数据库(MySQL、MariDB、Oracle等)所支持的高并发的吞吐量以及复杂的事务操作。

那么关于那些缩写是什么意思呢？我做了一个表格来描述一下：

![a](b00001.png)


LDAP中的命名模型，也即LDAP中的条目定位方式。在LDAP中每个条目均有自己的DN。DN是该条目在整个树中的唯一名称标识，如同文件系统中，带路径的文件名就是DN。

在LDAP中共有四类10中操作：查询类操作，如搜索、比较；更新类操作，如添加条目、删除条目、修改条目名；认证类操作，如绑定、解绑定；其它操作，如放弃和扩展操作。除了扩展操作，另外9种是LDAP的标准操作；扩展操作是LDAP中为了增加新的功能，提供的一种标准的扩展框架，当前已经成为了LDAP标准的扩展操作，有修改密码和StartTLS扩展，在新的RFC标准和草案中增加一些新的扩展操作，不同LDAP厂商也均定义了自己的扩展操作。

下面是关于OpenLdap的一些操作手册内容：

### 安装openldap

yum -y install openldap openldap-servers openldap-clients  openldap-devel compat-openldap migrationtools  openldap-servers-sql

migrationtools 实现OpenLDAP 用户及用户组的添加，migrationtools 开源工具通过查找/etc/passwd、/etc/shadow、/etc/groups 生成LDIF 文件，并通过ldapadd 命令更新数据库数据，完成用户添加

其中compat-openldap这个包与主从有很大的关系

### 查看版本

slapd -VV

[root@test ~]# slapd -VV

@(#) $OpenLDAP: slapd 2.4.44 (May 16 2018 09:55:53) $

mockbuild@c1bm.rdu2.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd

### 配置openldap

注意:从OpenLDAP2.4.23版本开始所有配置数据都保存在/etc/openldap/slapd.d/中，不再使用slapd.conf作为配置文件。

### 设置openldap的管理员密码：

[root@test slapd.d]# slappasswd -s admin@123

{SSHA}ptUSB1Mt+E4EvtscPolhiwLIICTLg1yN

上述加密后的字段保存下，等会我们在配置文件中会使用到。

修改olcDatabase={2}hdb.ldif文件

cd /etc/openldap/slapd.d/cn=config/

vim olcDatabase\=\{2\}hdb.ldif

### 修改域信息

olcSuffix: dc=jinni,dc=com

olcRootDN: cn=root,dc=jinni,dc=com

olcRootPW: {SSHA}ptUSB1Mt+E4EvtscPolhiwLIICTLg1yN

注意：冒号后面一定加空格，其中cn=root中的root表示OpenLDAP管理员的用户名，而olcRootPW表示OpenLDAP管理员的密码。

LDAP是一种通讯协议，如同HTTP是一种协议一样的！

LDAP连接服务器的连接字串格式为：ldap://servername/DN

其中DN有三个属性，分别是CN,OU,DC 

CN, OU, DC 都是 LDAP 连接服务器的端字符串中的区别名称（DN, distinguished   name）

CN：Common Name 为用户名或服务器名，最长可以到80个字符，可以为中文；

DC：Domain Component   DC类似于dns中的每个元素，例如h3c.com，“.”符号分开的两个单词可以看成两个DC

DN：Distinguished Name   类似于DNS，DN与DNS的区别是：组成DN的每个值都有一个属性类型，例如:H3c.com是一个dns，那么用dn表示为：dc=h3c,dc=com 级别越高越靠后。H3c和com的属性都是DC。

DN可以表示为ldap的某个目录，也可以表示成目录中的某个对象，这个对象可以是用户等。 

### ldapadd 

修改olcDatabase={1}monitor.ldif文件

vim olcDatabase\=\{1\}monitor.ldif



注意：该修改中的dn.base是修改OpenLDAP的管理员的相关信息的。

验证OpenLDAP的基本配置，使用如下命令：

slaptest -u

我们可以看出OpenLDAP的基本配置是否有问题。

### 启动OpenLDAP服务

systemctl enable slapd

systemctl start slapd

### 配置ldap数据库

OpenLDAP默认以Berkeley DB作为后端数据库，BerkeleyDB数据库主要以散列的数据类型进行数据存储，如以键值对的方式进行存储。

BerkeleyDB是一类特殊的面向查询进行优化、面向读取进行优化的数据库，主要用于搜索、浏览、更新查询操作，一般对于一次写入数据、多次查询和搜索有很好的效果。BerkeleyDB不支持事务型数据库(MySQL、MariDB、Oracle等)所支持的高并发的吞吐量以及复杂的事务操作。

现在来开始配置OpenLDAP数据库，使用如下命令：

cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG

chown ldap:ldap -R /var/lib/ldap

chmod 700 -R /var/lib/ldap

注意：/var/lib/ldap/就是BerkeleyDB数据库默认存储的路径。

### 导入基本Schema
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

修改migrate_common.ph文件
migrate_common.ph文件主要是用于生成ldif文件使用，修改migrate_common.ph文件，如下：

vim /usr/share/migrationtools/migrate_common.ph +71

$DEFAULT_MAIL_DOMAIN = “jinni.com”;

$DEFAULT_BASE = “dc=jinni,dc=com”;

$EXTENDED_SCHEMA = 1;        #开启扩展模式

到此OpenLDAP的配置就已经全部完毕，下面我们来开始添加用户到OpenLDAP中。

### 添加用户及用户组
默认情况下OpenLDAP是没有普通用户的，但是有一个管理员用户。管理用户就是前面我们刚刚配置的root。

现在我们把系统中的用户，添加到OpenLDAP中。为了进行区分，我们现在新加两个用户ldapuser1和ldapuser2，和两个用户组ldapgroup1和ldapgroup2，如下：

### 添加用户组，使用如下命令：

groupadd ldapgroup1

groupadd ldapgroup2

### 添加用户并设置密码，使用如下命令：

useradd -g ldapgroup1 ldapuser1

useradd -g ldapgroup2 ldapuser2

echo ‘123456’ | passwd –stdin ldapuser1

echo ‘123456’ | passwd –stdin ldapuser2

把刚刚添加的用户和用户组提取出来，这包括该用户的密码和其他相关属性，如下：

grep “:10[0-9][0-9]” /etc/passwd > /root/users

grep “:10[0-9][0-9]” /etc/group > /root/groups

根据上述生成的用户和用户组属性，使用migrate_passwd.pl文件生成要添加用户和用户组的ldif，如下：

/usr/share/migrationtools/migrate_passwd.pl /root/users > /root/users.ldif

/usr/share/migrationtools/migrate_group.pl /root/groups > /root/groups.ldif

注意：后续如果要新加用户到OpenLDAP中的话，我们可以直接修改users.ldif文件即可。

### 导入用户及用户组到OpenLDAP数据库
配置openldap基础的数据库，如下：

cat > base.ldif << EOF

dn: dc=jinni,dc=com

o: jinni com

dc: jinni

objectClass: top

objectClass: dcObject

objectclass: organization

dn: cn=root,dc=jinni,dc=com

cn: root

objectClass: organizationalRole

description: Directory Manager

dn: ou=People,dc=jinni,dc=com

ou: People

objectClass: top

objectClass: organizationalUnit

dn: ou=Group,dc=jinni,dc=com

ou: Group

objectClass: top

objectClass: organizationalUnit

EOF

 

### 导入基础数据库，使用如下命令：

ldapadd -x -w "admin@123" -D "cn=root,dc=jinni,dc=com" -f /root/base.ldif

 

此密码是之前创建的root管理员密码，如果密码错误，会报错：



文件格式错误，会报错：ldap_add: Protocol error (2)

    additional info: no attributes provided



解决：删除 base.ldif里面空行，同时，dn前留一空行。

导入用户到数据库，使用如下命令：

ldapadd -x -w “admin@123” -D “cn=root,dc=jinni,dc=com” -f /root/users.ldif

 

### 导入用户组到数据库，使用如下命令：

dapadd -x -w “admin@123” -D “cn=root,dc=jinni,dc=com” -f /root/groups.ldif

查看BerkeleyDB数据库文件，使用如下命令：

ll /var/lib/ldap/

此时BerkeleyDB数据库文件中多了cn.bdb、sn.bdb、ou.bdb等数据库文件

查询OpenLDAP的相关信息
用户和用户组全部导入完毕后，我们就可以查询OpenLDAP的相关信息。

### 查询OpenLDAP全部信息，使用如下命令：

ldapsearch -x -b “dc=jinni,dc=com” -H ldap://127.0.0.1

### 查询添加的OpenLDAP用户信息，使用如下命令：

ldapsearch -LLL -x -D ‘cn=root,dc=jinni,dc=com’ -w “admin@123” -b ‘dc=jinni,dc=com’ ‘uid=ldapuser1’

### 查询添加的OpenLDAP用户组信息，使用如下命令：

ldapsearch -LLL -x -D ‘cn=root,dc=jinni,dc=com’ -w “admin@123” -b ‘dc=jinni,dc=com’ ‘cn=ldapgroup1’

### 把OpenLDAP用户加入到用户组
尽管我们已经把用户和用户组信息，导入到OpenLDAP数据库中了。但实际上目前OpenLDAP用户和用户组之间是没有任何关联的。

如果我们要把OpenLDAP数据库中的用户和用户组关联起来的话，我们还需要做另外单独的配置。

现在我们要把ldapuser1用户加入到ldapgroup1用户组，需要新建添加用户到用户组的ldif文件，如下：

cat > add_user_to_groups.ldif <<  EOF

dn: cn=ldapgroup1,ou=Group,dc=jinni,dc=com

changetype: modify

add: memberuid

memberuid: ldapuser1

EOF

执行如下命令：

ldapadd -x -w “admin@123” -D “cn=root,dc=jinni,dc=com” -f /root/add_user_to_groups.ldif

### 查询添加的OpenLDAP用户组信息，如下：

ldapsearch -LLL -x -D ‘cn=root,dc=jinni,dc=com’ -w “admin@123” -b ‘dc=jinni,dc=com’ ‘cn=ldapgroup1’

我们可以很明显的看出ldapuser1用户已经加入到ldapgroup1用户组了。

### 开启OpenLDAP日志访问功能
默认情况下OpenLDAP是没有启用日志记录功能的，但是在实际使用过程中，我们为了定位问题需要使用到OpenLDAP日志。

新建日志配置ldif文件，如下：

cat > /root/loglevel.ldif << “EOF”

dn: cn=config

changetype: modify

replace: olcLogLevel

olcLogLevel: stats

EOF

### 导入到OpenLDAP中，并重启OpenLDAP服务，如下：

ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/loglevel.ldif

systemctl restart slapd

修改rsyslog配置文件，并重启rsyslog服务，如下：

cat >> /etc/rsyslog.conf << “EOF”

local4.* /var/log/slapd.log

EOF

systemctl restart rsyslog

查看OpenLDAP日志，如下：

tail -f /var/log/slapd.log

### 修改OpenLDAP默认监听端口
在前面的文章中，我们已经介绍了OpenLDAP默认监听的端口是389.如果我们现在要修改OpenLDAP监听端口的话，我们可以修改/etc/sysconfig/slapd文件。例如我们现在把OpenLDAP监听的端口修改为4567，可以进行如下操作：

vim /etc/sysconfig/slapd

SLAPD_URLS=”ldapi://0.0.0.0:4567/ldap://0.0.0.0:4567/”

### 重启OpenLDAP服务，如下：

systemctl restart slapd.service

netstat -tunlp

到此完结！

### 部分参数

-x   进行简单认证

-D   用来绑定服务器的DN

-h   目录服务的地址

-w   绑定DN的密码（ldap管理员密码）

-f    使用ldif文件进行条目添加的文件

-Y    指定用于身份验证的SASL机制。如果没有指定，程序将选择服务器知道的最佳机制

-H   指定引用ldap服务器的URI;只允许协议/主机/端口字段;一个用空格或逗号分隔的URI列表。

例子 

ldapadd -x -D "cn=root,dc=starxing,dc=com" -w secret -f /root/test.ldif 

ldapadd -x -D "cn=root,dc=starxing,dc=com" -w secret (这样写就是在命令行添加条目)

ldapsearch 

-x    进行简单认证

-D   用来绑定服务器的DN

-w   绑定DN的密码

-b   指定要查询的根节点

-H   指定要查询的服务器

-LLL  禁用打印无关信息

ldapsearch -x -D "cn=root,dc=starxing,dc=com" -w secret -b "dc=starxing,dc=com" 

使用简单认证，用 "cn=root,dc=starxing,dc=com" 进行绑定，

要查询的根是 "dc=starxing,dc=com"。这样会把绑定的用户能访问"dc=starxing,dc=com"下的

所有数据显示出来。

ldapsearch -x -W -D "cn=administrator,cn=users,dc=osdn,dc=zzti,dc=edu,dc=cn" -b "cn=administrator,cn=users,dc=osdn,dc=zzti,dc=edu,dc=cn" -h troy.osdn.zzti.edu.cn 

ldapsearch -b "dc=canon-is,dc=jp" -H ldaps://192.168.0.92:636 

ldapdelete 

ldapdelete -x -D "cn=Manager,dc=test,dc=com" -w secret "uid=test1,ou=People,dc=test,dc=com" 

ldapdelete -x -D 'cn=root,dc=it,dc=com' -w secert 'uid=zyx,dc=it,dc=com' 

这样就可以删除'uid=zyx,dc=it,dc=com'记录了，应该注意一点，如果o或ou中有成员是不能删除的。

ldappasswd 

-x   进行简单认证

-D   用来绑定服务器的DN

-w   绑定DN的密码

-S   提示的输入密码

-s pass 把密码设置为pass

-a pass 设置old passwd为pass

-A   提示的设置old passwd

-H   是指要绑定的服务器

-I   使用sasl会话方式

#ldappasswd -x -D 'cm=root,dc=it,dc=com' -w secret 'uid=zyx,dc=it,dc=com' -S

New password:

Re-enter new password: 

就可以更改密码了，如果原来记录中没有密码，将会自动生成一个userPassword

创建ldap账号

 

LDAP添加新用户
具体过程如下：

1）首先运行一个shell脚本，脚本内容如下：

#!/bin/sh

#首先创建一个linux帐户
useradd $1 
passwd $1

#转gid到ldap帐户
cat /etc/group | grep $1 >/tmp/group.in
/usr/share/openldap/migration/migrate_group.pl /tmp/group.in > /tmp/group.ldif
ldapadd -x -D "cn=admin,dc=cas,dc=cn" -w your_ldap_passwd -f /tmp/group.ldif

#转uid到ldap帐户
cat /etc/passwd | grep $1 > /tmp/passwd.in
/usr/share/openldap/migration/migrate_passwd.pl /tmp/passwd.in > /tmp/passwd.ldif
ldapadd -x -D "cn=admin,dc=cas,dc=cn" -w your_ldap_passwd -f /tmp/passwd.ldif

#删掉创建的linux帐户, 使帐户成为纯粹的ldap帐户，而不是local帐户
userdel $1
rm -rf /home/$1
#rm /tmp/group.ldif
#rm /tmp/passwd.ldif

ldapsearch -x "uid=$1" #可用于显示刚刚添加到ldap数据库中的用户信息

 

2）用户添加好以后，需要给其设定初始密码，运行命令如下：

# ldappasswd -x -D "cn=admin,dc=cas,dc=cn" -w your_ldap_passwd "uid=XiaoMing,ou=People,dc=cas,dc=cn" -S

 

3）给用户设定登陆后的家目录(Home directory)：这个是可选的：

ldapmodify -x -D "cn=admin,dc=cas,dc=cn" -W 
dn: uid=XiaoMing,ou=People,dc=cas,dc=cn 
changetype: modify
replace: homeDirectory
homeDirectory: /home/xiaoming # 此处添加你为用户指定的家目录。如果目录不存在，不用担心，系统会自动添加的

 

4）如果不小心将ldap管理员密码丢失，只需要在ldap服务器主机上运行命令slappasswd设置新的密码就可以了
