# 从实现一个Event Emitter说起

事件驱动（Event-driven）模型是 Javascript 编程中绕不过去的概念，事件发射器 (Event Emitter) 则是事件驱动模型的核心。Event Emitter 与 class（及其 inherit）很相似的是，几乎每一个稍大的前端项目都会把它实现一遍。大多数情况下它被抽象成一个 Emitter 类，通过实例方法提供事件绑定（on/bind）、事件触发（fire/trigger）等功能。

实现一个基本的 Event Emitter，要处理的主要逻辑包括：

1. 维护每个事件对应的回调函数（listener）队列
2. 在事件被触发时逐个调用

应该说这不是一个很复杂的逻辑，对于一个熟悉事件驱动编程的 JSer 来说，实现起来轻而易举。可是为什么这里还是要说说实现一个 Event Emitter 呢，我们先从一个例子开始：

注：为了代码简洁清晰，本文中

* 所有的示例都以单例形式实现 emitter，而不是一个类
* 认为 forEach、filter 等会遍历数组的方法与手工 for 循环挨个处理效率相同，不考虑函数调用的开销

### Emmiter v0

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        (this.listeners[type] = this.listeners[type] || []).push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            index;
        if (listeners && listeners.length)
            if ((index = listeners.indexOf(listener)) >= 0)
                listeners.splice(index, 1);
    },

    fire: function (type, data) {
        var listeners = this.listeners[type];
        if (listeners && listeners.length)
            listeners.forEach(function (listener) {
                listener(data);
            });
    }
};
```

这是一份简单的实现，拿它当例子是因为其中隐藏了一个很典型的 bug。这里不卖关子，bug 的起因在于使用 `splice` 方法移除绑定，而 `splice` 会改变数组中受影响项的后继项的位置（下标），以下的代码可以触发这个bug：

```javascript
var fn1 = function () {
    // fn1 在被调用时解除事件绑定
    emitter.off('hello', fn1);
};
emitter.on('hello', fn1);
emitter.on('hello', fn2);
// 触发事件时 fn2 不会被调用
// 因为 fn1 执行的时候把自己通过 splice 从数组中移除了，fn2 在 listeners 数组中的下标就由1变成2，所以 fire 中的 forEach 会直接跳过它
emitter.fire('hello');
```

为了修复这个bug，我们给出针对性修改后的v1：

### Emmiter v1 (fix "off-on-fire" bug)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        (this.listeners[type] = this.listeners[type] || []).push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;
        if (listeners && listeners.length) {
            listeners.forEach(function (exist, i) {
                // set item null instead of remove it
                if (exist === listener) listeners[i] = null;
            });
        }
    },

    fire: function (type, data) {
        var listeners = this.listeners[type];
        if (listeners && listeners.length) {
            listeners.forEach(function (listener) {
                // check if listener not null
                listener && listener(data);
            });
        }
    }
};
```

这里我们通过将要被删除的项赋值为 `null`，避免了删除特定项对其后继项下标的影响，自然也就不会有 `fire` 时回调函数被跳过的风险。不过这个做法还不够完美，它有个副作用：每次 `off` 都会使得 `listeners` 数组中多出一个空项 (`null`)，数组的规模只会越来越大，遍历的效率自然也随之下降。

另外值得注意的是，这个版本与 v0 相比，不仅是修复了 bug，其 `off` 行为也发生了变化：单次`off` 会把所有符合条件的 `listener` 从队列中移除。而大部分情况下我们期望的可能是单次 `off` 移除单个 `listener`。鉴于该差异可以通过控制循环时的行为简单地加以调整，这里我们不做优劣区分，假设它们都是合理的。

我们将目光聚焦于解决 `null` 项导致数组规模不可逆地增大的问题：

