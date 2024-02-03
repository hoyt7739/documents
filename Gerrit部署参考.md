# Gerrit部署参考
*by power*<br/>
*2023/11/14*<br/>
<br/>

### 1. 系统环境
安装ubuntu服务器版（ https://ubuntu.com/download/server ）。<br/>
更新系统：
```shell
sudo apt-get update
```
<br/>

### 2. 安装SSH
```shell
sudo apt-get install ssh
```
OpenSSH8.8之后不再默认支持rsa，需要修改配置文件添加内容：
```shell
sudo vim /etc/ssh/sshd_config
```
```txt
HostKeyAlgorithms=+ssh-rsa,ssh-dss
KexAlgorithms=+diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```
重启SSH服务：
```shell
sudo systemctl restart sshd.service
```
<br/>

### 3. 安装java-sdk
查看当前支持版本：
```shell
sudo apt-get install java-sdk
```
择一版本安装，如：
```shell
sudo apt-get install openjdk-11-jdk
```
<br/>

### 4. 安装ngnix
```shell
sudo apt-get install nginx
```
配置ngnix访问权限，修改第一行：
```shell
sudo vim /etc/nginx/nginx.conf
```
```txt
user root;
```
<br/>

### 5. 创建专用用户
```shell
sudo adduser gerrit
```
配置sudo权限，添加一行：
```shell
sudo vim /etc/sudoers
```
```txt
gerrit  ALL=(ALL:ALL) ALL
```
切到该用户：
```shell
sudo su gerrit
```
<br/>

### 6. 安装Gerrit
下载Gerrit最新版（ https://www.gerritcodereview.com/#download ），然后安装：
```shell
cd ~
mkdir review_site
java -jar gerrit-*.war init -d review_site/
```
设置下列选项，插件全y，其余按需设置或默认：
```txt
*** User Authentication
Authentication method [openid/?]: HTTP

*** Email Delivery
SMTP server hostname [localhost]: <专用邮箱SMTP服务器>
SMTP server port [(default)]: <SMTP端口>
SMTP encryption [none/?]: <[SSL]>
SMTP username : <邮箱地址，需要授权码>

*** HTTP Daemon
Behind reverse proxy [y/N]? y
Listen on port [8081]: <Gerrit服务端口>
Canonical URL [http://machine/]: <Gerrit地址>
```
配置nginx反向代理：
```shell
sudo vim /etc/nginx/conf.d/gerrit.conf
```
```txt
server {
    listen 81;
    server_name localhost;
    allow all;
    deny all;

    auth_basic "Welcome to Gerrit!";
    auth_basic_user_file /home/gerrit/review_site/etc/gerrit.password;

    location / {
        proxy_pass http://<host>:8081;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
    }
}
```
配置Gerrit，修改对应字段：
```shell
sudo vim ~/review_site/etc/gerrit.config
```
```txt
[gerrit]
        canonicalWebUrl = http://<host>:81/
[container]
        user = gerrit
[auth]
        type = HTTP
[sendemail]
        enable = true
        smtpServer = <专用邮箱SMTP服务器>
        smtpServerPort = <SMTP端口>
        smtpEncryption = <[SSL]>
        sslVerify = <是否SSL>
        smtpUser = <邮箱地址>
        from = <邮箱地址>
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = proxy-http://*:8081/
```
配置邮箱授权码，添加字段：
```shell
sudo vim ~/review_site/etc/secure.config
```
```txt
[sendemail]
        smtpPass = <code>
```
启动Gerrit：
```shell
sudo bash ~/review_site/bin/gerrit.sh start
```
重启nginx并检查工作状态：
```shell
sudo systemctl restart nginx.service
sudo netstat -ltpn
```
<br/>

### 7. 创建用户
创建第一个用户（管理员）：
```shell
cd ~/review_site/etc
htpasswd -c gerrit.password <admin>
```
登录Gerrit（端口使用nginx监听的81），然后添加邮箱和SSH公钥：
```txt
http://<host>:81/
```
创建普通用户并添加邮箱（需要登录一次才能添加邮箱）：
```shell
htpasswd -m gerrit.password <username>
ssh -p 29418 <admin>@<host> gerrit set-account --add-email <email> <username>
```
<br/>

### 8. 创建群组
管理员 -> Browse -> Group -> CREATE NEW<br/>
将Owners设置为Administrators，以免普通用户能修改成员列表。<br/>
切到Members添加成员。<br/>
<br/>

### 9. 权限设置
按需调整All-Projects权限，典型的配置：
```txt
[project]
	description = Access inherited by all other projects.
[receive]
	requireContributorAgreement = false
	requireSignedOffBy = false
	requireChangeId = true
	enableSignedPush = false
[submit]
	mergeContent = true
[capability]
	administrateServer = group Administrators
	priority = batch group Service Users
	streamEvents = group Service Users
[access "refs/*"]
	read = group Administrators
	read = group Project Owners
[access "refs/heads/*"]
	create = group Administrators
	create = group Project Owners
	editTopicName = +force group Administrators
	editTopicName = +force group Project Owners
	forgeCommitter = group Administrators
	forgeCommitter = group Project Owners
	label-Code-Review = -2..+2 group Administrators
	label-Code-Review = -2..+2 group Project Owners
	submit = group Administrators
	submit = group Project Owners
	revert = group Administrators
	revert = group Project Owners
	forgeAuthor = group Administrators
	forgeAuthor = group Project Owners
	rebase = group Administrators
	rebase = group Project Owners
[access "refs/meta/config"]
	create = group Administrators
	label-Code-Review = -2..+2 group Administrators
	push = group Administrators
	submit = group Administrators
	label-Verified = -1..+1 group Administrators
[access "refs/tags/*"]
	create = group Administrators
	create = group Project Owners
	createSignedTag = group Administrators
	createSignedTag = group Project Owners
	createTag = group Administrators
	createTag = group Project Owners
[access "refs/for/refs/*"]
	pushMerge = group Administrators
	pushMerge = group Project Owners
	push = group Administrators
	push = group Project Owners
[label "Code-Review"]
	function = MaxWithBlock
	defaultValue = 0
	value = -2 This shall not be submitted
	value = -1 I would prefer this is not submitted as is
	value = 0 No score
	value = +1 Looks good to me, but someone else must approve
	value = +2 Looks good to me, approved
	copyCondition = changekind:NO_CHANGE OR changekind:TRIVIAL_REBASE OR is:MIN
```
创建项目后，典型的项目权限配置：
```txt
[access]
	inheritFrom = All-Projects
[submit]
	action = inherit
[access "^refs/for/refs/heads/(?:dev|fix)_[\\w\\.]*"]
	push = group Project_Developers
	pushMerge = group Project_Developers
[access "^refs/heads/(?:dev|fix)_[\\w\\.]*"]
	label-Code-Review = -2..+2 group Project_Developers
	rebase = group Project_Developers
	revert = group Project_Developers
	submit = group Project_Developers
	forgeAuthor = group Project_Developers
[access "refs/heads/*"]
	read = group Project_Developers
	read = group Project_Readers
[access "refs/*"]
	owner = group Project_Managers
```