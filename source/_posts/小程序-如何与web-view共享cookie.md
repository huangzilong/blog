---
title: 小程序| 如何与web-view共享cookie
date: 2018-11-26 21:04:58
tags:
---

> 目前我司准备在小程序里通过web-view嵌套h5,希望在小程序调用api(wx.request())，然后server端通过api的header设置cookie。因为cookie均是通过http设置或者携带，以为这样，就可以在web-view里通过http请求(xhr或者fetch)带上小程序请求的api中设置的cookie。事实证明`too young too naive `。

## 小程序没法设置cookie到web-view中

首先，我们启一个server，这个server目的在于返回一个get请求，并设置cookie, 接收一个post请求，用于查看http是否带上了cookie

``` JavaScript
// app.js

const Koa = require('koa');
const cors = require('koa2-cors');
const Router = require('koa-router');
// cors 用于设置 http2 中的 CORS `Access-Control-Allow-Origin: *`
const app = new Koa();
const router = new Router();

router.get('/', (ctx, next) => {
  ctx.cookies.set('mini-program', new Date().toISOString());
  ctx.body = 'mini-program' + new Date().toISOString();
});

router.post('/', (ctx, next) => {
  console.log('cookie:', ctx.request.header.cookie)
  ctx.body = 'success';
});

app.use(router.routes())

app.use(cors({
  credentials: true, // 用于跨域发送cookie，false cookie会被浏览器拦截
}));

app.listen(8082);
```

然后，我们仅仅看小程序请求，会不会带上cookie。

```JavaScript
beforeMount: function() {
  wx.request({
    url: 'http://127.0.0.1:8082',
    method: 'GET',
    success: function() {
      wx.request({
        url: 'http://127.0.0.1:8082',
        method: 'POST',
      })
    },
  })
}
```

我们可以看到，http get 请求 response 的header里有一个Set-Cookie的字段，而post请求中，并没有把cookie带上。

那在webview里会怎样呢？

使用[http-server](https://www.npmjs.com/package/http-server)在80端口启一个很简单的h5页面，该页面作用是发起以下请求

```html
<!DOCTYPE html>
<html>
<head>
<title></title>
</head>
<body>
<script>
  var xhr = new XMLHttpRequest();
  xhr.open('POSt', 'http://127.0.0.1:8082');
  xhr.withCredentials = true; // 只有设置了这个，才能在跨域的情形下，带上cookie
  xhr.send();
</script>
</body>
</html>
```

然后在小程序中的web-view中使用该url

```wxml
<web-view wx:if="{{showWebView}}" src="http://127.0.0.1"></view-view>
```

确保发起了get请求后，再把 `showWebView`设置为`true`

可以看到，在web-view里的请求，header也没有cookie。

## web-view如何使用小程序中的cookie

wx.request会返回http header，取到`Set-Cookie`这个字段，然后使用`encodeURIComponent`编码一下，通过url传给web-view,在web-view中通过document.cookie赋值。

## 小程序中如何模拟cookie

可以参考 [一行代码让微信小程序支持 cookie](https://github.com/charleslo1/weapp-cookie)

### 为什么小程序与web-view与不能共享cookie

小程序的wx.request()是通过jsCore调用系统原生api发起的请求，即便header里带有`set-cookie`，也不会在web-view对应的'浏览器'中设置cookie，而是由原生应用来处理这个header中的`set-cookie`，至于怎么操作，要看原生应用了。这个可以参考[如何处理 iOS 原生网络请求中的 cookie ？](https://www.jianshu.com/p/d144bd7226b7)。

比较有趣的是，小程序中的web-view和微信中直接打开的h5，因为用的是同一个浏览器内核，所以，它们的cookie、storage是可以共享的。


---

[原文链接](https://github.com/huangzilong/evolution/)

