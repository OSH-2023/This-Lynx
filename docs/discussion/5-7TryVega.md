# 5-7 尝试运行vega

## 尝试在同一物理机中的多台虚拟机上运行vega

该文档描述尝试使用vega在同一物理机中的多台虚拟机上运行的过程。理论上只要在同一局域网下，过程应该是一样的。
下面列举的过程不一定全为必要项。因为程序未能顺利执行完，无法排除非必要的步骤。

#### 1. 配置slave的ssh：
**下载ssh server：**
    `sudo apt-get install openssh-server`
**配置/etc/sshd/sshd_config：**
    首先`sudo vim /etc/ssh/sshd_config`
    然后将PubkeyAuthentication改为yes
    PasswordAuthentication改为yes
    将PermitUserEnvironment改为yes
    上面修改了的项均需取消注释
**配置公钥**
    在slave机的家目录/.ssh/authorized_keys文件末尾加入master机的公钥
**配置vega所需的environment**
    在slave的家目录/.ssh/environment文件中写入`VEGA_LOCAL_IP=localIP`，其中localIP为slave机的ip地址
**新增用于远程连接的用户**
    `sudo useradd username`，将username换成自己想要的用户名即可
    然后`sudo passwd username`，设置密码
**启动ssh**
    `service ssh start`，如果中途修改了配置，则需`service ssh restart`
**尝试连接**
    可以尝试用master机通过ssh连接slave。输入`ssh username@slaveIP`，按提示输入密码即可。其中username为前面设置的slave中用于远程连接的用户名，slaveIP为slave机的IP地址。如果成功连接则说明ssh配置无误。

#### 2. 配置VEGA
**配置hosts.conf**
    将master家目录的hosts.conf文件改为：
    ```
    master = "masterIP"
    slaves = ["username@slaveIP"]
    ```
    masterIP为master机的IP地址，username为slave机中用于远程连接的用户名，slaveIP为slave机的IP地址。引号不能去掉，若有多个slave可以在中括号中添加多个`"username@slaveIP"`，用逗号分隔（多slave未测试，该文档描述的过程为单slave的情况）。
    将slave的hosts.conf文件改为相同的内容。
**设置环境变量**
    在master的vega文件夹下，在终端运行指令：
    ```bash
    export VEGA_DEPLOYMENT_MODE=distributed
    export VEGA_LOCAL_IP=masterIP
    ```
    其中masterIP即为master的IP地址。

#### 3. 运行
在vega文件夹下`cargo run --example make_rdd`，顺利通过编译并提示输入密码，但无论输入密码正确与否，均继续要求输入密码，无法执行下一步。

## 尝试局域网内连接到各自的虚拟机上

### WSL2

1. 因为WSL内部的openssh-server不完全，需要重新安装一下

```shell
$ sudo apt-get remove openssh-server
$ sudo apt-get install openssh-server
```

2. 配置WSL内部的ssh，以下均需要管理员权限

```
# /etc/ssh/sshd_config
...
Port 2223
...
PermitRootLogin yes
...
PasswordAuthentication yes
...
```

```
# /etc/hosts.allow
...
sshd: ALL
...
```

3. 重启ssh服务

```shell
$ sudo service ssh --full-restart
```

4. 查看WSL内部的IP地址

```shell
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.11.54  netmask 255.255.240.0  broadcast 172.18.15.255
        ...
```

如上中`172.18.11.54`即为对应的IP地址

5. Windows下管理员权限打开CMD，输入命令

```cmd
> netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=2222 connectaddress=172.18.11.54 connectport=2223
```

其中`listenaddress`指向本地所有地址，`listenport`为监听端口，`connectaddress`为WSL内部的IP地址，`connectport`为WSL内部的ssh服务端口，即`/etc/ssh/sshd_config`中的`Port`项

6. 查看Windows下的IP地址

```cmd
> ipconfig
...
Wireless LAN adapter WLAN:

   Connection-specific DNS Suffix  . : ustc.edu.cn
   Link-local IPv6 Address . . . . . : fe80::e2e5:debc:bc26:d885%9
   IPv4 Address. . . . . . . . . . . : 100.65.31.103
   Subnet Mask . . . . . . . . . . . : 255.255.192.0
   Default Gateway . . . . . . . . . : 100.65.0.1
...
```

7. 在局域网下其余主机上使用ssh连接即可

```shell
$ ssh root@100.65.31.103 -p 2222
```