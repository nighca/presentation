### 一起实现一个Event Emitter

### v1

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
                if (exist === listener) listeners.splice(i, 1);
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

### v1存在bug

* splice会改变数组条目的下标，导致遍历出问题（条目被跳过）

### v2 fix "off-on-fire" bug

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

### v2会向数组中引入null，数组规模扩大

### v3 fix "off-on-fire" bug & avoid null in array

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

### 添加feature

* 触发事件时可以传入任意数目的参数

### v4 more args for fire

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

### 性能优化的点？

* http://jsperf.com/call-vs-apply-vs-bind-kbrain

### v5 better performance (call instead of apply)

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
                    case 0: listener.call(null); break;
                    case 1: listener.call(null, arg1); break;
                    case 2: listener.call(null, arg1, arg2); break;
                    case 3: listener.call(null, arg1, arg2, arg3); break;
                    case 4: listener.call(null, arg1, arg2, arg3, arg4); break;
                    default: listener.apply(null, args);
                }
            });
        }
    }
};
```

### 再优化一点？

```javascript
listeners.forEach(fn);
```

```javascript
for (var i = 0, l = listeners.length; i < l; i++)
	fn(listeners[i], i);
```

### v6 better performance (use listener instead of [listener])

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
        if (!Array.isArray(listeners)) {
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
        if (!Array.isArray(listeners)) {
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
        if (!Array.isArray(listeners)) {
            switch (argsNum) {
                case 0: listeners.call(null); break;
                case 1: listeners.call(null, arg1); break;
                case 2: listeners.call(null, arg1, arg2); break;
                case 3: listeners.call(null, arg1, arg2, arg3); break;
                case 4: listeners.call(null, arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener.call(null); break;
                case 1: listener.call(null, arg1); break;
                case 2: listener.call(null, arg1, arg2); break;
                case 3: listener.call(null, arg1, arg2, arg3); break;
                case 4: listener.call(null, arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

### 添加feature

* once方法，触发一次后自动解绑

### v7 add once

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        var listeners = this.listeners[type];

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
            switch (argsNum) {
                case 0: listeners.call(null); break;
                case 1: listeners.call(null, arg1); break;
                case 2: listeners.call(null, arg1, arg2); break;
                case 3: listeners.call(null, arg1, arg2, arg3); break;
                case 4: listeners.call(null, arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener.call(null); break;
                case 1: listener.call(null, arg1); break;
                case 2: listener.call(null, arg1, arg2); break;
                case 3: listener.call(null, arg1, arg2, arg3); break;
                case 4: listener.call(null, arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

### 存在的bug

* once绑定的事件无法手动通过off解绑

### v8 fix once bug

```javascript
var emitter = {
    listeners: {},

    on: function (type, listener) {
        var listeners = this.listeners[type];

        if (!listeners) {
            this.listeners[type] = listener;
            return;
        }

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
            switch (argsNum) {
                case 0: listeners.call(null); break;
                case 1: listeners.call(null, arg1); break;
                case 2: listeners.call(null, arg1, arg2); break;
                case 3: listeners.call(null, arg1, arg2, arg3); break;
                case 4: listeners.call(null, arg1, arg2, arg3, arg4); break;
                default: listeners.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                case 0: listener.call(null); break;
                case 1: listener.call(null, arg1); break;
                case 2: listener.call(null, arg1, arg2); break;
                case 3: listener.call(null, arg1, arg2, arg3); break;
                case 4: listener.call(null, arg1, arg2, arg3, arg4); break;
                default: listener.apply(null, args);
            }
        });
    }
};
```

### 还是不够好

* 基于\_origin不被使用的约定（谁跟你约定的。。），如果调用方绑定时传入的函数自带了\_origin属性呢

```javascript
fn._origin = fn2;
emitter.on('hello', fn);
emitter.off('hello', fn2);
```

### v9 fix once bug better

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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
            switch (argsNum) {
                // call handler
                case 0: listeners.handler.call(null); break;
                case 1: listeners.handler.call(null, arg1); break;
                case 2: listeners.handler.call(null, arg1, arg2); break;
                case 3: listeners.handler.call(null, arg1, arg2, arg3); break;
                case 4: listeners.handler.call(null, arg1, arg2, arg3, arg4); break;
                default: listeners.handler.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                // call handler
                case 0: listener.handler.call(null); break;
                case 1: listener.handler.call(null, arg1); break;
                case 2: listener.handler.call(null, arg1, arg2); break;
                case 3: listener.handler.call(null, arg1, arg2, arg3); break;
                case 4: listener.handler.call(null, arg1, arg2, arg3, arg4); break;
                default: listener.handler.apply(null, args);
            }
        });
    }
};
```

### 又引入新的bug了。。

* once中的off行为可能会把通过on绑定的同一个监听函数干掉

```javascript
emitter.once('hello', fn);
emitter.on('hello', fn);
emitter.fire('hello');
```

### v10 fix bug

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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
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

        if (!Array.isArray(listeners)) {
            switch (argsNum) {
                // call handler
                case 0: listeners.handler.call(null); break;
                case 1: listeners.handler.call(null, arg1); break;
                case 2: listeners.handler.call(null, arg1, arg2); break;
                case 3: listeners.handler.call(null, arg1, arg2, arg3); break;
                case 4: listeners.handler.call(null, arg1, arg2, arg3, arg4); break;
                default: listeners.handler.apply(null, args);
            }
            return;
        }

        listeners.forEach(function (listener) {
            switch (argsNum) {
                // call handler
                case 0: listener.handler.call(null); break;
                case 1: listener.handler.call(null, arg1); break;
                case 2: listener.handler.call(null, arg1, arg2); break;
                case 3: listener.handler.call(null, arg1, arg2, arg3); break;
                case 4: listener.handler.call(null, arg1, arg2, arg3, arg4); break;
                default: listener.handler.apply(null, args);
            }
        });
    }
};
```

### 更多的特性？

* context

* 通配符

* 。。。

### 总结一下。。

* 高频行为是最好的性能优化切入点，极小的改进可能带来很大的收益

* 复杂度随特性增加呈指数级增长，添加新特性的代价是给已有特性引入风险

* 测试用例永远不能覆盖足够多的情况，越是基础的工具，越要足够简单

* 追求大而全是自寻死路