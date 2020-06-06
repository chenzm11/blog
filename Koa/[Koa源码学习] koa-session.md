# [Koa源码学习] koa-session

## 前言

在`Koa`应用程序中，可以通过`koa-session`模块，提供对`session`的支持。那么接下来，我们就来看看其内部是如何实现的。

## session

在引入`koa-session`模块后，就可以创建`session`对应的中间件，其代码如下所示：

```js
/* koa-session/index.js */
module.exports = function(opts, app) {
  // ...
  // 格式化配置参数
  opts = formatOpts(opts);
  // 在app.context上添加session相关的属性
  extendContext(app.context, opts);

  // 返回session中间件
  return async function session(ctx, next) {
    // ...
  };
};
```

在调用该方法时，首先会执行`formatOpts`和`extendContext`方法，其代码如下所示：

```js
/* koa-session/index.js */
function formatOpts(opts) {
  opts = opts || {};
  // session对应的cookie名称
  // key
  opts.key = opts.key || 'koa.sess';

  // back-compat maxage
  if (!('maxAge' in opts)) opts.maxAge = opts.maxage;

  // defaults
  if (opts.overwrite == null) opts.overwrite = true;
  if (opts.httpOnly == null) opts.httpOnly = true;
  // delete null sameSite config
  if (opts.sameSite == null) delete opts.sameSite;
  if (opts.signed == null) opts.signed = true;
  if (opts.autoCommit == null) opts.autoCommit = true;

  // cookie模式下的编码方法
  // setup encoding/decoding
  if (typeof opts.encode !== 'function') {
    opts.encode = util.encode;
  }
  if (typeof opts.decode !== 'function') {
    opts.decode = util.decode;
  }

  // store、externalKey、ContextStore、opts.genid都用于store模式
  // externalKey可以自定义sessionId的存储方式
  const store = opts.store;
  // ...

  const externalKey = opts.externalKey;
  // ...

  const ContextStore = opts.ContextStore;
  // ...

  if (!opts.genid) {
    if (opts.prefix) opts.genid = () => `${opts.prefix}${uuid()}`;
    else opts.genid = uuid;
  }

  return opts;
}

function extendContext(context, opts) {
  if (context.hasOwnProperty(CONTEXT_SESSION)) {
    return;
  }
  // 给app.context添加额外的属性
  Object.defineProperties(context, {
    [CONTEXT_SESSION]: {
      get() {
        // 每个请求上下文都会创建一个ContextSession实例
        if (this[_CONTEXT_SESSION]) return this[_CONTEXT_SESSION];
        this[_CONTEXT_SESSION] = new ContextSession(this, opts);
        return this[_CONTEXT_SESSION];
      },
    },
    session: {
      get() {
        return this[CONTEXT_SESSION].get();
      },
      set(val) {
        this[CONTEXT_SESSION].set(val);
      },
      configurable: true,
    },
    sessionOptions: {
      get() {
        return this[CONTEXT_SESSION].opts;
      },
    },
  });
}
```

可以看到，上面两个方法主要用来做一些初始化的工作，在`formatOpts`方法中，针对`cookie`模式，提供了`opts.encode`和`opts.decode`，而另外的`store`、`externalKey`、`ContextStore`、`opts.genid`都用于`store`模式，`extendContext`方法主要是在`app.context`上添加`session`相关的属性，因为每个请求对应的上下文对象都是`app.context`的实例，所以，定义到原生上后，在中间件中就可以很方便的通过`ctx`访问这里定义的内容了。

完成初始化工作后，该方法就返回一个`session`中间件，在收到来自客户端的请求时，就会执行这个中间件，对`session`进行处理。需要注意的是，在`koa-session`中，存在两种使用方式，一种是`cookie`模式，另一种是`store`模式，那么接下来，我们就分别看看它们是如何工作的。

## cookie模式

默认情况下，`koa-session`使用的是`cookie`模式，也就是说，所有数据都通过`cookie`存放在客户端，服务器不保存任何数据。我们首先来看看`session`对应的中间件，其代码如下所示：

```js
/* koa-session/index.js */
module.exports = function(opts, app) {
  // ...
  return async function session(ctx, next) {
    const sess = ctx[CONTEXT_SESSION];
    // 对于cookie模式，这里不会执行
    if (sess.store) await sess.initFromExternal();
    try {
      await next();
    } catch (err) {
      throw err;
    } finally {
      if (opts.autoCommit) {
        await sess.commit();
      }
    }
  };
};
```

