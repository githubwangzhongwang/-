# ThinkPHP反序列化漏洞



## 一、安装Composer

用于获取不同版本的ThinkPHP，需要安装Composer（php用来管理依赖关系的工具，类似于ptyhon中的pop工具）。

#### 1、配置php运行环境

创建软连接（类似于windows中创建的快捷方式），将php的运行目录链接到usr下：

~~~
ln -s /opt/lampp/bin/php  /usr/local/bin/php
~~~

这样，可以在任意目录下执行php命令（实现的功能类似于配置环境变量，但用软连接的方式代替了）。

#### 2、下载Composer并安装

~~~
curl -sS https://getcomposer.org/installer | php
~~~

上述命令，从连接中下载installer程序，并交给php命令运行，这样，便会在当前目录中安装好composer.phar。

#### 3、将composer.phar移动到/usr/local/bin目录下并重命名为composer

~~~
mv composer.phar /user/local/bin/composer
~~~

#### 4、配置国内镜像

~~~
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
~~~



## 二、安装ThinkPHP

#### 1、安装5.1.25

~~~
composer create-project topthink/think tpdemoo 5.1.25 --prefer-dist
~~~

thinkphp会安装到当前目录中，但安装的是5.1.*（其中`*`表示最新版本），想要安装5.1.25，而不是最新版本

#### 2、安装指定版本

在安装的tpdemo目录中，打开composer.json，指定版本：

~~~
"require": {
	"php": ">=5.6.0",
	"topthink/framework": "5.1.25"
}
~~~

制定好版本，进行更新：

~~~
composer update
~~~



## 三、反序列化



知识点：

1，抽象类

抽象类使用abstract来修饰：

~~~
abstract class a
~~~

这种抽象类，不能实例化，是用来让子类继承的，



2，trait类，use的使用方法

~~~
<?php
trait A {
	function aa(){
		echo 'a';
		$this->cc();
	}
}
trait B {
	function bb() {
		echo 'b';
	}
}
class C {
	use A;
	use B;
	function cc() {
		// $this->aa();
		echo "cc";
	}
}
$c = new C();
$c->cc();
?>
~~~

这样类C中，调用了其他类（A）中的方法，正常来说调用不会成功，但是使用了use，将A和Buse进来了，这样C类中可以调用a和b类的方法，比较简单就可以理解。

但是A类中最下面，在A类中调用了B类中的方法，也会正常执行；类C当中的方法，A类中和B类中也可以正常调用。因为use的过程相当于C类将A类中的代码，B类中的代码全都加入到了C类中，也就是A，B，C形成了一个大的类，A类，B类，C类的实例都可以调用他们中的任意的方法。
