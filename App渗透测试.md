# App渗透测试



## mumu模拟器+Burp抓包

### 环境配置

#### 一、mumu模拟器配置

1、安装后设置mumu模拟器

```
性能为directx
网络先不开桥接				 #保证后面不会影响adb连接，配置完证书可以再打开桥接模式
其他中打开root权限			 # 添加证书需要root权限
磁盘共享设置为可写系统盘	 # 添加证书需要写入权限
机型添加任意手机号
```

2、关于本机，点击五次版本号开启开发者模式

3、配置网络

```
设置为手动代理，设置ip为本机ip地址（本机查看ipconfig，如192.168.1.46），端口8080（和burp代理地址保持一致），目的是保持模拟器和burp在同一网络，方便模拟器能连接到burp代理
```



#### 二、burp证书安装到mumu模拟器

1、安装openssl

2、burp配置，添加proxy listeners

```
添加192.168.1.46:8080		#192.168.1.46是本机地址，也可以添加0.0.0.0，监听所有流量，但是burp无法修改为0.0.0.0
```

1、burp生成证书，命名为burp.der

2、修改证书后缀，从der改为pem

```
openssl x509 -inform DER -in burp.der -out PortSwiggerCA.pem
```

3、计算修改后的证书的hash

```
openssl.exe x509 -subject_hash_old -in PortSwiggerCA.pem
```

4、修改证书名称为 9a5ba575.0 。其中9a5ba575为上面计算出的hash，后面是 点零 。



#### 三、使用adb上传证书到mumu模拟器

1、打开adb.exe文件所在位置，将证书保存在和 adb.exe 同一目录下

```
D:\tools\MuMuPlayer\nx_device\12.0\shell	# 我的是在这个位置，可能有不一样的，总之在shell下
```

2、文件夹下打开终端（cmd不行就用powershell），执行连接命令

```
.\adb.exe connect 127.0.0.1:7555	# mumu模拟器端口默认7555，如果出现连接问题，查看网络连接配置，root权限等是否配置正确
.\adb.exe -s 127.0.0.1:7555 shell	#也可以直接shell，如果报错不止一个设备，就需要-s 127.0.0.1:7555
```

3、在 adb中以 读写 方式（rw）重新挂载/分区

```
mount -o remount,rw /system		# 如果报错，可使用命令.\adb.exe -s 127.0.0.1:7555 root，然后在mumu弹窗上同意
exit
```

4、将证书复制到系统安全目录，并添加读写权限

```
.\adb.exe -s 127.0.0.1:7555 push 9a5ba575.0 /etc/security/cacerts/				# 复制证书到模拟器
.\adb.exe -s 127.0.0.1:7555 shell chmod 644 /etc/security/cacerts/9a5ba575.0	# 添加权限
```

5、查看证书是否添加成功

```
.\adb.exe -s 127.0.0.1:7555 shell
cd /etc/security/cacerts
ls -l |grep 9a5ba575.0
```

6、重启

```
reboot
```

7、查看burp是否可以抓到流量

