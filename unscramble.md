# cookie—parser代码解读

## 一  代码简介

- 名称：cookie—parser
- 功能：通过req.cookies取到传过来的cookie，并把它们转成对象
- 仓库包含文件及其作用：
>1. index.js 文件是项目主文件，也是项目的入口文件，其中暴露的函数作为第三方模块引用
>2. .gitignore 文件用来排除不必要的项目文件，或者敏感的信息文件
>3. LICENSE 文件统一用 MIT 共享协议
>4. README.md 文件用来对项目做描述和说明
>5. package.json 文件用于存储工程的元数据，描述项目的依赖项，类似配置文件
>6. test 文件夹其中的cookieParser.js文件可以对index.js文件进行测试使用

## 二  代码解读（index.js文件）

- 语法语句：
>1. for循环语句
>2. if条件语句

- 代码内容：
>1. 引用模块
```
var cookie = require('cookie')
var signature = require('cookie-signature')
```
>2. 暴露模块接口
```
module.exports = cookieParser
module.exports.JSONCookie = JSONCookie
module.exports.JSONCookies = JSONCookies
module.exports.signedCookie = signedCookie
module.exports.signedCookies = signedCookies

```
>3. cookieParser函数
```
function cookieParser (secret, options) {
  return function cookieParser (req, res, next) {
    if (req.cookies) {
      return next()
    }                         
   //利用回调函数判断是否请求到了cookie，如果已经有cookie对象，则退出中间件继续运行

    var cookies = req.headers.cookie          //从headers中取cookie
    var secrets = !secret || Array.isArray(secret)
      ? (secret || [])
      : [secret]

    req.secret = secrets[0]
    req.cookies = Object.create(null)       //创建一个对象，解析后的且未使用签名的cookie保存在req.cookies中
    req.signedCookies = Object.create(null)          

    // no cookies
    if (!cookies) {
      return next()
    }                                     //没有获取到cookie的情况下，退出中间件继续运行

    req.cookies = cookie.parse(cookies, options)      //调用cookie的parse方便把cookie字符串转成cookies对象

    // parse signed cookies
    if (secrets.length !== 0) {
      req.signedCookies = signedCookies(req.cookies, secrets)
      req.signedCookies = JSONCookies(req.signedCookies)       // JSON字符序列转化为JSON对象
    }                                              
    //判断是否为singed cookie如果是，则去掉签名，同时删除req.cookies中对应的property，将这些去掉签名的cookie组成一个对象，保存在req.signedCookies中
    // parse JSON cookies
    req.cookies = JSONCookies(req.cookies)            //把req.cookies转化为对象

    next()
  }
}
```
>4. JSONCookie函数 解析JSON字符序列
```
function JSONCookie (str) {
  if (typeof str !== 'string' || str.substr(0, 2) !== 'j:') {
    return undefined
  }                       //判断是否为JSON字符序列，如果不是返回undefined

  try {
    return JSON.parse(str.slice(2))
  } catch (err) {
    return undefined
  }                     //解析JSON字符序列
}
```
>5. JSONCookies函数 接续cookie中的JSON字符序列
```
function JSONCookies (obj) {
  var cookies = Object.keys(obj)      //获取obj对象的property
  var key
  var val

  for (var i = 0; i < cookies.length; i++) {
    key = cookies[i]
    val = JSONCookie(obj[key])

    if (val) {
      obj[key] = val
    }                   //如果是JSON字符序列则保存
  }

  return obj
}
```
>6. signedCookie函数
```
function signedCookie (str, secret) {
  if (typeof str !== 'string') {
    return undefined
  }

  if (str.substr(0, 2) !== 's:') {
    return str
  }                                   //当发现含有s:开始则是签名过的cookie，这时就用了signature.unsign解签
                                    
  var secrets = !secret || Array.isArray(secret)
    ? (secret || [])
    : [secret]                         //判断是否添加了签名，如果添加了签名则去掉签名，否则返回原字符串

  for (var i = 0; i < secrets.length; i++) {
    var val = signature.unsign(str.slice(2), secrets[i])

    if (val !== false) {
      return val
    }
  }

  return false
}
```
>7. signedCookies函数
```
function signedCookies (obj, secret) {            //当客户端发送cookie时
  var cookies = Object.keys(obj)        //获取obj对象的property
  var dec
  var key
  var ret = Object.create(null)         //创建返回对象
  var val

  for (var i = 0; i < cookies.length; i++) {
    key = cookies[i]
    val = obj[key]
    dec = signedCookie(val, secret)

    if (val !== dec) {
      ret[key] = dec
      delete obj[key]
    }
  }                              //判断是否是去掉签名后的value，如果是保存该值到ret中同时删除obj中的相应property

  return ret
}
```
>8. 解析后的unsigned cookie保存在req.cookies中，而解析后的signed cookie只保存在req.signedCookies中

## 三  代码测试
- 运行时的错误
1. 没有权限：利用以前编写的修改权限js文件04-my-chomd.js将权限修改成777
![image](https://github.com/caoqiann/cookie-parser/blob/master/error1.png)
2. 文件目录等未找到：在index.js首行添加#!/usr/bin/node
![image](https://github.com/caoqiann/cookie-parser/blob/master/error2.png)
3. 找不到cookie模块：利用npm install安装模块
![image](https://github.com/caoqiann/cookie-parser/blob/master/error3.png)

- 运行成功(没有return undefined和false)
![image](https://github.com/caoqiann/cookie-parser/blob/master/finish.png)
