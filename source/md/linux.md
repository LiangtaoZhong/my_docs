# Linux常见问题

##  Ubuntu设置静态IP

将linux开发板与电脑网口连接，方便调试，需要将**电脑有线网卡**和**开发板的有线网卡**配置到同一局域网下。操作如下：

1. 打开配置文件

   ```shell
   cd /etc/network
   vim /interfaces
   ```

   

2. 添加如下配置：

   ```shell
   auto eth0
   iface eth0 inet static
   address 192.168.0.11
   gateway 192.168.0.1
   netmask 255.255.255.0
   ```

   其中`eth0`为要配置的网卡。`address`为IP地址，`gateway`为网关，`netmask`为子网掩码。根据需要设置。

3. 测试

   双向互Ping，若能ping通，则在同一局域网。

   **需要注意的是有时候linux开发板功能较少，并没有对ping进行相应的处理，这时候用电脑去ping开发板可能会出现不能ping通的情况。**

## ubuntu开机自启动

1. 为程序创建一个`.sh`的启动脚本。例如`demorun.sh`，保存的路径为`/root/demorun.sh`。
2. 更改文件权限为可运行`chmod +x demorun.sh`。
3. 然后把`/root/demorun.sh&`写入`/etc/rc.local`文件中的`exit 0`前面。
4. 其中`&`的的意思为后台运行，否则会阻塞正常开机。

## ubuntu查看和设置环境变量

### 查看环境变量

1. `env`
   env命令是environment的缩写，用于列出所有的环境变量

2. `export`

   单独使用`export`命令也可以像`env`列出所有的环境变量，不过`export`命令还有其他额外的功能

3. `echo $PATH`

   `echo $PATH`用于列出变量`PATH`的值，里面包含了已添加的目录

### 设置环境变量

1. 追加某个环境变量的值

   ```shell
   # 加到PATH末尾
   export PATH=$PATH:/path/to/your/dir
    
   # 加到PATH开头
   export PATH=/path/to/your/dir:$PATH
   ```

   **PATH表示要追加到的变量名，注意先后顺序**

2. 命名一个新的环境变量

   ```shell
   export VAR_NAME=value
   ```

### 环境变量作用域

1. 当前终端
   打开终端输入`export VAR_NAME=value`
2. 当前用户
   将环境变量的设置语句写入`~/.bashrc`,然后`source ~/.bashrc`
3. 所有用户
   将环境变量设置语句写入`/etc/profile`，然后`source ~/etc/profile`



## 双系统共享硬盘

有两种方法：

> 1. 先进入Windows再==**重启**==进入ubuntu。
> 2. 关闭windows的快速启动。

## rc.local执行权限

rc.local由init进程调度，所以执行权限为root。

## ubuntu解压中文名压缩包乱码

windows下打包的压缩包为GBK/GB2312编码，Linux为UTF-8编码。因此linux下解压windows压缩包会乱码。解决办法：

```shell
unzip -O GBK xxx.zip
```

## 创建文件/文件夹快捷方式(创建软链接)

其实就是创建文件或者文件夹的==软链接==。方法如下：

```shell
sudo ln -s /src/file /dist/file   #不加-s为添加硬链接
```

一定要使用**绝对路径**，否则会链接出错。

## GitHub下载加速（亲测有效，前提FQ）

```shell
# Ubuntu 下命令为 expo

export http_proxy=http://192.168.0.102:10809
export https_proxy=http://192.168.0.102:10809
export http_proxy_user=user
export http_proxy_pass=pass
export https_proxy_user=user
export https_proxy_pass=pass

#windows cmd
set http_proxy=http://127.0.0.1:10809
set https_proxy=http://127.0.0.1:10809
set http_proxy_user=user
set http_proxy_pass=pass
set https_proxy_user=user
set https_proxy_pass=pass

# 恢复
set http_proxy=
set https_proxy=

```

## 恢复误删的.bashrc

备份文件在`/etc/skel/`，复制到用户目录下就好。

## 树莓派等arm架构更新清华源

### x86架构

```shell
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```

### arm架构

将x86中的`https://mirrors.tuna.tsinghua.edu.cn/ubuntu/`改为`https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/`即可。

## ubuntu改变屏幕亮度

1. `$ sudo add-apt-repository ppa:apandada1/brightness-controller` 
2. `$ sudo apt-get update` 
3. `$ sudo apt-get install brightness-controller` 
4. `$ brightness-controler` 

## ubuntu挂载和卸载U盘

1. 查看U盘设备 形如`/dev/sd*`

   ```shell
   sudo fdisk -l 
   ```

2. 挂载U盘

   ```shell
   sudo mount /dev/sd* /mnt # mount [设备] [挂载点]
   ```

3. 卸载U盘

   ```shell
   sudo umount /dev/sd* # umount [设备]
   ```

   

## ubuntu NVIDIA驱动

### ubuntu安装NVIDIA驱动

0. 退出图形界面  `Ctrl+ALT+F1`，进入命令行界面
   如果第一次安装先禁用系统自带的驱动

   > 1. sudo vim /etc/modrpobe.d/black-nouveau.conf  打开配置文件
   >
   > 2. 添加如下
   >
   >    ```
   >    blacklist nouveau
   >    options nouveau modeset=0
   >    ```
   >
   > 3. lsmod | grep nouveau 命令无返回结果表示禁用成功

1. 停用服务

   ```shell
   sudo service lightdm stop
   ```

2. 卸载驱动

   ```shell
   sudo apt-get remove --purge nvidia*
   ```

3. 安装驱动

   ```shell
   sudo ./NVIDIA-Linux-x86_64-*.run –no-opengl-files
   ```

     只安装驱动文件，不安装OpenGL文件。这个参数最重要

4. 重新打开服务

   ```shell
   sudo service lightdm start
   ```

5. 重启

### 查看显卡信息

1. 查看 Nvidia 显卡利用率：显存占用和算力情况

   ```shell
   watch -n 0.5 nvidia-smi #0.5 秒更新一次显卡利用情况，并查看 NVIDIA 驱动版本
   ```

2. 查看 Cuda 版本

   ```shell
   cat  /usr/local/cuda/version.txt
   ```

3. 查看 Cudnn 版本

   ```shell
   cat /usr/local/cuda/include/cudnn.h | grep CUDNN_MAJOR -A 2
   ```

### nvcc 正常、nvidia-smi失败

1. ```shell
   sudo apt-get install dkms  
   ```
   
2. 通过` cd /usr/src  &&  ls`  查看对应安装的驱动版本

3. 重新生成对应nvidia的驱动模块  

   sudo dkms install -m nvidia -v +对应的驱动版本号（我是440.64）即为：  

   ```shell
   sudo dkms install -m nvidia -v 440.64  
   ```
   
4. 再次执行：NVIDIA-SMI 即可