在执行`session`中间件时，第一步就是获取`ctx[CONTEXT_SESSION]`，结合上面的`extendContext`方法，所以这里会创建一个`ContextSession`的实例，它就是本次请求的`sessionCtx`，用来操作`session`的接口，其构造函数如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  constructor(ctx, opts) {
    this.ctx = ctx;
    this.app = ctx.app;
    this.opts = Object.assign({}, opts);
    // 对于cookie模式，this.store为undefined
    this.store = this.opts.ContextStore ? new this.opts.ContextStore(ctx) : this.opts.store;
  }

  get() {
    // 获取：ctx.session[name]
    // 设置：ctx.session[name]=value
    // ...
  }

  set(val) {
    // 整体设置ctx.session
    // ...
  }
}
```

回到上面的`session`中间件中，对于`cookie`模式来说，此时就直接调用`next`方法，执行下一个中间件。

由于在`extendContext`方法中，我们在`app.context`上定义了一个名为`session`的访问器属性，而它对应的`get`、`set`访问器，刚好指向了`ContextSession`实例的`get`、`set`方法，所以在接下来的中间件中，就可以直接通过`ctx.session`接口，对`session`进行操作了(`set`方法是用来直接给`ctx.session`赋值的)。

在执行`ctx.session[name]`或`ctx.session[name]=value`时，都会执行这里的`get`方法，其代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  get() {
    const session = this.session;
    // already retrieved
    if (session) return session;
    // unset
    if (session === false) return null;

    // 对于cookie模式，会执行initFromCookie方法，初始化session
    // create an empty session or init from cookie
    this.store ? this.create() : this.initFromCookie();
    return this.session;
  }
}
```

可以看到，在`cookie`模式下，第一次访问会执行`initFromCookie`方法，对`session`进行初始化，其代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  initFromCookie() {
    debug('init from cookie');
    const ctx = this.ctx;
    const opts = this.opts;

    // 从Cookie请求头中获取session的数据，如果opts.signed为true，内部会做校验
    const cookie = ctx.cookies.get(opts.key, opts);
    if (!cookie) {
      this.create();
      return;
    }

    let json;
    debug('parse %s', cookie);
    try {
      // 使用opts.decode解析数据
      json = opts.decode(cookie);
    } catch (err) {
      // ...
    }

    debug('parsed %j', json);

    // 再次对session的数据做校验
    if (!this.valid(json)) {
      this.create();
      return;
    }

    // 根据当前session的数据，创建Session实例
    // support access `ctx.session` before session middleware
    this.create(json);
    this.prevHash = util.hash(this.session.toJSON());
  }

  valid(value, key) {
    const ctx = this.ctx;
    // 1. session无值时，派发session:missed事件
    if (!value) {
      this.emit('missed', { key, value, ctx });
      return false;
    }

    // 2. session过期时，派发session:expired事件
    if (value._expire && value._expire < Date.now()) {
      debug('expired session');
      this.emit('expired', { key, value, ctx });
      return false;
    }

    // 3. 如果提供了valid选项，会再次做校验，不通过时派发session:invalid事件
    const valid = this.opts.valid;
    if (typeof valid === 'function' && !valid(ctx, value)) {
      // valid session value fail, ignore this session
      debug('invalid session');
      this.emit('invalid', { key, value, ctx });
      return false;
    }
    return true;
  }

  emit(event, data) {
    setImmediate(() => {
      // 向app派发事件
      this.app.emit(`session:${event}`, data);
    });
  }
}
```

可以看到，在`initFromCookie`方法中，首先从`Cookie`请求头中获取`session`的数据，然后使用`opts.decode`方法，将其解析成对象，接着对数据进行校验，如果全部通过，就调用`this.create(json)`方法，创建对应的`Session`实例；否则，就调用`this.create()`方法，创建一个新的`Session`实例，其代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  create(val, externalKey) {
    debug('create session with val: %j externalKey: %s', val, externalKey);
    // 对于cookie模式，这里不会执行
    if (this.store) this.externalKey = externalKey || this.opts.genid && this.opts.genid(this.ctx);
    // 创建Session实例
    this.session = new Session(this, val);
  }
}
```

```js
/* koa-session/lib/session.js */
class Session {
  constructor(sessionContext, obj) {
    this._sessCtx = sessionContext;
    this._ctx = sessionContext.ctx;
    if (!obj) {
      this.isNew = true;
    } else {
      for (const k in obj) {
        // restore maxAge from store
        if (k === '_maxAge') this._ctx.sessionOptions.maxAge = obj._maxAge;
        else if (k === '_session') this._ctx.sessionOptions.maxAge = 'session';
        else this[k] = obj[k];
      }
    }
  }
}
```

