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