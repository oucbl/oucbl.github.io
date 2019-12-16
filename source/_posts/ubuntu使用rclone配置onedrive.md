---
title: Ubuntu安装配置rclone（Onedrive应用）
categories: 学习笔记
tags:
  - rclone
  - Onedrive
abbrlink: '374e8550'
date: 2019-04-22 23:19:11
---



# rclone安装

## 命令行安装

### 脚本安装

```sh
curl https://rclone.org/install.sh | sudo bash
# 或者
curl https://rclone.org/install.sh | sudo bash -s beta  # 测试版
```

###  源文件安装

从预编译二进制文件安装：curl命令可换为wget、axel，当然也可以直接（浏览器）手动下载

```sh
# 1. 获取源文件并解压缩，然后切换到解压目录
curl -O https://downloads.rclone.org/rclone-current-linux-amd64.zip
unzip rclone-current-linux-amd64.zip
cd rclone-*-linux-amd64

# 2. 复制二进制文件并修改用户（组）权限
sudo cp rclone /usr/bin/
sudo chown root:root /usr/bin/rclone
sudo chmod 755 /usr/bin/rclone
```

## deb包安装

下载对应deb包：https://rclone.org/downloads/

### dpkg命令

```shell
sudo dpkg -i xxx.deb
sudo apt-get install -f
```

### gdeb命令

```shell
sudo apt install gdebi-core   # 安装第三方工具
sudo gdebi xxx.deb
```

# rclone配置

## 获取客户端ID和密钥

在默认情况下执行请求时，rclone使用所有rclone用户共享的一对客户端ID和密钥。

可以按照以下步骤获取自己的**客户ID和密钥**：

1. 打开https://apps.dev.microsoft.com/#/appList ，然后单击添加应用（Add an app） 。
2. 输入应用的名称，然后点击“继续”。 **应记下应用程序 ID（Application Id）以便配置中使用**。
3. 在“应用程序机密（Application Secrets）”下，单击“ 生成新密码（Generate New Password）“ ，系统会随即机生成一段密码。**注意：此密码只出现一次，应立即复制并保存该密码以便配置中使用。**
4. 在”平台（Platforms）“下，单击“添加平台（Add platform）” ，然后单击`Web` 。 在”重定向URL(Redirect URLs)“输入`http://localhost:53682/` 。
5. 在”Microsoft Graph 权限（Microsoft Graph Permissions）“下 ，添加委派的权限（delegated permissions）：`Files.Read`，`Files.ReadWrite`，`Files.Read.All`，`Files.ReadWrite.All`，`offline_access`，`User.Read`。
6. 滚动到页面底部，然后单击“ 保存(Save)“ 。



下图是完成后的截图示例：

