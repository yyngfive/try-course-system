# 简单探索一些南大新选课系统

新选课系统网址：xk.nju.edu.cn

## 前言
注意到老系统计划停用的消息，打算读一下新系统的代码，试着从新系统中爬取课程。我所做的工作约等于用Python简单地模拟了一下登录流程，并以获取当前课表为例实际应用了一下。代码本身比较粗略，但一步步分析的过程还是很有趣的。

新系统采用了金智的选课系统，网上可能已经有各种抢课之类的代码了，但新系统对我来说还是很新鲜的。所以我在这里也不重复造轮子了，只是简单 梳理一下我的探索过程

## 登录新系统
首先要说明的是，新系统提供了十分丰富的查询函数，足够用的代码注释以及十分令人头疼的汉语拼音缩写。~当然，其中也有诸如“ocde”这种拼写错误。~

先提交一个登录请求，发现向`login.do`通过post方式提交了四个参数：
|name|value|
|----|-----|
|loginName|学号|
|loginPwd|加密后的密码|
|verifyCode|验证码|
|vtoken|验证码的token|

要完成登录，需要获取这四个参数。首先，最简单的是学号，通过手动输入。然后是密码部分。阅读源码，发现系统对密码进行了两次处理。首先是使用了`strEnc`对密码进行加密，并且使用了三个额外的密钥，之后将加密后的结果进行Base64编码。

寻找加密函数的定义，发现在`des.min.js`中。那么可以基本确定用的是DES加密，不过我对加密并不是很了解，也不清楚它使用的加密模式和填充算法。所以这里直接通过`Pyexecjs`调用这个JS函数。

接下来只要知道三个Key是什么就可以了。简单搜索可以发现，这三个Key分别是“this”“password”“is”。这就有些敷衍了。统一身份认证的登录界面使用AES加密密码，其中密钥应该是从服务器获取的随机Salt，把加密数据和64位的随机字符串拼接后，还添加了16位的随机偏置向量。

不过，这样就很容易得到加密后的密码。之后是验证码的问题，只要获取验证码的图片即可。流程大概是这样的：

通过post方式请求`vcode.do`获得一个验证码的随机token->将token改名为vtoken->使用vtoken作为参数，通过get方式请求`image.do`获取验证码图片。

这里，我用了`ddddocr`这个库来自动识别验证码，但似乎有出错的概率。不过这里也可以手动填写。

发送post请求以后，会得到一段JSON数据和新的Cookie，这是两个关键的东西。

JSON数据中，可以看到有一个token。观察Cookie，发现新增了_WEU。

## 获取课表
登陆以后，就可以把token作为请求头，同时利用Cookie进行各种登陆后的操作了。这里获取一下当前的课表。

获取课表只需向`programCourse.do`发送一个post请求，请求的参数比较复杂，新系统中有一个构建函数专门用来做这件事情。
|name|value|
|----|-----|
|querySetting|请求数据|

这里的请求数据实际上是构造了一个对象：
```JSON
{
    "data":{
        "studentCode":"学号",
        "electiveBatchCode":"学期",
        "teachingClassType":"ZY",
        "queryContent":"ZXZY:专业编号,ZXNJ:年级,"
    },
    "pageSize":"10",
    "pageNumber":"0",
    "order":"isChoose -"
}
```
|name|value|
|----|-----|
|studentCode|学号|
|electiveBatchCod|学期|
|teachingClassType|教学班类型|
|queryContent|专业和年级等信息|

data外是一些关于页面的参数，直接带上就行。data的具体内容是不固定的。阅读参数构造函数的代码，校区、过滤冲突、过滤已满以及等参数也可能被加进去。

至于queryContent，则是下拉菜单的内容，像这里就是专业和年级。

通过post请求，就可以得到一个十分详尽的课程列表的JSON数据，查询起来还是比较方便的。

## 获取学生信息
通过向`学号.do`发送post请求，同时添加登录Cookie并把token作为请求头，可以得到学生信息。比如专业和年级之类信息。这些信息可以用于查询课表。

## 使用方法
```Python
pip install -r requirements.txt
```
调用相关类即可

