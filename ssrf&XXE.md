# ssrf

服务器调用外部资源，其中外部资源的地址由用户定义，用户（攻击者）利用被调资源对服务器的信任，访问到敏感资源或其他信息。

### curl_exec（）

后台代码由函数curl_exec()来执行

### file_get_content（）

后台代码由函数file_get_content()来执行

### payload

1）http协议的可以访问内网的其他地址，端口，进行内网扫描，从回显信息中判断。

2）使用dict://协议、Gopher://协议、file://伪协议等来读取文件。

3）/ssrf_curl.php?url=file:///etc/passwd

# XXE

（XML External Entity Injection）外部实体注入，由于程序在解析输入的XML数据时，解析了攻击者伪造的外部实体而产生的。例如PHP中的simplexml_load默认情况下会解析外部实体，有XXE漏洞的标志性函数为simplexml_load_string()。

### XML基础

XML 指可扩展标记语言（EXtensible Markup Language）
XML 是一种标记语言，很类似 HTML
XML 的设计宗旨是传输数据，而非显示数据
XML 标签没有被预定义。您需要自行定义标签。
XML 被设计为具有自我描述性。
XML 是 W3C 的推荐标准



### 利用条件

1）xml外部实体引用，需要用到libxml，其版本2.9以上默认关闭外部实体引用，以上的版本需要手动开启

​		simplexml_load_string($xml , LIBXML_NOENT)。其中LIBXML_NONET为外部引用的开关。

2）目标主机没有禁用外部实体的引用。

3）用户可以控制XML的输入内容。

### payload

~~~
<?xml version="1.0"?> 
<!DOCTYPE foo [    
<!ENTITY xxe SYSTEM "file:///etc/passwd" > ]> 
<foo>&xxe;</foo>
~~~





