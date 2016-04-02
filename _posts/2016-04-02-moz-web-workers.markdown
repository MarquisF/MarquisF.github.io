---
layout: post
title:  "mozilla web workers 改动"
date:   2rd Apr. 2016
categories: 对于MDN创建一个通用的 「异步 eval()」例子的改动
---

在学习web workers时看了MDN上的文档
https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers
里面提到了一个这样的例子用来创建通用的 「异步 eval()」

{% highlight js %}
// Syntax: asyncEval(code[, listener])

var asyncEval = (function () {

  var aListeners = [], oParser = new Worker("data:text/javascript;charset=US-ASCII,onmessage%20%3D%20function%20%28oEvent%29%20%7B%0A%09postMessage%28%7B%0A%09%09%22id%22%3A%20oEvent.data.id%2C%0A%09%09%22evaluated%22%3A%20eval%28oEvent.data.code%29%0A%09%7D%29%3B%0A%7D");

  oParser.onmessage = function (oEvent) {
    if (aListeners[oEvent.data.id]) { aListeners[oEvent.data.id](oEvent.data.evaluated); }
    delete aListeners[oEvent.data.id];
  };


  return function (sCode, fListener) {
    aListeners.push(fListener || null);
    oParser.postMessage({
      "id": aListeners.length - 1,
      "code": sCode
    });
  };

})();
{% endhighlight %}

其中构造oParser时构造器中传入的字符串参数经过decodeURIComponent处理后是这样的

{% highlight js %}
onmessage = function (oEvent) {
  postMessage({
    "id": oEvent.data.id,
    "evaluated": eval(oEvent.data.code)
  });
}
{% endhighlight %}

但显然由于闭包构建的缓存中只有一个web workers对象，所以这个asyncEval函数在处理多个传入的eval字符串时是按传入的先后顺序依次eval这些字符串的。比如
{% highlight js %}
  asyncEval( "var sum = 0; for( var i = 0; i < 10000000; i ++) {sum += i}; sum", function( sMessage ) {
    console.log( "first result: " + sMessage );
  });

  asyncEval( "var sum = 0; for( var i = 0; i < 10002; i ++) {sum += i}; sum", function( sMessage ) {
    console.log( "second result: " + sMessage );
  });
{% endhighlight %}

第二个相对节省时间简单的计算需要等第一个耗费时间的计算结束之后才能开始，我们得到的结果先是first result，然后才是second result。

为了让这些字符串的eval不相互制约，我将缓存中的aListeners改成了一个对象，并在函数执行时创建web workers对象同时将其存进aListeners缓存对象。就像这样

{% highlight js %}
var asyncEval = (function () {
  var aListeners = {};
  /*
    manyEval.js:
    onmessage = function (oEvent) {
      postMessage({
        "id": oEvent.data.id,
        "evaluated": eval(oEvent.data.code)
      });
    }
  */

  return function ( sCode, fListener ) {
    // 定义将被缓存的回调函数的id
    var callbackId = Math.ceil( 2000 * Math.random() );

    aListeners[ callbackId ] = {
      callback: fListener || null,
      worker: new Worker('manyEval.js')
    };

    aListeners[ callbackId ][ 'worker' ].onmessage = function( oEvent ) {
      if ( aListeners[ oEvent.data.id ] ) {
        aListeners[ oEvent.data.id ].callback( oEvent.data.evaluated );
      }
      delete aListeners[ oEvent.data.id ];
    };
    // 可以在这里console.log(aListeners)观察缓存对象的情况

    aListeners[ callbackId ]['worker'].postMessage({
      "id": callbackId,
      "code": sCode
    });
  };
})();
{% endhighlight %}

这样再执行上面包含两个求和运算的代码时就可以先看到second result，然后再看到first result了。


