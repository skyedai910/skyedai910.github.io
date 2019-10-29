---
title: rclone all in one|伪最全指南
visitors: 阅读量
copyright: true
tags:
  - rclone
  - onedrive
  - googledrive
  - 网盘
abbrlink: 6bde393d
date: 2019-09-26 13:29:20
updated: 2019-09-26 13:29:20
categories:
password:
top:
---

## 介绍

Rclone 是一款的命令行工具，支持在不同对象存储、网盘间同步、上传、下载数据。

- 官网网址：[https://rclone.org](https://www.moewah.com/go/aHR0cHM6Ly9yY2xvbmUub3Jn)
- Github 项目：[https://github.com/ncw/rclone](https://www.moewah.com/go/aHR0cHM6Ly9naXRodWIuY29tL25jdy9yY2xvbmU=)

支持的主流对象存储有：

```
Google Drive 
Amazon S3 #消息称Amazon单方面禁止了 rclone 在他家存储上使用。
Openstack Swift / Rackspace cloud files / Memset Memstore
Dropbox
Google Cloud Storage
Amazon Drive
Microsoft One Drive
Hubic
Backblaze B2
Yandex Disk
The local filesystem
```

Rclone 更完整的云存储支持列表 -> [查看完整列表](https://www.moewah.com/go/aHR0cHM6Ly9yY2xvbmUub3JnL292ZXJ2aWV3Lw==)



## 安装

本教程在Debian\ Ubuntu下，部署成功。其他系统安装方法，请参考：https://rclone.org/install/。

除安装部分以外，挂载、授权等各系统大同小异。



### 1、安装rclone

> 安装含环境以及常用软件



Debian 8 & Ubuntu

```shell
apt-get update
apt-get install screen screen unzip fuse
curl -O https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip rclone-current-linux-amd64.zip
cd rclone-*-linux-amd64
sudo cp rclone /usr/bin/
sudo chown root:root /usr/bin/rclone
sudo chmod 755 /usr/bin/rclone
```



Centos 7

```shell
yum update
yum install epel-release
yum install curl unzip screen fuse fuse-devel
curl -O https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip rclone-current-linux-amd64.zip
cd rclone-*-linux-amd64
sudo cp rclone /usr/bin/
sudo chown root:root /usr/bin/rclone
sudo chmod 755 /usr/bin/rclone
```



### 2、初始化配置

#### 挂载Google Drive

```shell
rclone config
```

然后，出现交互式的配置界面：(这里演示配置google drive)

```shell
e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q> n #选着n新建一个配置
name> gd #配置的名称，以后会用到要牢记
Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / 1Fichier
   \ "fichier"
 2 / Alias for an existing remote
   \ "alias"
 3 / Amazon Drive
   \ "amazon cloud drive"
 4 / Amazon S3 Compliant Storage Provider (AWS, Alibaba, Ceph, Digital Ocean, Dreamhost, IBM COS, Minio, etc)
   \ "s3"
 5 / Backblaze B2
   \ "b2"
 6 / Box
   \ "box"
 7 / Cache a remote
   \ "cache"
 8 / Dropbox
   \ "dropbox"
 9 / Encrypt/Decrypt a remote
   \ "crypt"
10 / FTP Connection
   \ "ftp"
11 / Google Cloud Storage (this is not Google Drive)
   \ "google cloud storage"
12 / Google Drive
   \ "drive"
13 / Google Photos
   \ "google photos"
14 / Hubic
   \ "hubic"
15 / JottaCloud
   \ "jottacloud"
16 / Koofr
   \ "koofr"
17 / Local Disk
   \ "local"
18 / Mega
   \ "mega"
19 / Microsoft Azure Blob Storage
   \ "azureblob"
20 / Microsoft OneDrive
   \ "onedrive"
21 / OpenDrive
   \ "opendrive"
22 / Openstack Swift (Rackspace Cloud Files, Memset Memstore, OVH)
   \ "swift"
23 / Pcloud
   \ "pcloud"
24 / Put.io
   \ "putio"
25 / QingCloud Object Storage
   \ "qingstor"
26 / SSH/SFTP Connection
   \ "sftp"
27 / Union merges the contents of several remotes
   \ "union"
28 / Webdav
   \ "webdav"
29 / Yandex Disk
   \ "yandex"
30 / http Connection
   \ "http"
31 / premiumize.me
   \ "premiumizeme"
Storage> 12 #选择12配置google drive
** See help for drive backend at: https://rclone.org/drive/ **
Google Application Client Id
Setting your own is recommended.
See https://rclone.org/drive/#making-your-own-client-id for how to create your own.
If you leave this blank, it will use an internal key which is low performance.
Enter a string value. Press Enter for the default ("").
client_id>  #留空，直接回车
Google Application Client Secret
Setting your own is recommended.
Enter a string value. Press Enter for the default ("").
client_secret> #留空，直接回车
Scope that rclone should use when requesting access from drive.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / Full access all files, excluding Application Data Folder.
   \ "drive"
 2 / Read-only access to file metadata and file contents.
   \ "drive.readonly"
   / Access to files created by rclone only.
 3 | These are visible in the drive website.
   | File authorization is revoked when the user deauthorizes the app.
   \ "drive.file"
   / Allows read and write access to the Application Data folder.
 4 | This is not visible in the drive website.
   \ "drive.appfolder"
   / Allows read-only access to file metadata but
 5 | does not allow any access to read or download file content.
   \ "drive.metadata.readonly"
scope> 1 #选择1 基于全权限
ID of the root folder
Leave blank normally.
Fill in to access "Computers" folders. (see docs).
Enter a string value. Press Enter for the default ("").
root_folder_id> #留空，直接回车
Service Account Credentials JSON file path 
Leave blank normally.
Needed only if you want use SA instead of interactive login.
Enter a string value. Press Enter for the default ("").
service_account_file> 
Edit advanced config? (y/n)
y) Yes
n) No
y/n> n #选择n
Remote config
Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine
y) Yes
n) No
y/n> n #选择n
If your browser doesn't open automatically go to the following link: https://accounts.google.com/o/oauth2/auth?access_type=offline&client_id=202264815644.apps.googleusercontent.com&redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob&response_type=code&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fdrive&state=a5ec519cfa857748b3634e73e0b9026e
Log in and authorize rclone for access
Enter verification code> #复制上面链接在浏览器打开，并登陆你需要挂载的goole drive，授予权限之后，复制token输入到下面。
--------------------
[gd]
client_id = xxxxx
client_secret = xxxxx
token = xxxxx
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y  #选择y
Current remotes:

Name                 Type
====                 ====
Rats                 onedrive

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q> q  #选择q退出
```



#### 挂载onedrive



在选择挂载云盘时选择onedrive，之后参数配置与google drive一样。区别点在于token获取。

```shell
For this to work, you will need rclone available on a machine that has a web browser available.
Execute the following on your machine:
    rclone authorize "onedrive"
Then paste the result below:
result> {"access_token":""}  #输入在客户端授权的内容
```

到这里需要我们在可视化的操作系统下（如win），安装rclone获取token。那么我们就到[官网](https://rclone.org/downloads/)先下载对应可视化系统的rclone。

![img](https://fi.madjack.info/rclonewin.jpg)

在rclone文件下打开cmd（win下shift+右键），然后输入``rclone authorize "onedrive"``

登录帐号后，cmd会显示token，请将``{}``以内的全部字符复制到挂载机上授权。

其余配置与google drive相同



## 挂载

### 1、挂载为磁盘

```shell
#新建本地文件夹，路径自己定，即下面的/root/gd
mkdir /root/gd
#通过screen背景执行rclone挂载gd到本地/root/gd目录
/usr/bin/screen -d -m -S rcmount /usr/bin/rclone mount gd: /root/gd --allow-other --allow-non-empty --vfs-cache-mode writes
```

若成功挂载，输入``df``，可见目录中有gd。



**卸载硬盘**

如果没有进行开机自动挂载，重启就会自动卸载。

```shell
fusermount -qzu /root/gd   #输入之前挂载的本地文件夹路径，即/root/gd
```



### 2、设置自启

> 这里介绍rat大佬博客上的方法，另外一种手动挂载请看[这里](<https://wp.madjack.info/linux/rclone-googledrive-onedrive.html>)

#### 1、下载并编辑脚本

```shell
wget https://www.moerats.com/usr/shell/rcloned && nano rcloned
```

修改一下内容：

```shell
NAME=""  #rclone name名，及配置时输入的Name
REMOTE=''  #远程文件夹，OneDrive网盘里的挂载的一个文件夹,留空挂载根目录
LOCAL=''  #挂载地址，VPS本地挂载目录
```

#### 2、设置自启

使用命令：

```shell
#Debian系统
apt-get install sudo -y

#设置自启
mv rcloned /etc/init.d/rcloned
chmod +x /etc/init.d/rcloned
update-rc.d -f rcloned defaults
bash /etc/init.d/rcloned start
```

检测信息显示`rclone`启动成功即可。

![请输入图片描述](https://www.moerats.com/usr/picture/rclone_GD(2).png)



## 基本操作指令



### 文件复制

```shell
rclone copy -v --stats 10s gd:/iNT/ gd:/backup
## 以上範例為把之前掛載的gd團隊盤資料夾裡面iNT資料夾 複製到gd团队盘的backup下
參數 -v 會顯示當下複制內容 --stats 10s 為每10秒顯示完整進度 ##
```



### 列表

```shell
rclone ls gd:backup
rclone lsl gd:backup # 比上面多一个显示上传时间
rclone lsd gd:backup # 只显示文件夹
```



### 新建文件夹

```shell
rclone mkdir gd:backup
```



### 其他

```shell
rclone config - 以控制会话的形式添加rclone的配置，配置保存在.rclone.conf文件中。
rclone copy - 将文件从源复制到目的地址，跳过已复制完成的。
rclone sync - 将源数据同步到目的地址，只更新目的地址的数据。   –dry-run标志来检查要复制、删除的数据
rclone move - 将源数据移动到目的地址。
rclone delete - 删除指定路径下的文件内容。
rclone purge - 清空指定路径下所有文件数据。
rclone mkdir - 创建一个新目录。
rclone rmdir - 删除空目录。
rclone check - 检查源和目的地址数据是否匹配。
rclone ls - 列出指定路径下所有的文件以及文件大小和路径。
rclone lsd - 列出指定路径下所有的目录/容器/桶。
rclone lsl - 列出指定路径下所有文件以及修改时间、文件大小和路径。
rclone md5sum - 为指定路径下的所有文件产生一个md5sum文件。
rclone sha1sum - 为指定路径下的所有文件产生一个sha1sum文件。
rclone size - 获取指定路径下，文件内容的总大小。.
rclone version - 查看当前版本。
rclone cleanup - 清空remote。
rclone dedupe - 交互式查找重复文件，进行删除/重命名操作。
```



## 常见问题



### Q：上传到Onedrive时，不稳定进程会被杀掉

我在512M机子上跑的时候确实出现过这种情况。可能是内存不够的原因。这里有两个解决办法：1️⃣增加虚拟内存，教程看[这里](<https://blog.csdn.net/znzhizs/article/details/79364228>)。2️⃣限制rclone的上传速度以及缓存区的大小等等配置，教程看[这里](<https://www.moerats.com/archives/877/>)



## 碎碎念

> 在 Linux OS環境下
> 雖然也可以使用一些像cp的本機指令，但還是建議學會透過rclone指令來操作
> 有不少參數設定可以讓掛載的Cloud Service更具體化
> 例如使用cp複制時，只會覆蓋原本已有的資料
> 使用rclone copy時，就會檢查本機 or 遠端已存在什麼資料
> 只會複制現階段本機 or 遠端缺少的資料
>
> 一般rcmount並不佔用本機硬碟的使用空間，除非sync同步或是把遠端cloud data copy到本機
> 但也不建議在使用pt時，直接輔種掛載的團隊硬盤，這樣非常容易出錯，而且並不會節省到任何本機硬碟空間
> 因為系統還是會下載遠端資料夾到資料到本機來補種，這點需特別注意
>
> ——引用自[消失的亞特蘭提斯](https://wp.madjack.info/)



## 参考文章

* [在Debian/Ubuntu上使用rclone挂载OneDrive网盘](https://www.moerats.com/archives/491/)
* [消失的亞特蘭提斯](https://wp.madjack.info/)
* [Rclone 使用教程](moewah.com/archives/876.html)

