---
title: 局域网内构建统一可访问的YUM源
pubDate: 2024-10-08
categories: ['Linux']
description: ''
---

## 1.构建内部YUM源的必要性

YUM光盘源默认只能本机使用，局域网其他服务器无法使用YUM光盘源，所以需要通过HTTP构建整个局域网都可以访问的内部YUM源。内部YUM源可以带来以下好处：

### 1.1 提高软件包安装和更新速度
通过搭建内部YUM源，可以将常用的软件包缓存到本地，减少从外部源下载的时间。

### 1.2 节省带宽
内部YUM源可以减少对外部源的访问，从而节省网络带宽。

### 1.3 安全可靠
内部YUM源可以避免使用不受信任的外部源，降低安全风险。

### 1.4 提供离线更新支持
对于无法连接互联网的服务器，内部YUM源可以提供离线更新支持。这在企业内网需求中尤为重要，因为不是所有服务器都能连接互联网。

### 1.5 解决软件依赖关系问题
通过搭建内部YUM源，可以自动处理软件包之间的依赖关系，确保在安装或更新软件包时，所有必需的依赖项都能被正确处理。

## 2.搭建内部YUM源步骤

### 2.1 准备实验环境
- **角色**：服务器端
- **操作系统**：Rocky Linux release 9.1
- **IP地址**：10.10.10.200

- **角色**：客户端
- **操作系统**：Rocky Linux release 9.1
- **IP地址**：10.10.10.201

### 2.2 基于光盘构建本地YUM源
无网环境需要做本地YUM源，首先需要在虚拟机上挂载ISO镜像。

1. **挂载光盘**
    ```sh
    [root@localhost ~]# mount /dev/cdrom /mnt
    mount: /mnt: WARNING: source write-protected, mounted read-only.
    ```

2. **备份原有repo文件**
    ```sh
    [root@localhost ~]# mkdir /etc/yum.repos.d/backup
    [root@localhost ~]# mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup
    ```

3. **创建新repo文件**
    ```sh
    [root@localhost ~]# cat >> /etc/yum.repos.d/local.repo << EOF
    [Base]
    name=Base
    baseurl=file:///mnt/BaseOS
    enabled=1
    gpgcheck=0

    [AppStream]
    name=AppStream
    baseurl=file:///mnt/AppStream
    enabled=1
    gpgcheck=0
    EOF
    ```

4. **安装软件测试**
    ```sh
    [root@localhost ~]# yum install -y telnet
    ```

如果顺利安装软件包，就说明基于光盘做的YUM源已经做好了。

### 2.3 安装HTTP服务器
在YUM服务器上创建一个简单的HTTP服务，可以使用Apache或Nginx，这里使用Apache。

```sh
[root@localhost ~]# yum install httpd -y
```
### 2.4 创建repodata目录# 放置整个rockyLinux镜像的软件包

```sh
[root@localhost ~]# mkdir /var/www/html/rockylinux
```



### 2.5 将需要发布软件包复制到repodata目录1、将光盘挂载后的文件拷贝到repodata目录下

```sh
[root@localhost ~]# cp -r /mnt/* /var/www/html/rockylinux
```

整个镜像文件拷贝需要时间较长一点



### 2.6 安装createrepo包

```sh
[root@localhost ~]# yum install -y createrepo
```

### 2.7 运行createrepo来创建仓库元数据

```sh
[root@localhost ~]# createrepo /var/www/html/rockylinux
Directory walk started
Directory walk done - 6615 packages
Temporary output repo path: /var/www/html/rockylinux/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```


做成repo文件

```sh
[root@localhost ~]# mkdir /var/www/html/repos/rockylinx
[root@localhost ~]# cat >> /var/www/html/repos/rockylinx/rockylinux.repo << EOF
[Base]
name=Base
baseurl=http://10.10.10.200/rockylinux/BaseOS
enabled=1
gpgcheck=0
[AppStream]
name=AppStream
baseurl=http://10.10.10.200/rockylinux/AppStream
enabled=1
gpgcheck=0
EOF
```

### 2.8 启动HTTP服务

```sh
# 启动HTTP并设置开机自启动

[root@localhost ~]#  systemctl enable --now httpd

# 查看httpd状态

[root@localhost ~]# systemctl status httpd
```

## 3.客户端使用yum源

### 3.1  备份原有的repo

```sh
[root@localhost ~]# mkdir /etc/yum.repos.d/backup
[root@localhost ~]# mv  /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup
```

### 3.2 获取yum源的两种方法

 方法一：直接wget已经在服务器端做好的repo文件

```sh
[root@localhost ~]# cd /etc/yum.repos.d/
[root@localhost yum.repos.d]# wget
http://10.10.10.200/repos/rockylinx/rockylinux.repo
```


方法二：在客户端创建新的repo文件

```sh
[root@localhost ~]# cat >> /etc/yum.repos.d/rockylinux.repo << EOF
[Base]
name=Base
baseurl=http://10.10.10.200/rockylinux/BaseOS
enabled=1
gpgcheck=0
[AppStream]
name=AppStream
baseurl=http://10.10.10.200/rockylinux/AppStream
enabled=1
gpgcheck=0
EOF
```

### 3.3 测试yum源

```sh
# 先清一下原有yum源数据
[root@localhost ~]# yum clean all
# 安装telnet测试一下
[root@localhost ~]# yum install -y telnet
```


成功安装就代表内部yum源已经做成功了。

**局域网内其他服务器也可以通过wget直接获取或配置repo文件来构建可用的yum源。**

### 3.4  httpd作为共享服务器使用

可以在httpd的发布目录下创建一个software目录，将一些常用的软件包放置到里面，局域网内的客户端可以直接通过wget来直接获取软件包

服务器端创建发布目录并将软件包上传

```sh
# 创建software目录

[root@localhost ~]# mkdir /var/www/html/software

# 此处上传Tomcat包到software为例

[root@localhost ~]# cp /root/apache-tomcat-8.5.97.tar.gz /var/www/html/software
```

客户端获取软件包

```sh
[root@localhost ~]# wget http://10.10.10.200/software/apache-tomcat-8.5.97.tar.gz
```

【温馨提示】：本次操作的服务器端是RockyLinux操作系统，不只是可以做rockylinux操作系统的YUM源，也可以在服务器端配置多种操作系统的yum源，方法相同，如CentOS，openEuler，麒麟V10等。
