---
title: spring boot使用systemd部署
date: 2020-09-18 09:48:34
tags:
  - springboot
  - java
categories:
  - 运维
img: /gallery/thumbnails/spring-boot.png
summary: spring boot使用systemd部署
---
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>${spring-boot.version}</version>
    <configuration>
        <executable>true</executable>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
```
# -f --force
ln -sf $JAVA_HOME/bin/java /sbin/java
```

```bash
tee /etc/systemd/system/xxl-job-admin.service <<-'EOF'
[Unit]
Description=定时任务服务
After=syslog.target

[Service]
User=root
Restart=on-failure
RestartSec=10
startLimitIntervalSec=60

Environment=MYDIR=/home/modularization/xxlJobAdmin

WorkingDirectory=/home/modularization/xxlJobAdmin
ExecStart=/sbin/java -jar -Dspring.profiles.active=test /home/modularization/xxlJobAdmin/xxl-job-admin-2.2.1-SNAPSHOT.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```
```bash
tee /etc/systemd/system/messagemanager.service <<-'EOF'
[Unit]
Description=消息管理程序
After=xxl-job-admin.service
Requires=xxl-job-admin.service

[Service]
User=root
Restart=on-failure
RestartSec=20
startLimitIntervalSec=60

WorkingDirectory=/home/modularization/messageManager
ExecStart=/sbin/java -jar -Dspring.profiles.active=test /home/modularization/messageManager/bdlbsc-message-manager-0.0.1-SNAPSHOT.jar
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```
```bash
# realod service
systemctl daemon-reload
```
```bash
systemctl enable xxl-job-admin
systemctl enable messagemanager
```
```bash
journalctl -f -u messagemanager.service
```
有依赖关系会直接启动xxl-job-admin
# 阿里云流水线主机部署脚本
## 编译
```
tee deploy.sh <<-'EOF'
systemctl start messagemanager
EOF
mvn clean package -DskipTests
```
## 上传路径
```
target/bdlbsc-message-manager-0.0.1-SNAPSHOT.jar
```

## 部署
### 下载路径
```
/home/modularization/messageManager/message-manager.tgz
```
### 部署脚本
```
set -e
mkdir -p /home/modularization/messageManager;
systemctl stop messagemanager
tar xf /home/modularization/messageManager/message-manager.tgz -o -C /home/modularization/messageManager;
chmod +x  /home/modularization/messageManager/deploy.sh;
/home/modularization/messageManager/deploy.sh;
```