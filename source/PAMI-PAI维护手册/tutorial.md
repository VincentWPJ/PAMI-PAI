# PAMI-PAI维护手册
## PAMI-PAI计算节点配置
### 服务器配置
1、重装系统：需安装ubutu18.04系统，尽量使用原版英文系统，设置集群统一的用户名和密码

2、开机也许会循环卡住在登陆界面，无需关注（可能是因为内核和显卡驱动的原因，后面使用命令行装完驱动后可以正常进入桌面），因为我们是配置为服务器节点，所以使用 ```ctrl+alt+F3```（F1-F7随意）进入命令行即可

3、开机登录，保证网线连接，使用apt安装```sudo apt install net-tools，openssh-server```分别是用于查看本机ip地址和允许远程连接

4、使用```ifconfig```命令查看本机的IP地址，后面的步骤都使用自己的电脑远程登录操作

### 远程登录操作以下步骤
1、在自己机器上使用```ssh-copy-id username@ip_address```，将ssh公钥发送到服务器，此操作可以保证后面是免密登录，username是服务器的用户名，ip_address是服务器ip地址，紧接着会要求输入服务器密码，输入过程是不会显示的，输入完直接回车即可

2、远程登陆机器 ```ssh username@ip_address```

3、安装基础操作需要用到的包：vim（用于简单的文件编辑），ntp（保证集群时间统一），python（一般都有），nfs-common（网络文件系统），zip

4、首先设置系统不要自己更新啦，```sudo vim /etc/apt/apt.conf.d/20auto-upgrades```,将所有更新，下载都设置为"0"

5、为了保证机器的下载速度，尽量更换国内的镜像源，我选择的是清华镜像源，采用如下命令即可更新：
```
sudo sed -i "s@http://.*archive.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g" /etc/apt/sources.list
sudo sed -i "s@http://.*security.ubuntu.com@https://mirrors.tuna.tsinghua.edu.cn@g" /etc/apt/sources.list
```
6、添加镜像源后，在此基础上更新依赖：
```
sudo apt update
sudo apt upgrade
```
7、安装docker（重点）
```
sudo apt install apt-transport-https ca-certificates software-properties-common curl
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64,arm64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install docker-ce
sudo docker run hello-world
(docker最后会run一个容器，屏幕上若出现"Hello from Docker!"，则表明docker安装成功)
```
8、安装nvidia-docker
```
（如果系统装过nvidia-docker或其他GPU容器，先卸载）
sudo docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo apt-get purge -y nvidia-docker
```
添加仓库
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
```
安装nvidia-docker2
```
sudo apt-get install -y nvidia-docker2
sudo pkill -SIGHUP dockerd
sudo docker run --runtime=nvidia --rm nvidia/cuda:10.1-base nvidia-smi
（同样地，最后一步是run一个nvidia的官方容器，并答应nvidia-smi信息，若正常出现显卡信息，则证明nvidia-docker安装成功）
```
安装完成后，每次使用docker命令，都需要sudo，可以通过给docker sudo权限解决这个问题：（该命令会使得当前用户拥有sudo权限，谨慎使用该命令）
```
sudo groupadd docker
sudo usermod -aG docker $USER  # 此处 $USER 无需替换，直接使用即可
newgrp docker
```
注意：（一般不会发生这种事情）某些局域网内会使用127.17.0.0网段，导致因docker默认产生的网桥地址和服务器地址冲突，所有数据会无法正常传输。参考[Docker-Compose导致ssh连接断开的问题](https://blog.siaimes.me/2021/04/29/p55.html)
```
查看此时docker网桥的IP地址：
ifconfig
我服务器的docker0是127.17.0.1

在/etc/docker/daemon.json中添加：
{
  "default-address-pools": [{"base":"10.10.0.0/16","size":24}]
}

修改完json文件后重启docker：
sudo systemctl restart docker

再次查看docker网桥的IP地址：
ifconfig
此时我服务器的docker0是10.10.0.1
```
9、安装GPU驱动
```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
sudo apt install nvidia-driver-470 #根据自己显卡情况选择
sudo apt autoremove xserver-xorg
sudo apt autoremove --purge xserver-xorg
sudo apt-mark hold nvidia-driver-470 # Freeze NVIDIA Drivers
nvidia-smi #查看显卡信息
nvidia-smi -a | grep Product\ Name #查看显卡详细名称，nvidia-smi有时会省略部分名称
```
10、安装nvidia-container-runtime
```
(前文已添加过仓库，直接安装即可)
sudo apt update
sudo apt install nvidia-container-runtime

查看/etc/docker/daemon.json信息：
{
  "default-address-pools": [{"base":"10.10.0.0/16","size":24}]
  "default-runtime": "nvidia",
  "runtimes": {
      "nvidia": {
          "path": "/usr/bin/nvidia-container-runtime",
          "runtimeArgs": []
      }
  },
}
**一定要添加"default-runtime": "nvidia"，否则后面会报错：The default runtime is not set correctly

重启docker服务
sudo systemctl restart docker
```

11、设置GPU常驻内存，解决GPU初始化缓慢，无任务时利用率居高不下，偶尔丢卡。
```
在/lib/systemd/system/rc-local.service中添加以下内容：
[Install]
WantedBy=multi-user.target
Alias=rc-local.service

设置rc-local开机自启：（此步会设置软连接）
sudo systemctl enable rc-local

在/etc/rc.local中添加以下内容：
#!/bin/sh -e
nvidia-smi -pm 1
exit 0

修改执行权限：
sudo chmod +x /etc/rc.local
```

### 容器dev-box（管理集群）
创建docker 容器管理dev-box：
```
sudo docker run -itd \
        -e COLUMNS=$COLUMNS -e LINES=$LINES -e TERM=$TERM \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v ${HOME}/pai-deploy:/home/pai-deploy  \
        -v ${HOME}/pai:/home/pai \    
        -v $(HOME)/.ssh:/root/.ssh    
        --pid=host \
        --privileged=true \
        --net=host \
        --name=dev-box \
        --restart=always \
        siaimes/dev-box:v1.8.1
```
### Q&A
master机器无法使用kubectl，报错：The connection to the server localhost:8080 was refused - did you specify the right host or port?
解决方案：
```
mkdir -p #HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
可以正常使用kubectl get nodes


## PAMI-PAI存储节点配置
### 存储节点硬件配置
### 存储节点文件系统

## 集群网络环境
### 网络硬件配置 