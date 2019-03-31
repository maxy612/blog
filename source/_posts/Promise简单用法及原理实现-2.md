---
title: Promise简单用法及原理实现(2)
date: 2017-10-18 21:45:42
tags: ["Promise"]
summary: 本篇承接上篇未完成的promise部分，主要介绍promise的实现和在ES7中的发展。
category: "学习"
---

1. Promise的实现
2. Promise的延伸

## Promise的实现

之前，无论在笔试或面试中，我都遇到过让手写promise的题目。当时只知道怎么用，手写还是比较x疼的。前后看了四五版的promise的实现，对于如何手写promise，也逐渐有了自己的思路，下面让我们一步步地实现一个promise。

首先，理一下promise都有那些方法。
- then 传入promise实例在resolve后和reject后的方法，同时返回一个新的promise实例
- catch 传入一个onRejected的方法，当promise实例的status变为rejected时执行该方法，并返回一个新的promise实例
- resolve 可以传入一个值或promise实例，返回一个状态为resolved的promise实例。
- reject 可以传入一个Error实例或值，返回一个状态为rejected的promise实例。
- race 传入一个promise实例的数组，返回一个状态为resolved或rejected的promise实例。
- all 传入一个promise实例的数据，返回一个状态为resolved或rejected的promise实例。

方法已经列出了，下面根据方法来逐步实现promise。

- - - - - - - - - - - - - -

Promise是一个构造函数，让我们用ES6的语法来写一个MyPromise类。
{% codeblock lang:javascript %}
class MyPromise {
  constructor(fun) {
    // 设置初始的状态和值
    this.status = 'pending';
    this.data = undefined; // 用于在执行resolve时调用回调函数，作为回调函数的参数传入
    this.successFns = []; // 在resolve后执行回调
    this.failFns = []; // 在reject后执行回调

    function resolve(value) {
      // 当异步任务执行成功后，执行resolve
    }

    function reject(reason) {
      // 异步任务执行出错时，执行reject
    }

    try {
      // 执行异步任务，传入成功和失败的处理函数
      fun(resolve.bind(this), reject.bind(this));
    } catch (e) {
      // 如果异步任务执行出错，直接按失败处理
      reject.bind(this, e);
    }
  }
}
{% endcodeblock %}

上边的代码初始化了promise的构造函数，其中resolve用在当传入的fun异步任务执行成功后，改变promise实例的status为resolved并执行成功后的回调函数;reject用于在当传入的fun异步任务失败或执行出错后，改变promise实例的status为rejected并执行失败后的回调；data用于维护异步任务执行成功或失败后传入的值。

- - - - - - - - - - - -

接下来我们实现resolve, reject

{% codeblock lang:javascript %}
function resolve(value) {
  let _self = this;
  // 只有当status为pending时，才能改变status的状态并执行回调。
  if (_self.status === 'pending') {
    _self.status = 'resolved';
    _self.data = value;

    // promise的回调异步执行，所以此处用setTimeout异步执行成功后的回调函数。
    setTimeout(() => {
        for (var i = 0, len = _self.successFns.length; i < len; i++) {
          _self.successFns[i](_self.data);
        }
    })
  }
}

// 同resolve，reject也是同样的实现

function reject(reason) {
  var _self = this;
  if (_self.status === 'pending') {
    _self.status = 'rejected';
    _self.data = reason;

    setTimeout(() => {
      for (var i = 0, len = _self.failFns.length; i < len; i++) {
        _self.failFns[i](_self.data);
      }
    })
  }
}
{% endcodeblock %}

- - - - - - - - - - - - - -- - -

其实，promise的最核心的方法是then, 实现了它，其它的方法就容易地多。

then是所有promise实例都会用到的方法，所以应该把它定义在原型上。当执行then时，首先需要对传入的参数进行处理。如果传入的参数类型不是function，要设置一个默认的处理函数。
处理参数以后，需要判断当前promise实例的status。

如果promise实例状态(status)为pending，那我们需要返回一个新的promise实例，在返回的promise实例内，将传入的函数加入到实例的successFns和failFns回调队列上，待当前的promise实例的status改变时，通过回调队列执行返回的新的promise的异步任务并设置status；

如果promise实例状态(status)为resolved时，返回一个新的promise实例，在返回的promise实例中执行传入的onResolved函数并获得其返回值，如果返回值是一个promise实例，直接执行返回的promise实例的then方法，传入resolve和reject，设置promise实例的状态。如果返回值为其它类型，直接将返回值作为参数传入返回实例的resolve中；如果在执行onResolved过程中出错，则将错误信息传入reject中，设置返回promise实例的状态。

