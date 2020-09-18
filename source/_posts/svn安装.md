---
title: svn安装
date: 2020-09-18 09:45:22
tags:
  - 日常运维
categories:
  - 运维
img:
summary: svn服务器安装
---
```
yum install subversion
mkdir -p /home/svn/repo
svnadmin create /home/svn/repo
```
修改`/home/svn/repo`svnserve.conf配置
```
# anon-access = read
  auth-access = write
  password-db = passwd
  authz-db = authz
  realm = /home/svn/repo
```
```
[groups]
# harry_and_sally = harry,sally
# harry_sally_and_joe = harry,sally,&joe
admin = admin
developer = developer

# [/foo/bar]
# harry = rw
# &joe = r
# * =

#[/home/svn/repo]
#@admin = rw

# [repository:/baz/fuz]
# @harry_and_sally = rw
# * = r

[repo:/]
@admin = rw
* =

# repo就是仓库名字，配置仓库目录下的权限
[repo:/requirement]
@admin = rw
@developer = rw
* =
```
svn systemd部署（开机启动）
```bash
tee /etc/systemd/system/svn.service <<-'EOF'
[Unit]
Description=svn服务
After=syslog.target

[Service]
User=root
Type=forking
Restart=on-failure
RestartSec=10

ExecStart=/usr/bin/svnserve --daemon --pid-file=/home/svn/svn.pid -d -r /home/svn
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```
```
systemctl enable svn
systemctl start svn
systemctl status svn
```