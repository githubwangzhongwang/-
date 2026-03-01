# docker

## 安装

1.卸载系统自带的docker版本

~~~ 
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
~~~

2.更新yum包 （此过程需要等待一小会儿）

~~~ 
yum update
~~~

3.安装docker所需要的依赖包

~~~
yum install -y yum-utils device-mapper-persistent-data lvm2
~~~

4.配置yum源

~~~
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
~~~

5.查看仓库中所有的docker版本

~~~ 
yum list docker-ce --showduplicates | sort -r
~~~

能查到很长一段

6.安装docker的最新版本，不指定版本号即默认安装

~~~ 
yum install -y docker-ce
~~~

（如果要指定版本号安装可以输入命令：yum install docker-ce-18.09* -y ，此时指定的就是**docker-ce-18.09**的版本）

7.验证docker是否安装成功

~~~ 
docker version
docker info
~~~



## 配置

1.配置docker daemon的守护进程

~~~ 
cd /etc/docker/

vim /etc/docker/daemon.json
~~~

添加如下配置信息：

~~~
{

“exec-opts”:[“native.cgroupdriver=systemd”]

}
~~~

2.配置docker服务器

~~~ 
vi /usr/lib/systemd/system/docker.service
~~~

添加如下信息：

~~~ 
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
~~~

3.重新加载daemon守护程序

~~~ 
systemctl daemon-reload
~~~

4.查看docker运行状态

~~~
systemctl status docker
~~~

5.Docker 需要用户具有 sudo 权限，为了避免每次命令都输入`sudo`，可以把用户加入 Docker 用户组（[官方文档](https://docs.docker.com/install/linux/linux-postinstall/#manage-docker-as-a-non-root-user)）。

~~~
$ sudo usermod -aG docker $USER
~~~

 ExecStart=/usr/bin/dockerd -H fd:// -s overlay2 --containerd=/run/containerd/co    ntainerd.sock



## 启动

1.启动

~~~
systemctl start docker
~~~

2.关闭

~~~
systemctl stop docker
~~~

3.设置开机自启动

~~~
systemctl restart docker

systemctl enable docker
~~~









## 使用





## 总结

不行，docker刚刚安装时，可以启动，但是无法拉取镜像（也就是 docker image pull hello-word），猜测应该是imgae仓库镜像地址需要修改，这时候发现docker已经无法重启（systemctl restart docker）无法使用，docker无法启动，也关不了，都会报错

~~~
Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
~~~

猜测应该是配置/etc/docker/daemon.json文件中的内容导致的，或者是 /lib/systemd/system/docker.service文件配置时添加的

~~~
ExecStart=/usr/bin/dockerd -H fd:// 改成ExecStart=/usr/bin/dockerd -H fd:// -s overlay2 
~~~

这句配置导致的出错。原因不明。