![web page](http://ww1.sinaimg.cn/large/9d89d613ly1g2iw9okpgnj20uk0rsjv5.jpg)


## 配置向导

配置示例：

客户端id:  `399f84a2-32f5-47fa-a866-d415078a91d9`

密钥：`heq****************************`

```shell
bl@bl:~$ rclone config
Current remotes:    

Name                 Type
====                 ====
onedrive             onedrive

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q> n  # 新建选择 n  (我已经配置过了1个，所以多了几个选项)
name> myod      # 自定义配置名称      	                                    
Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / A stackable unification remote, which can appear to merge the contents of several remotes
   \ "union"
 2 / Alias for a existing remote
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
13 / Hubic
   \ "hubic"
14 / JottaCloud
   \ "jottacloud"
15 / Koofr
   \ "koofr"
16 / Local Disk
   \ "local"
17 / Mega
   \ "mega"
18 / Microsoft Azure Blob Storage
   \ "azureblob"
19 / Microsoft OneDrive
   \ "onedrive"
20 / OpenDrive
   \ "opendrive"
21 / Openstack Swift (Rackspace Cloud Files, Memset Memstore, OVH)
   \ "swift"
22 / Pcloud
   \ "pcloud"
23 / QingCloud Object Storage
   \ "qingstor"
24 / SSH/SFTP Connection
   \ "sftp"
25 / Webdav
   \ "webdav"
26 / Yandex Disk
   \ "yandex"
27 / http Connection
   \ "http"
Storage> 19   # 输入OneDrive对应的编号
** See help for onedrive backend at: https://rclone.org/onedrive/ **

Microsoft App Client Id
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_id> 399f84a2-32f5-47fa-a866-d415078a91d9     #  应用/客户端id
Microsoft App Client Secret
Leave blank normally.
Enter a string value. Press Enter for the default ("").
client_secret> heq************************    # 密码/密钥
Edit advanced config? (y/n)
y) Yes
n) No
y/n> n   # n，不进行高级配置
Remote config
Make sure your Redirect URL is set to "http://localhost:53682/" in your custom config.
Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine
y) Yes
n) No
y/n> y  #  y, 同意自动配置，（浏览器会自动重定向本地请求授权，成功返回“Success! All done. Please go back to rclone.”）
If your browser doesn't open automatically go to the following link: http://127.0.0.1:53682/auth
Log in and authorize rclone for access
Waiting for code...
Got code
Choose a number from below, or type in an existing value
 1 / OneDrive Personal or Business
   \ "onedrive"
 2 / Root Sharepoint site
   \ "sharepoint"
 3 / Type in driveID
   \ "driveid"
 4 / Type in SiteID
   \ "siteid"
 5 / Search a Sharepoint site
   \ "search"
Your choice> 1   # 1，Onedrive个人或者商业版
Found 1 drives, please select the one you want to use:
0:  (personal) id=7bbf0a5dc86d77be
Chose drive to use:> 0   # 0
Found drive 'root' of type 'personal', URL: https://onedrive.live.com/?cid=7bbf0a5dc86d77be
Is that okay?
y) Yes
n) No
y/n> y   # y， 确认版本
--------------------
[myod]
type = onedrive
client_id = 399f84a2-32f5-47fa-a866-d415078a91d9
client_secret = heq****************************
token = {"access_token":"token，通过~/.config/rclone/rclone.conf文件也可以查看","expiry":"2019-04-27T03:12:32.874487926+08:00"}
drive_id = 7bbf0a5dc86d77be
drive_type = personal
--------------------
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y  # y， 确认配置
Current remotes:

Name                 Type
====                 ====
myod                 onedrive
onedrive             onedrive

e) Edit existing remote
n) New remote
d) Delete remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
e/n/d/r/c/s/q> q  # q， 退出
```



## 测试连接


```bash
bl@bl:~$ rclone lsd onedrive:
          -1 2017-06-12 19:00:15         1 Office Online 扩展
          -1 2019-04-26 23:51:59         5 oucbl
          -1 2019-04-26 01:02:53         2 图片
          -1 2019-02-26 04:03:36         9 文档
          -1 2017-01-11 22:17:39         0 电子邮件附件
```

$ **rclone lsd `onedrive`:**         # 查看当前网盘的目录。（有时候网络不稳定，需要多试几次）

`onedrive`是rclone配置名，需要修改为自己的配置名称



# 挂载Onedrive

普通用户权限也可以挂载成功，只是使用过程中可能出现一些问题，具体没深入研究；这里就直接使用管理权限。

有教程推荐使用 **screen** 在后台运行挂载命令，较稳定一点；这里就不说明了。

## 挂载远程目录到本地

示例：远程网盘目录`oucbl`挂载到本地`/home/bl/One-Drive`下：

```sh
bl@bl:~$ sudo rclone mount onedrive:/oucbl /home/bl/One-Drive --copy-links --no-gzip-encoding --no-check-certificate --allow-other --allow-non-empty --umask 000

```

相关参数解释如下：

- --copy-links - 显示软链接内容
- --no-gzip-encoding - 不使用 gzip-encoding
- --no-check-certificate - 不验证ssl证书
- --allow-other - 允许其它用户访问
- --allow-non-empty - 允许挂载目录非空
- --umask 000 - 覆写文件掩码为 000

更多参数选项请参考官方文档：https://rclone.org/commands/rclone_mount/

## 停止挂载

一般情况下使用`Ctrl+C`便可停止挂载，如果停止失败，使用如下命令停止挂载（本地`/home/bl/One-Drive`）：

```sh
bl@bl:~$ sudo fusermount -qzu /home/bl/One-Drive  #停止挂载
```

# 备注

1T Onedrive教育申请：https://products.office.com/en-us/student?tab=students

---

鉴于网络问题，暂不打算常用，这里不做开机自启配置了，具体可在参考里面2个博客中查看。

---

另一个免费OneDrive客户端：onedrive

github: https://github.com/skilion/onedrive

教程示例：https://www.maketecheasier.com/sync-onedrive-linux/

# 参考

rclone-rsunc for cloud storage--Configure

https://rclone.org/docs/

rclone-rsunc for cloud storage--Microsoft OneDrive

https://rclone.org/onedrive/



Ubuntu 18.04 手动安装 rclone 并自动挂载 Google Drive

https://timelate.com/archives/install-rclone-on-ubuntu.html#cl-11

Linux下rclone简单教程(支持VPS数据同步,多种网盘,支持挂载)

https://ymgblog.com/2018/03/09/296/