### Emmiter v2 (fix "off-on-fire" bug & avoid null in array)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        (this.listeners[type] = this.listeners[type] || []).push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;

        if (listeners && listeners.length) {
            // create a new list
            this.listeners[type] = listeners.filter(function (exist) {
                return exist !== listener;
            });
        }
    },

    fire: function (type, data) {
        var listeners = this.listeners[type];
        if (listeners && listeners.length) {
            listeners.forEach(function (listener) {
                listener(data);
            });
        }
    }
};
```

这里我们采用了更好的方法来解决 v0 的 bug：在 `off` 时将 `listeners` 数组拷贝一份（拷贝时剔除本次应该被解除的绑定项）。虽然一次 `off` 需要对更多的项进行写操作，但它与 v1 的做法是相同的算法复杂度；另外考虑到相比 `on`/`fire`，`off`属于低频行为，引入的性能损失可忽略不计。

很轻松地解决掉第一个问题，下面我们考虑向 emitter 添加更多的特性。现有的 `fire` 只支持在触发事件时传入 `type`（事件类型）与 `data`（事件触发相关数据）两个参数，而只有 `data` 会被传递给 `listener`，为满足一个常见的需求，我们改动 emitter 以支持触发事件时传入任意数目的参数（除 `type` 外它们都会成为 `listener` 被调用时的参数）。

### Emitter v3 (more args for fire)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        (this.listeners[type] = this.listeners[type] || []).push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;
        if (listeners && listeners.length) {
            this.listeners[type] = listeners.filter(function (exist) {
                return exist !== listener;
            });
        }
    },

    fire: function (type) {
        var listeners = this.listeners[type];
        if (listeners && listeners.length) {
            // record arguments in args
            var args = Array.prototype.slice.call(arguments, 1);
            listeners.forEach(function (listener) {
                listener.apply(null, args);
            });
        }
    }
};
```

按照我们的惯例，这是个有 bug 的版本...我开玩笑的，这次并没有。不过接下来的工作更有意义，我们先看一个链接：http://jsperf.com/call-vs-apply-vs-bind-kbrain 。

好了你当然明白我的意思，这次的改进空间在于性能。为什么我们要如此在意性能呢——Emitter 是个相当基础的功能模块，在一个项目里可能有很多个模块依赖它、继承它，一次消息同步可能包含了多次的事件触发，Emitter 的方法被调用次数大概是与项目规模的平方成正比的。这么来看的话，在项目规模变大时，对 Emitter 的方法性能的优化很可能带来系统整体性能的极大提升。

继续前边的话题，为了 `fire` 时可以传入任意数量的参数，我们使用 `apply` 来调用 `listener`，但从 benchmark 中不难发现，相比 `call` 或直接调用函数本身，`apply` 的性能损失很大。

### Emitter v4 (better performance, call instead of apply)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        (this.listeners[type] = this.listeners[type] || []).push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;
        if (listeners && listeners.length) {
            this.listeners[type] = listeners.filter(function (exist) {
                return exist !== listener;
            });
        }
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type],
            args, argsNum;

        if (listeners && listeners.length) {
            args = Array.prototype.slice.call(arguments, 1);
            argsNum = args.length;

            listeners.forEach(function (listener) {
                // if less than 5 arguments, use "call" instead of "apply"
                switch (argsNum) {
                    case 0: listener(); break;
                    case 1: listener(arg1); break;
                    case 2: listener(arg1, arg2); break;
                    case 3: listener(arg1, arg2, arg3); break;
                    case 4: listener(arg1, arg2, arg3, arg4); break;
                    default: listener.apply(null, args);
                }
            });
        }
    }
};
```

直接调用函数本身或使用 `foo.call()` 的局限性在于必须明确在编码时明确知道参数的个数，而考虑到在实际应用中很少有 `fire` 时额外参数多于4个的，这里将参数数0-4的情况特殊处理，可以达到绝大部分情况下的优化。

值得一提的是，示例中我们的做法是直接调用函数（`listener`）本身，事实上一般实现中都会采用 `listener.call()`的形式，这在需要指定 `listener` 被调用时的 context 时比较有意义。

那么，再接再厉。现有的版本是不是可以继续抠一下呢？在抠细节之前，再重复一下，这里为了代码简单清晰，认为
```javascript
listeners.forEach(fn);
```
与
```javascript
for (var i = 0, l = listeners.length; i < l; i++)
    fn(listeners[i], i);
