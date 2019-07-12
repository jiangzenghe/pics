---
title: 关于跨域的设置（Header Editor）
date: 2018-07-17 21:57:20
tags:
---

前的跨域设置都比较初级，一般采用的方法是：
windows

	右键 chrome 浏览器快捷图标的属性，在快捷方式标签页的目标输入框的最后加上：
	--args --disable-web-security --user-data-dir=C:\ChromeUserData

mac

	终端启动：
	$ open -a "Google Chrome" --args --disable-web-security --user-data-dir=~/ChromeUserData/

或者将上面的命令保存为 chrome.txt 纯文本文件，右键文件图标的显示简介，找到打开方式选定为“终端”，保存后双击文件运行。

<!--more-->

后来发现一个方便又好用的google chrome 插件Header Editor，然后分别在请求头和响应头中添加规则就好了，
重要的是对规则的理解，其实就是用js篡改请求头和响应头中的信息，重点：请求头和响应头中那些东西是合跨域有关的，如何影响。

下载后新建2条规则：
1.修改请求头
规则类型：修改请求头；
匹配类型：全部；
执行类型：自定义函数；


在输入框内插入以下代码：

```
var removeHeaderNameMap = {
//origin: 1,
//'user-agent': 1,
'access-control-request-method': 1,
'access-control-request-headers': 1
}
var acrm = 'access-control-request-method'
var acrh = 'access-control-request-headers'
var headers = val.filter(h => !removeHeaderNameMap[h.name.toLowerCase()])
val.length = 0
Array.prototype.push.apply(val, headers)
```
2.修改响应头
规则类型：修改响应头；
匹配类型：全部；
执行类型：自定义函数；


在输入框内插入以下代码：
```
var acao = 'Access-Control-Allow-Origin'
var acam = 'Access-Control-Allow-Methods'
var acah = 'Access-Control-Allow-Headers'
var removeHeaderNameMap = {}

var isModified = val.some(h => {
if (h.name === acao) {
h.value = '*'
return true
}
})
isModified || val.push({name: acao, value: '*'})
val.push({name: acam, value: 'POST,DELETE,INPUT,HEAD,OPTIONS,TRACE,CONNECT'})
val.push({name: acah, value: 'Authorization,authorization,content-type,necaptchavalidate'})
```

备注：
访问内网的时候没有启用跨域的请求头和响应头设置，但是访问正式环境的时候，出现了跨域的提示。然后折腾半天，分四种情况：
1.请求头、响应头修改都不开
错误
1）<font color=red>Failed to load https://www.btb.com/api/account/asset/detail:</font>

![情况1](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-1.png?raw=true)

<font color=red>Response for preflight has invalid HTTP status code 500.</font>

2）<font color=red>Failed to load https://www.btb.com/api/bops-operation/bops/operation/banner/list?currentPage=1&device=0&pageSize=10:</font>

![情况2](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-2.png?raw=true)

<font color=red>Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://192.168.8.50:8080' is therefore not allowed access. The response had HTTP status code 403.</font>

2.请求头修改 开、响应头修改 不开
错误
<font color=red>Failed to load https://www.btb.com/api/account/asset/detail:</font>

![情况3](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-3.png?raw=true)

<font color=red>Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://192.168.8.50:8080' is therefore not allowed access.
（200 都是200）</font>
![情况4](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-4.png?raw=true)

3.请求头修改 不开、响应头修改 开
错误
1）<font color=red>Failed to load https://www.btb.com/api/account/asset/detail:</font>

![情况5](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-5.png?raw=true)

<font color=red>Response to preflight request doesn't pass access control check: The 'Access-Control-Allow-Origin' header contains multiple values 'http://192.168.8.50:8080, *', but only one is allowed. Origin 'http://192.168.8.50:8080' is therefore not allowed access.
（500）</font>

2）<font color=red>Failed to load https://www.btb.com/api/bops-operation/bops/operation/banner/list?currentPage=1&device=0&pageSize=10:</font>

![情况6](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-6.png?raw=true)

<font color=red>Response for preflight has invalid HTTP status code 403.</font>

4.请求头和响应头修改都开
正常
1）detail

![情况7](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-7.png?raw=true)

2）banner

![情况8](https://github.com/jiangzenghe/pics/blob/master/tech/20180717215720-8.png?raw=true)

原理分析：
相关跨域的基础分析：
可能需要一个链接--
上述场景的自己分析：
1.代码设置分析

2.场景出现分析