如果promise实例状态(status)为rejected时，和status为resolved时处理方式大体相同，不再过多赘述。

{% codeblock lang:javascript %}
then(onResolved, onRejected) {
  onResolved = typeof onResolved === 'function' ? onResolved : function (val) {return val;};
  onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw Error(e);};

  let _self = this;

  if (_self.status === 'pending') {
    return new MyPromise(function(resolve, reject) {
      _self.successFns.push(function () {
          try {
            var x = onResolved(_self.data);
            if (x instanceof MyPromise) {
              x.then(resolve, reject);
            }
            resolve(x);
          } catch(e) {
            reject(e);
          }
      })

      _self.failFns.push(function () {
        try {
          var x = onRejected(_self.data);
          if (x instanceof MyPromise) {
            x.then(resolve, reject);
          }
          resolve(x);
        } catch(e) {
          reject(e);
        }
      })
    })
  }

  if (_self.status === 'resolved') {
    return new MyPromise(function (resolve, reject) {
      try {
        var x = onResolved(_self.data);
        if (x instanceof MyPromise) {
          x.then(resolve, reject);
        }
        resolve(x);
      } catch(e) {
        reject(e);
      }      
    })
  }

  if (_self.status === 'rejected') {
    return new MyPromise(function (resolve, reject) {
      try {
        var x = onRejected(_self.data);
        if (x instanceof MyPromise) {
          x.then(resolve, reject);
        }
        resolve(x);
      } catch(e) {
        reject(e);
      }
    })
  }
}
{% endcodeblock %}

- - - - - - - - - - - - - - - - - -

最核心的部分then实现了，下面我们来实现catch。

catch专门用于处理promise实例为rejected时的情况，也是借用了then方法，相当于then(undefined, onRejected)。

{% codeblock lang:javascript %}
catch(onRejected) {
  return this.then(undefined, onRejected);
}
{% endcodeblock %}

- - - - - - - - - - - - - -

下面，我们来实现resolve, reject, race, all这四个静态方法。

{% codeblock lang:javascript %}
static resolve(val) {
  // 类型判断
  if (val instanceof MyPromise) {
    return val;
  }

  return new MyPromise(function (resolve, reject) {
    resolve(val);
  })
}

static reject(reason) {
  return new MyPromise(function (resolve, reject) {
    reject(reason);
  })
}

static race(promiseArr) {
  if (!Array.isArray(promiseArr)) {
    return MyPromise.reject('arguments must be Array');
  }

  if (promiseArr.length === 0) {
    return MyPromise.resolve([]);
  }

  return new MyPromise(function(resolve, reject) {
    promiseArr.forEach(val => {
      MyPromise
        .resolve(val)
        .then(val => {
          resolve(val);
        })
        .catch(e => reject(e));
    })    
  })
}

static all(promiseArr) {
  return new MyPromise(function(resolve, reject) {
    var results = [],
        remaining = promiseArr.length;
    for (var i = 0, len = promiseArr.length; i < len; i++) {
      MyPromise
        .resolve(promiseArr[i])
        .then(val => {
          remaining--;
          results[i] = val;
          if (remaining === 0) {
            return resolve(results);
          }
        })
        .catch(e => {
          return reject(e);
        })
    }
  })
}
{% endcodeblock %}

大致就是这么多的实例方法和静态方法了，现在贴一份完整的promise代码：

{% codeblock lang:javascript %}
class MyPromise {
  constructor(fun) {
    // 设置初始的状态和值
    this.status = 'pending';
    this.data = undefined; // 用于在执行resolve时调用回调函数，作为回调函数的参数传入
    this.successFns = []; // 在resolve后执行回调
    this.failFns = []; // 在reject后执行回调

    function resolve(value) {
      // 当异步任务执行成功后，执行resolve
      let _self = this;
      // 只有当status为pending时，才能改变status的状态并执行回调。
      if (_self.status === 'pending') {
        _self.status = 'resolved';
        _self.data = value;

        // promise的回调异步执行，所以此处用setTimeout异步执行成功后的回调函数。
        setTimeout(() => {
            for (var i = 0, len = _self.successFns.length; i < len; i++) {
              _self.successFns[i](_self.data);
            }
        })
      }      
    }

    function reject(reason) {
      // 异步任务执行出错时，执行reject
      var _self = this;
      if (_self.status === 'pending') {
        _self.status = 'rejected';
        _self.data = reason;

        setTimeout(() => {
          for (var i = 0, len = _self.failFns.length; i < len; i++) {
            _self.failFns[i](_self.data);
          }
        })
      }      
    }

    try {
      // 执行异步任务，传入成功和失败的处理函数
      fun(resolve.bind(this), reject.bind(this));
    } catch (e) {
      // 如果异步任务执行出错，直接按失败处理
      reject.bind(this, e);
    }
  }