```
是等价的（实际上也几乎是等价的）。那么再看这个比较：http://jsperf.com/one-item-array-vs-fn 。

### Emitter v5 (better performance, use listener instead of [listener])

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        var listeners = this.listeners[type];

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        // listener instead of [listener]
        if (typeof listeners === 'function') {
            this.listeners[type] = [listeners, listener];
            return;
        }

        listeners.push(listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;

        if (!listeners) return;

        // listener instead of [listener]
        if (typeof listeners === 'function') {
            // use null instead of []
            if (listeners === listener) this.listeners[type] = null;
            return;
        }

        this.listeners[type] = listeners.filter(function (exist) {
            return exist !== listener;
        });
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type];

        if (!listeners) return;

        var args = Array.prototype.slice.call(arguments, 1),
            argsNum = args.length;

        // if only one listener
        if (typeof listeners === 'function') {
            switch (argsNum) {
                case 0: listeners(); break;
                case 1: listeners(arg1); break;
                case 2: listeners(arg1, arg2); break;
                case 3: listeners(arg1, arg2, arg3); break;
                case 4: listeners(arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener(); break;
                case 1: listener(arg1); break;
                case 2: listener(arg1, arg2); break;
                case 3: listener(arg1, arg2, arg3); break;
                case 4: listener(arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

本次的改动也基于一个假设：存在着相当比例的事件，只被绑定了一次（即只有一个回调函数）。在这样的前提下，如果将 `[listener]` 的存储方式改为 `listener`，代价是多一次类型判断，收益是避免了对仅包含一项内容的数组 `[listener]` 的遍历。

说真的，我觉得抠到这里已经有点丧心病狂了...不过这个手段被使用并不少，大概是实现者都觉得，反正就几行代码的事，能优化一点是一点吧。

好了，鉴于到目前为止我们的 Emitter 依然功能简陋，我们来继续完善。这次我们加入 `once` 方法，顾名思义，这是具有触发一次后自动解绑能力的绑定方法。

### Emitter v6 (add once)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        var listeners = this.listeners[type];

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (typeof listeners === 'function') {
            this.listeners[type] = [listeners, listener];
            return;
        }

        listeners.push(listener);
    },

    once: function (type, listener) {
        var _this = this;

        // wrap with a function which will "off" itself while fired
        var realListener = function () {
            listener.apply(this, arguments);
            _this.off(type, realListener);
        };

        _this.on(type, realListener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;

        if (!listeners) return;

        if (typeof listeners === 'function') {
            if (listeners === listener) this.listeners[type] = null;
            return;
        }

        this.listeners[type] = listeners.filter(function (exist) {
            return exist !== listener;
        });
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type];

        if (!listeners) return;

        var args = Array.prototype.slice.call(arguments, 1),
            argsNum = args.length;

        if (typeof listeners === 'function') {
            switch (argsNum) {
                case 0: listeners(); break;
                case 1: listeners(arg1); break;
                case 2: listeners(arg1, arg2); break;
                case 3: listeners(arg1, arg2, arg3); break;
                case 4: listeners(arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener(); break;
                case 1: listener(arg1); break;
                case 2: listener(arg1, arg2); break;
                case 3: listener(arg1, arg2, arg3); break;
                case 4: listener(arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

为了使新添加的 `once` 功能对已有流程产生尽可能小的影响，我们的做法（也是很常见的做法）是:将 `once` 需要添加的 `listener` 包装成一个一旦执行就把自己解绑的函数 `realListener`，然后将该函数通过正常的方式（`on`）绑定到给定事件上。

可见 `once` 的添加并没有造成对已有功能实现的侵入，干净无污染，想想有点小激动呢。不过新功能的使用上有个小问题：通过 `once` 绑定的 `listener` 无法手动通过 `off` 解绑，这个很好理解，因为真正放到 `listeners` 队列里的是 `realListener`，而不是 `listener`。

咋办，修呗，原有方法该改还是得改啊：

### Emitter v7 (fix once bug)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        var listeners = this.listeners[type];

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (typeof listeners === 'function')) {
            this.listeners[type] = [listeners, listener];
            return;
        }

        listeners.push(listener);
    },

    once: function (type, listener) {
        var _this = this;

        var realListener = function () {
            listener.apply(this, arguments);
            _this.off(type, realListener);
        };

        // record origin listener
        realListener._origin = listener;

        _this.on(type, realListener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;

        if (!listeners) return;

        if (typeof listeners === 'function')) {
            if (
                listeners === listener ||
                (listeners._origin && listeners._origin === listener)
            ) this.listeners[type] = null;
            return;
        }

        this.listeners[type] = listeners.filter(function (exist) {
            return (
                exist !== listener &&
                // do compare with _origin
                !(exist._origin && exist._origin === listener)
            );
        });
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type];

        if (!listeners) return;

        var args = Array.prototype.slice.call(arguments, 1),
            argsNum = args.length;

        if (typeof listeners === 'function')) {
            switch (argsNum) {
                case 0: listeners(); break;
                case 1: listeners(arg1); break;
                case 2: listeners(arg1, arg2); break;
                case 3: listeners(arg1, arg2, arg3); break;
                case 4: listeners(arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener(); break;
                case 1: listener(arg1); break;
                case 2: listener(arg1, arg2); break;
                case 3: listener(arg1, arg2, arg3); break;
                case 4: listener(arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

这种级别的问题当然难不倒我们，既然真实的 `listener` 已经被替换了，那么我们把这个信息在 `realListener` 中带上（比如本例中挂在 `realListener._origin` 上），在 `off` 查找的时候把这种情况额外考虑进来就 OK 了。

可是，还是不够好。

为什么说不够好？本例的实现隐式地引入了一个约定：`listener._origin` 不会被使用（谁跟你约定的？）；如果通过 `on` 传入的 `listener` 有一个名为 `_origin` 的property，而且值恰好是队列中的某个 `listener`，这种情况下 `off` 时对 `_origin` 的额外检查会导致不预期的解绑：

```javascript
fn._origin = fn2;
emitter.on('hello', fn);
emitter.off('hello', fn2);
```

虽然正常使用中这样的可能性极小，但还是存在的。即便把这个 `"_origin"` 用随机生成的唯一ID代替，也只是无限缩小了“撞车”的概率，并没有从根本上解决问题。

作为一个合格的工程师，这种不完美简直让人浑身难受。幸好，解决起来也不算麻烦：

### Emitter v8 (fix once bug better)

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener, orgin) {
        var listeners = this.listeners[type];

        // record handler (will be called) & origin
        listener = {
            handler: listener,
            origin: origin || listener
        };

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (typeof listeners.handler === 'function') {
            this.listeners[type] = [listeners, listener];
            return;
        }

        listeners.push(listener);
    },

    once: function (type, listener) {
        var _this = this;

        var realListener = function () {
            listener.apply(this, arguments);
            _this.off(type, listener);
        };

        // pass listener as "orgin"
        _this.on(type, realListener, listener);
    },

    off: function (type, listener) {
        var listeners = this.listeners[type],
            i;

        if (!listeners) return;

        if (typeof listeners.handler === 'function') {
            // compare with "origin"
            if (listeners.origin === listener) this.listeners[type] = null;
            return;
        }

        this.listeners[type] = listeners.filter(function (exist) {
            return exist.origin !== listener;
        });
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type];

        if (!listeners) return;

        var args = Array.prototype.slice.call(arguments, 1),
            argsNum = args.length;

        if (typeof listeners.handler === 'function') {
            switch (argsNum) {
                // call handler
                case 0: listeners.handler(); break;
                case 1: listeners.handler(arg1); break;
                case 2: listeners.handler(arg1, arg2); break;
                case 3: listeners.handler(arg1, arg2, arg3); break;
                case 4: listeners.handler(arg1, arg2, arg3, arg4); break;
                default: listeners.handler.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                // call handler
                case 0: listener.handler(); break;
                case 1: listener.handler(arg1); break;
                case 2: listener.handler(arg1, arg2); break;
                case 3: listener.handler(arg1, arg2, arg3); break;
                case 4: listener.handler(arg1, arg2, arg3, arg4); break;
                default: listener.handler.apply(null, args);
            }
        });
    }
};
```