可以看到，对于新建的`Session`，其`isNew`属性为`true`，其他情况下，就将数据拷贝到`Session`实例中即可。

在首次创建`Session`实例后，就可以通过`ctx.session`直接取到该`Session`的实例，然后就可以像普通对象一样，向其中添加或删除数据了。

当`session`之后的中间件执行完毕后，程序会再次执行到`session`中间件中，由于默认的`opts.autoCommit`为`true`，所以会执行`sess.commit`方法，对`session`进行提交，其代码如下所示：

```js
/* koa-session/lib/context.js */
class Session {
  async commit() {
    const session = this.session;
    const opts = this.opts;
    const ctx = this.ctx;

    // not accessed
    if (undefined === session) return;

    // 删除session
    // removed
    if (session === false) {
      await this.remove();
      return;
    }

    // 判断是否需要更新session
    const reason = this._shouldSaveSession();
    debug('should save session: %s', reason);
    if (!reason) return;

    // beforeSave钩子函数
    if (typeof opts.beforeSave === 'function') {
      debug('before save');
      opts.beforeSave(ctx, session);
    }
    // 更新session
    const changed = reason === 'changed';
    await this.save(changed);
  }
}
```

在`commit`方法中，会根据此时`session`的状态，对`session`做各种不同的操作，如果此时`session`为`false`，就调用`remove`方法，删除`session`，否则，会调用`_shouldSaveSession`方法，判断是否需要更新`session`，如果需要更新，就调用`save`方法，重新保存`session`，其代码如下所示：

```js
/* koa-session/lib/context.js */
class Session {
  _shouldSaveSession() {
    // 构建Session实例时，根据上一次session的数据创建的摘要
    const prevHash = this.prevHash;
    const session = this.session;

    // force save session when `session._requireSave` set
    if (session._requireSave) return 'force';

    // do nothing if new and not populated
    const json = session.toJSON();
    if (!prevHash && !Object.keys(json).length) return '';

    // 1. 新旧session的摘要不相同，说明做过修改，需要做更新操作
    // save if session changed
    const changed = prevHash !== util.hash(json);
    if (changed) return 'changed';

    // 2. 强制重新设置session
    // save if opts.rolling set
    if (this.opts.rolling) return 'rolling';

    // 3. 如果过期时间超过了一半，则需要重新设置session
    // save if opts.renew and session will expired
    if (this.opts.renew) {
      const expire = session._expire;
      const maxAge = session.maxAge;
      // renew when session will expired in maxAge / 2
      if (expire && maxAge && expire - Date.now() < maxAge / 2) return 'renew';
    }

    return '';
  }

  async remove() {
    // 设置expires为Thu, 01 Jan 1970 00:00:00 GMT，强制让cookie过期
    // Override the default options so that we can properly expire the session cookies
    const opts = Object.assign({}, this.opts, {
      expires: COOKIE_EXP_DATE, // 'Thu, 01 Jan 1970 00:00:00 GMT'
      maxAge: false,
    });

    const ctx = this.ctx;
    const key = opts.key;
    const externalKey = this.externalKey;

    // 对于cookie模式，这里不会执行
    if (externalKey) await this.store.destroy(externalKey);
    // 重新设置cookie，使cookie过期
    ctx.cookies.set(key, '', opts);
  }

  async save(changed) {
    const opts = this.opts;
    const key = opts.key;
    const externalKey = this.externalKey;
    let json = this.session.toJSON();
    // set expire for check
    let maxAge = opts.maxAge ? opts.maxAge : ONE_DAY;
    if (maxAge === 'session') {
      // do not set _expire in json if maxAge is set to 'session'
      // also delete maxAge from options
      opts.maxAge = undefined;
      json._session = true;
    } else {
      // 给数据添加_expire、_maxAge字段，用于下次对session的校验
      // set expire for check
      json._expire = maxAge + Date.now();
      json._maxAge = maxAge;
    }

    // 对于cookie模式，这里不会执行
    // save to external store
    if (externalKey) {
      // ...
    }

    // 使用opts.encode编码数据
    // save to cookie
    debug('save %j to cookie', json);
    json = opts.encode(json);
    debug('save %s', json);

    // 将session的数据添加到cookie中
    this.ctx.cookies.set(key, json, opts);
  }
}
```

