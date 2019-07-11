---
title: 关于json的理解 简单又深刻
date: 2018-07-24 22:31:45
tags:
---

JSON

简单的说，没有真正的Json对象，只有Json规则，Json字符串。格式无须赘述。理解所谓的对象、字符串，在传输过程中的处理与细节，比如编码的时候我们所以为的字符串，到底是什么样子，包不包含转义符，包含了多少，哪些转义符是有的，哪些是没有的，都是考验基本功的地方。先上问题的初始，看数据：对比下，pc端 request和response和android端 request和response，会发现有所不同。

<!--more-->

<font color=red>PC端：
kyc1 认证提交：request</font>

POST /api/user/v3/users/kyc/level-first HTTP/1.1
Host: dev.btb-inc.com
Content-Length: 111
Accept: application/json
Cache-Control: no-cache
Origin: http://dev.btb-inc.com
Authorization: eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJkZGUyMTZmMS05NmVhLTQ5NTAtYTNhZC01MDk0NmNlYTg1NGRZZ25zIiwidWlkIjoiMk5veURDS241dC9MT01NQUNtWlZyZz09Iiwic3ViIjoiMTg1KioqNjU4OSIsInN0YSI6MCwibWlkIjowLCJpYXQiOjE1MzIzNTM4MDMsImV4cCI6MTUzMjk1ODYwMywiaXNzIjoiYnRiIn0.nX-GZXm8qBijYaGcyYE17UiTz4y9uAAbOkYpyYTbqKV8gOFzwZfeSQ4zmDLggHfnrlha3t9uXUwzndsotQ8IuA
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36
Content-Type: application/json
Referer: http://dev.btb-inc.com/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: UM_distinctid=1634a0c347a573-09b27d26c9648b-3961430f-1fa400-1634a0c347b414; locale=zh-CN; CNZZDATA1273244937=1115560523-1527508387-%7C1532351300
Connection: keep-alive

<font color=green>{"countryId":"CN","form":"{\"countryId\":\"CN\",\"fullName\":\"问题系\",\"idCard\":\"370781198602262259\"}"}</font>

<br>
<font color=red>回复response</font>
HTTP/1.1 200 
Server: nginx
Date: Mon, 23 Jul 2018 14:06:58 GMT
Content-Type: application/json
Transfer-Encoding: chunked
X-Application-Context: api-gateway:8080
Expires: 0
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
X-Frame-Options: SAMEORIGIN
Proxy-Connection: keep-alive

{"code":3003,"data":null,"msg":"此身份证已认证"}

<font color=red>客户端：
kyc1 认证提交：request</font>
POST /api/user/v3/users/kyc/level-first HTTP/1.1
Authorization: eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiI2Mzc2OGMwZC1mZTY0LTRmMGEtYTIyZC0xMzAwNDdkY2YxMTlaWVhLIiwidWlkIjoiTmpBcy9tV1JzSE5BTWNCZkxmUHNlZz09Iiwic3ViIjoiMTg1KioqMjY1OCIsInN0YSI6MCwibWlkIjowLCJpYXQiOjE1MzIzNTY1MTcsImV4cCI6MTUzMjk2MTMxNywiaXNzIjoiYnRiIn0.gd9EM3z3ivTGjpFTmHvLzkNRJyhWS0w3YT8__SXk69klPBkMG8OEPYMwWzcsveH21sx0aEFrGInBtoxvOPvyAw
Cookie: locale=zh-CN;
User-Agent: Mozilla/5.0 (Linux; Android 6.0; MI 5C Build/MRA58K; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/67.0.3396.87 Mobile Safari/537.36|btb/1.3.1
Accept-Language: zh-CN
Content-Type: application/json; charset=utf-8
Content-Length: 97
Host: dev.btb-inc.com
Accept-Encoding: gzip
Connection: keep-alive

<font color=blue>{"countryId":"CN","form":{"countryId":"CN","fullName":"问题系","idCard":"370781198602262259"}}</font>

<br>
<font color=red>回复response</font>
HTTP/1.1 200 
Server: nginx
Date: Mon, 23 Jul 2018 14:36:11 GMT
Content-Type: application/json
Transfer-Encoding: chunked
X-Application-Context: api-gateway:8080
Expires: 0
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
X-Frame-Options: SAMEORIGIN
Connection: keep-alive

{"code":3003,"data":null,"msg":"此身份证已认证"}

绿蓝两个地方就是在提交form的时候不同的地方，说白了蓝色是定义了一个大的实体bean-wholeBean，包含form的实体，然后gson.toJson(wholeBean)绿色是大的实体bean，的form并不是一个实体，而是对应一个Json字符串，字符串的内容是对应form形式的实体转过的Json字符串，也就是说
form部分不是实体，是gson.toJson(formBean),然后整体还是gson.toJson(wholeBean),
<font color=red>本质区别就是wholeBean的from字段，到底是String，还是一个Object(formBean)</font>

但是在语言中，Json本身就是一个字符串，json格式化也是严格的，其中都带有""等严格规范，那么一个最简单的json，形如{"a":"abc"}，在传输过程中，或者说作为一个字符串，本身就是被""包围的，里面怎么保证整体呢，答案就是基础的转义字符，其实我们看到的并非我们看到的，调试过程中看到的String json = "{"a":"abc"}"其实是"{\"a\":\"abc\"}"。这就是问题的关键，然后类似绿色字体的内容实际应该什么样子呢？看调试的截图就知道了，如下（注意看开发ide给的调试值的颜色，都是有讲究的，小橙色其实都是带了转义的，只不过ide为了开发人员看方便，都给去掉了）
注意对比代码和debug信息，看清楚就理解了。

图1，代码部分
![代码部分](https://app.yinxiang.com/shard/s59/res/c3f1fd40-c7bd-408c-9b37-c9435af1e9bf/json%E5%88%86%E6%9E%901.png)

图2，debug信息部分
![debug部分](https://app.yinxiang.com/shard/s59/res/6c73bb97-0caa-4443-aa6e-895afa9f5978/json%E5%88%86%E6%9E%902.png)

Over，细节决定成败。
字符串中的"和\都需要转义。简单情况只转义",如"{"a":"abc"}" ---> "{\"a\":\"abc\"}"

复杂情况，就是
"{\"otcId\":2,\"pair\":
\"{\\\"countryId\\\":\\\"CN\\\",
\\\"form\\\":{\\\"countryId\\\":\\\"CN\\\",\\\"fullName\\\":\\\"问题系\\\"}}\"}"
这种，其实就是"{\"countryId\":\"CN\",\"form\":{\"countryId\":\"CN\",\"fullName\":\"问题系\"}}"转出来的

但是放到json里，最终还是要再加一轮转义符，不过开发人员不需要关心，只是在编码传输（http等）中需要这样，json的底层库都会给处理掉，
我们只关心最终的语义部分，跟协议的玩法是一样的。

问题：<font color=red>json字符串，作为参数传输，在http传输层，编码之后，到底包不包含转义字符？</font>