代码已经变得有点长了，看起来都满满的成就感！

这里我们调整了 `listener` 的结构，改为一个 Object，`listener.handler` 及 `listener.origin` 分别为真实的回调函数与初始的（外部传入的）回调函数。前者用于触发事件时调用，后者用于查找时比较。通过给 `on` 方法添加一个可选参数避免了结构变化可能引起的接口变更。

可惜的是，我们再一次引入了 bug。是的，我就是说上面这段代码，它的问题变得比刚才的隐性约定还要大：`once` 包装后的 `handler` 中的自动 `off` 行为可能会把通过 `on` 绑定的同一个监听函数移除，现在的比较 `listener.origin` 的方式，并不能区分通过 `on` 或 `once` 绑定的事件监听。bug 的触发也很简单：

```javascript
emitter.on('hello', fn); // 1
emitter.once('hello', fn); // 2
emitter.fire('hello'); // 这里会因为once的自动解绑，把1处绑定的fn给清除掉
```

注意这里无论 `off` 的行为是只移除第一个符合条件的 `listener`，还是一次移除所有符合条件的 `listener`，这个 bug 都是存在的，因为通过 `on` 绑定的 `fn` 被通过 `once` 绑定 `fn` 而产生的自动解绑行为移除了。

太赞了，又有 bug 可以修了！

### Emitter v9 (fix bug)

