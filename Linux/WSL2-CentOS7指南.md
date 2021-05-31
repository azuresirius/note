# WSL2-CentOS7 安装指南

wsl2下安装centos7，oracle 18c XE，Docker

## 准备工作

- win10版本更新到1903以后
- 下载CentOS7 wsl2版本 [CentOS 7.9](https://github.com/mishamosher/CentOS-WSL/releases/download/7.9-2009/CentOS7.zip)
- 下载windows wsl [linux内核更新](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
- 下载oracle 18c xe [离线安装包 3G左右](https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-18c-1.0-1.x86_64.rpm?AuthParam=1621588011_e8afdd31f97c11dd364fbbc2ccb0fe60)

## 开始安装

### 启用虚拟机功能

以管理员身份打开PowerShell并运行

`dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart`

**重新启动计算机**
    
### 安装Linux内核更新包

运行安装之前下载的wsl_update_x64.msi

### 将WSL 2 设置为默认版本

`wsl --set-default-version 2`

### 安装CentOS 7

将之前下载好的CentOS文件解压缩，得到CentOS.exe文件，右键管理员运行，安装自动进行，会在当前目录生成ext4.vhdx，就是虚拟磁盘。

如果要卸载，则是在命令行里运行 `Centos.exe clear` 即可。

### 设置CentOS

#### 首先运行`yum update`更新系统
#### 更换系统systemctl组件
    
由于这不是真正意义的Linux，所以系统的1号进程不是通常的/sbin/init，而是windows自己的/init，而这导致了systemctl命令会出现Failed to get D-Bus connection: Operation not permitted报错，所以采用别人给docker内系统设计的systemctl来替换原本的（docker也是虚拟化）。运行
```
curl https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py >temp
mv /usr/bin/systemctl /usr/bin/systemctl.old
mv temp /usr/bin/systemctl
chmod +x /usr/bin/systemctl
```
#### 设置网络环境
  
由于Windows和WSL2算是在一个局域网内的，这个局域网是由Hyper-V创建的。WSL2使用的网络适配器是'Default Hyper-V Switch'，这个适配器每次重启都会被删除重建，这会导致WSL2每次重启之后IP都会不固定。我们需要解决的问题是如何从windows访问WSL2（这个是最主要诉求，这个linux存在的意义就是让我们开发时去模拟真实的linux环境）和WSL2访问windows。主要解决思路来自于这篇文章[WSL2 的一些网络访问问题](https://lengthmin.me/posts/wsl2-network-tricks/)
  
这个命令可以获取当前windows主机的ip
```
ip route | grep default | awk '{print $3}'
# 或者
cat /etc/resolv.conf | grep nameserver | awk '{ print $2 }'
```

解决方案是在重启电脑（具体是在适配器分配IP）的时候，运行一个**任务计划程序**来执行powershell脚本将当前的wsl地址和windows地址都写入到hosts文件中（这个hosts文件在两个系统内是共享的），以后直接通过域名来访问各自的系统。
1. 将[这个链接](https://github.com/lengthmin/dotfiles/blob/master/windows/wsl2.ps1)中的代码保存到本地。
2. 打开**事件查看器** -> Windows日志 -> 系统 -> 找到`Hyper-V-VmSwitch`事件，找到类似`Port ... Friendly Name: WSL ... successfully created on switch ..`的内容。
3. 右键单击该事件，选择**将任务附加到该事件**。
4. **操作**选择**启动程序**， 程序中填写`powershell`，参数填`-file path\wsl2.ps1`。
5. 点击确定后会立即跳出事件的任务属性，其中找到**使用最高权限运行**并勾线即可。

上面那个脚本里的域名可以根据自己的喜好更改，而需要映射到局域网中的端口也可以在`$ports=@(80,443,8080)`中自己添加。

至此，基本的设定就算完成了。

### 安装各种软件

#### 安装Docker

docker的安装过程给我带来了极大的痛苦，试过了很多办法最后都是无法启动，原因也没找到，所以请一定不要手动自己安装，去微软官方下载Docker Desktop就好了[下载地址](https://www.docker.com/products/docker-desktop)，安装完成之后，可以选择添加docker到WSL2里，他会帮你全部设置完，太贴心了！

#### 安装Oracle

Oracle毕竟是商业的，所以我们选择免费的XE版本来安装，普通开发调试已经足够了。因为我们使用CentOS系统，所以官方提供了一个给这个类RHEL系统的预安装脚本，这里面帮你完成了很多的前期工作，别问我怎么知道的，我本来想把Oracle装在Debian上，由于前期工作过于复杂而放弃了。
```yum -y localinstall https://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64/getPackage/oracle-database-preinstall-18c-1.0-1.el7.x86_64.rpm```
本地硬盘的内容都在`/mnt/`下面，用`ln -s source destination`来设置软链接可以大大提高便利性哦。
```
ln -s /mnt/d/temp/oracle-database-xe-18c-1.0-1.x86_64.rpm oracle-database-xe-18c-1.0-1.x86_64.rpm
yum install -y oracle-database-xe-18c-1.0-1.x86_64.rpm
```
安装完成！


接下来是设置oracle和添加实例。请一定在这时使用命令`hostname 你设置的wsl域名`并重新打开linux shell来设置域名，要不然oracle在安装listener的时候会出错或者安装到ipv6上，我也不知道为什么。接着运行`/etc/init.d/oracle-xe-18c configure`来安装实例和配置，运行结束则配置完成，可以使用`netstat -tunlp`查看现在监听的端口，有5500和1521则表示成功。


**至此安装全部完成**整个过程中还是踩了不少坑的，wsl2依然没有很完善，期待微软后期继续改进吧。