可以看到，通过`remove`方法，就可以在`cookie`中删除`session`，通过`save`方法，就可以在`cookie`中更新`session`。那么接下来，我们就来看看`store`模式是如何工作的。

## store模式

在`cookie`模式中，所有的数据都是保存在客户端的，由于默认只是使用`base64`编码，所以用户可以直接得到`session`中的数据，而为了避免用户获取`session`中的数据，就要将这部分数据保存在后台，所以`koa-session`模块还提供`store`模式，用于将`session`保存在后台，用户只能获取到`sessionId`，这样用户就不能直接拿到`session`中的数据了。

`store`模式大体上与`cookie`模式类似，只是处理方式不同，为了启用`store`模式，需要传入`store`或`ContextStore`选项，在创建`ContextSession`的实例时，会根据配置设置`store`属性，代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  constructor(ctx, opts) {
    // ...
    this.store = this.opts.ContextStore ? new this.opts.ContextStore(ctx) : this.opts.store;
  }
}
```

然后在执行`session`中间件时，会调用`sess.initFromExternal`方法，从`store`中初始化`session`，其代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  async initFromExternal() {
    debug('init from external');
    const ctx = this.ctx;
    const opts = this.opts;

    let externalKey;
    if (opts.externalKey) {
      // 通过自定义的externalKey，从请求中获取sessionId
      externalKey = opts.externalKey.get(ctx);
      debug('get external key from custom %s', externalKey);
    } else {
      // 通过opts.key，从cookie中获取sessionId，这里cookie已经不是session的数据了
      externalKey = ctx.cookies.get(opts.key, opts);
      debug('get external key from cookie %s', externalKey);
    }


    if (!externalKey) {
      // create a new `externalKey`
      this.create();
      return;
    }

    // 通过sessionId，从store中获取session的数据
    const json = await this.store.get(externalKey, opts.maxAge, { rolling: opts.rolling });
    if (!this.valid(json, externalKey)) {
      // create a new `externalKey`
      this.create();
      return;
    }

    // 根据当前session的数据，创建Session实例
    // create with original `externalKey`
    this.create(json, externalKey);
    this.prevHash = util.hash(this.session.toJSON());
  }

  create(val, externalKey) {
    debug('create session with val: %j externalKey: %s', val, externalKey);
    // 首次创建时，通过opts.genid，生成sessionId
    if (this.store) this.externalKey = externalKey || this.opts.genid && this.opts.genid(this.ctx);
    // 创建Session实例
    this.session = new Session(this, val);
  }
}
```

可以看到，与之前`cookie`模式不同的是，这里从`cookie`中取到的是`sessionId`，而不是`session`的数据，所以`store`需要提供`get`方法，用于通过`sessionId`从`store`中取出其对应的`session`数据，其余的验证、创建`Session`实例的逻辑和之前是类似的。

在处理完成后，同样需要调用`commit`方法，对`session`进行提交，这里与之前的区别就是保存`session`的方式不同，代码如下所示：

```js
/* koa-session/lib/context.js */
class ContextSession {
  async remove() {
    // ...
    // 调用store.destroy方法，从store删除sessionId对应的数据
    if (externalKey) await this.store.destroy(externalKey);
    ctx.cookies.set(key, '', opts);
  }

  async save(changed) {
    // ...
    // save to external store
    if (externalKey) {
      debug('save %j to external key %s', json, externalKey);
      if (typeof maxAge === 'number') {
        // ensure store expired after cookie
        maxAge += 10000;
      }
      // 调用store.set方法，在store中保存session的数据
      await this.store.set(externalKey, json, maxAge, {
        changed,
        rolling: opts.rolling,
      });
      if (opts.externalKey) {
        // 通过自定义的externalKey，在响应头中添加sessionId
        opts.externalKey.set(this.ctx, externalKey);
      } else {
        // 将sessionId保存在cookie中
        this.ctx.cookies.set(key, externalKey, opts);
      }
      return;
    }
    // ...
  }
}
```

可以看到，对于`store`模式来说，保存和删除`session`，是通过调用`store.destroy`和`store.set`方法实现的，也就是说，`session`的数据可以保存在任何位置，客户端只需要通过`sessionId`，就可以间接访问到`session`的数据。

## 总结

在`Koa`应用程序中，`koa-session`模块可以提供对`session`的支持，它有`cookie`和`store`两种使用方式，`cookie`模式会将数据存储在客户端，`store`模式可以自定义存储的位置，客户端只需要提供`sessionId`即可。