```javascript
var emitter = {
    listeners: {},

    // add "once" flag
    on: function (type, listener, orgin, once) {
        var listeners = this.listeners[type];

        // record once
        listener = {
            handler: listener,
            origin: origin || listener,
            once: once
        };

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (typeof listeners.handler === 'function') {
            this.listeners[type] = [listeners, listener];
            return;
        }

        listeners.push(listener);
    },

    once: function (type, listener) {
        var _this = this;

        var realListener = function () {
            listener.apply(this, arguments);
            // off "once" listener by passing "once"=true
            _this.off(type, listener, true);
        };

        // pass listener as "orgin"
        _this.on(type, realListener, listener, true);
    },

    // add "once" flag
    off: function (type, listener, once) {
        var listeners = this.listeners[type],
            i;

        if (!listeners) return;

        if (typeof listeners.handler === 'function') {
            // compare once
            if (
                listeners.origin === listener &&
                !listeners.once === !once
            ) this.listeners[type] = null;
            return;
        }

        var newListeners = [];

        listeners.forEach(function (exist, i) {
            // compare once
            if (
                exist.orgin !== listener ||
                !listeners.once !== !once
            ) {
                newListeners.push(exist);
            }
        });

        this.listeners = newListeners;
    },

    fire: function (type, arg1, arg2, arg3, arg4) {
        var listeners = this.listeners[type];

        if (!listeners) return;

        var args = Array.prototype.slice.call(arguments, 1),
            argsNum = args.length;

        if (typeof listeners.handler === 'function') {
            switch (argsNum) {
                // call handler
                case 0: listeners.handler(); break;
                case 1: listeners.handler(arg1); break;
                case 2: listeners.handler(arg1, arg2); break;
                case 3: listeners.handler(arg1, arg2, arg3); break;
                case 4: listeners.handler(arg1, arg2, arg3, arg4); break;
                default: listeners.handler.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                // call handler
                case 0: listener.handler(); break;
                case 1: listener.handler(arg1); break;
                case 2: listener.handler(arg1, arg2); break;
                case 3: listener.handler(arg1, arg2, arg3); break;
                case 4: listener.handler(arg1, arg2, arg3, arg4); break;
                default: listener.handler.apply(null, args);
            }
        });
    }
};
```

为了区分 `on` 绑定的 `listener` 与 `once` 绑定的 `listener`，我们给 `listener` 添加了一个 `once` 标志位，另外在 `off` 时增加一个可选参数 `once`，通过比较这两者实现区分。幸运的是，`off` 新增的参数也不会影响它对外的表现，对于外部调用来说，只需要关心 `off` 原有的两个参数即可，这样的修复可以看成是无副作用的。

当然，得益于 `listener.once` 的存在，`listener.origin` 其实是可以省去的，将自动解绑的逻辑转移到 `fire` 中，这样甚至可以避免 `off` 新增的可选参数。这么做的话结构、接口更简单，不过代价是 `once` 的相关逻辑侵入了 `fire` 的实现，取舍看个人喜好吧，这里不赘述。

到目前为止，我们实现了一个具有基本的 `on`, `off`, `fire`, `once` 功能，没有明显 bug 的 Event Emitter，覆盖了平时绝大多数的需求。不过使用方的需求是不可穷尽的，稍加考察下市面上的已有 Event Emitter 模块，不难发现，有人做了（有需求），而我们还没有做的还有很多，比如：

* 指定回调函数被触发时的 context

* 支持空格分隔的多个事件一次绑定/触发（甚至 `"all"`）

    ```javascript
    emitter.on('eventA eventB eventC', function () {
        // ...
    });
    emitter.on('all', function () {
        // ...
    });
    ```

* 支持通配符

    ```javascript
    emitter.on('*-update', function () {
        // ...
    });
    ```

* ...

当然，这里不会把这些一一实现，我们就说到这。成熟的 Event Emitter 实现有很多，比如 [Node.js 的 events 模块](https://github.com/joyent/node/blob/master/lib/events.js)，[EventEmitter](https://github.com/Olical/EventEmitter)，[EventEmitter2](https://github.com/asyncly/EventEmitter2) 以及 [EventEmitter3](https://github.com/primus/eventemitter3)，如果去看一下他们不算很长的源代码，会发现本文借鉴了很多它们的实现，而几乎并没有什么超越之处。

所以这里的实现并不是目的，相反，以上所有都只是铺垫，铺垫有点长，下面是精简后的正文。

从实现一个 Event Emitter 说起，我的几点感想：

1. 性能优化从高频行为入手，越高频，越有价值
2. 复杂度随特性增加呈指数级增长，添加新特性的代价是给已有的及新增的特性引入风险
3. 由 2 可知，越是基础的工具，越要足够简单
4. 由 3 可知，基础设施追求大而全是自寻死路

最后，本文所有代码都用于针对性示例，不适用于生产环境，出 bug 概不负责。