  then(onResolved, onRejected) {
    onResolved = typeof onResolved === 'function' ? onResolved : function (val) {return val;};
    onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw Error(e);};

    let _self = this;

    if (_self.status === 'pending') {
      return new MyPromise(function(resolve, reject) {
        _self.successFns.push(function () {
            try {
              var x = onResolved(_self.data);
              if (x instanceof MyPromise) {
                x.then(resolve, reject);
              }
              resolve(x);
            } catch(e) {
              reject(e);
            }
        })

        _self.failFns.push(function () {
          try {
            var x = onRejected(_self.data);
            if (x instanceof MyPromise) {
              x.then(resolve, reject);
            }
            resolve(x);
          } catch(e) {
            reject(e);
          }
        })
      })
    }

    if (_self.status === 'resolved') {
      return new MyPromise(function (resolve, reject) {
        try {
          var x = onResolved(_self.data);
          if (x instanceof MyPromise) {
            x.then(resolve, reject);
          }
          resolve(x);
        } catch(e) {
          reject(e);
        }      
      })
    }

    if (_self.status === 'rejected') {
      return new MyPromise(function (resolve, reject) {
        try {
          var x = onRejected(_self.data);
          if (x instanceof MyPromise) {
            x.then(resolve, reject);
          }
          resolve(x);
        } catch(e) {
          reject(e);
        }
      })
    }
  }

  catch(onRejected) {
    return this.then(undefined, onRejected);
  }

  static resolve(val) {
    // 类型判断
    if (val instanceof MyPromise) {
      return val;
    }

    return new MyPromise(function (resolve, reject) {
      resolve(val);
    })
  }

  static reject(reason) {
    return new MyPromise(function (resolve, reject) {
      reject(reason);
    })
  }

  static race(promiseArr) {
    if (!Array.isArray(promiseArr)) {
      return MyPromise.reject('arguments must be Array');
    }

    if (promiseArr.length === 0) {
      return MyPromise.resolve([]);
    }

    return new MyPromise(function(resolve, reject) {
      promiseArr.forEach(val => {
        MyPromise
          .resolve(val)
          .then(val => {
            resolve(val);
          })
          .catch(e => reject(e));
      })    
    })
  }

  static all(promiseArr) {
    return new MyPromise(function(resolve, reject) {
      var results = [],
          remaining = promiseArr.length;
      for (var i = 0, len = promiseArr.length; i < len; i++) {
        MyPromise
          .resolve(promiseArr[i])
          .then(val => {
            remaining--;
            results[i] = val;
            if (remaining === 0) {
              return resolve(results);
            }
          })
          .catch(e => {
            return reject(e);
          })
      }
    })
  }
}
{% endcodeblock %}

私下测试了下代码，大致没什么问题了。其实，在这里，有一点需要特别注意：
在使用promise时，有一系列的链式操作。

{% codeblock lang:javascript %}
var a = new Promise((resolve, reject) => {
  resolve(4);
})

a.then()
 .then()
 .then()
 .then(val => {
   console.log(val); // 4
   console.log(vale); // error
 })
 .catch(e => {
    console.log('err: ', e); // err: vale is not defined;
 })

{% endcodeblock %}

在这里，好像promise实例的值能够在then中自由传递，catch能够捕获到之前所有then出现的错误。在这里，涉及到了值的透传。为什么能够这样呢？奥秘就在then原型方法中。

{% codeblock lang:javascript %}
onResolved = typeof onResolved === 'function' ? onResolved : function (val) {return val;};
onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw Error(e);};
{% endcodeblock %}

如果传入的值不是function，则为onResolved和onRejected设置默认的函数，这样无论状态是resolved还是rejected，都能够通过then或者catch返回一个新的promise实例并承接之前的状态。

promise的实现就先说到这里了。

## promise的延伸

在ES7中，关于同步和异步的问题，又出来了async 和 await用以解决异步的问题。
{% codeblock lang:javascript %}
var p1 = new Promise((resolve, reject) => {
	setTimeout(() => {
		resolve(5);
	}, 2000)
})

var p2 = new Promise((resolve, reject) => {
	setTimeout(() => {
		resolve(10);
	}, 3000)
})

var fun = async function () {
	var a = await p1;
	var b = await p2;
	console.log(a, b);
	return {a, b}
}

fun().then(res => {
	console.log(res);
})
{% endcodeblock %}

通过async和await的配合，我们将异步请求的过程改写成了类似同步编码的形式，更符合我们的思维习惯。

今天就讲到这里了，以后想起来什么再补充好了。bye......
