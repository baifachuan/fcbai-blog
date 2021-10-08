之前我折腾过很多版博客，之前得用“年”做单位了，很多版也是N种不同语言了，近些年懒得折腾了，目前一直是hexo，偶尔写文章需要手动发布，这一点有些不舒服，不过好在不是很频繁所以也能接受。

不过最近打算弄一个CI/CD还是把这部分自动化了，其实很简单，就是部署一个CI/CD工具，把github的markdown文件拉到服务器上，然后自动hexo g一下。

我的所有服务出口统一是也nginx代理了，关于工具开始想的是jenkins，不过发现始终无法启动，也就是war丢在tomcat下后，打开服务是404，但是一切启动正常，我又不想用yun把它安装成分散到系统的软件包，也不想为此搞一个docker。

所以就直接换成gocd了，gocd在于太过于复杂，我只会用到它的简单功能，不过无所谓了，另外得吐槽一下gocd这么多年的用户管理都是还是如此之不好用，用户创建得自己半手工去维护，我直接使用PHP去做了。使用的认证方式是：GOCD – BASIC USER PASSWORD LOGIN AND API (AUTHENTICATION)


```shell
1. generate password hash
$ php -r 'echo base64_encode(sha1("badger", true));'
ThmbShxAtJepX80c2JY1FzOEmUk=

2. create password file
sudo -u go touch /etc/go/passwd
echo "<username>:<password hash>" > /etc/go/passwd

3. enable password file
Admin -> Server Configuration -> User Management -> Password File Settings -> Password File Path -> /etc/go/passwd -> Save
```
就是上面这几个步骤，然后再在gocd里面配置一下，就ok，先就这样吧。