# [Koa源码学习] koa-bodyparser

## 前言

在原生的`http`模块中，请求`req`是`http.IncomingMessage`的实例，它是一个可读流，我们可以从中获取请求主体。在`Koa`中，通常会使用`koa-bodyparser`模块，它会解析请求数据，然后将其添加到`ctx.request.body`上，那么接下来，我们就来看看其内部是如何实现的。

## 注册中间件

`koa-bodyparser`模块的默认导出方法如下所示：

```js
/* koa-bodyparser/index.js */
module.exports = function (opts) {
  // ...

  // koa-bodyparser模块支持解析form、json、text、xml类型的请求，默认开启对json、form的解析
  var enableTypes = opts.enableTypes || ['json', 'form'];
  var enableForm = checkEnable(enableTypes, 'form');
  var enableJson = checkEnable(enableTypes, 'json');
  var enableText = checkEnable(enableTypes, 'text');
  var enableXml = checkEnable(enableTypes, 'xml');

  // ...

  // 各种类型对应的Content-Type

  // default json types
  var jsonTypes = [
    'application/json',
    'application/json-patch+json',
    'application/vnd.api+json',
    'application/csp-report',
  ];

  // default form types
  var formTypes = [
    'application/x-www-form-urlencoded',
  ];

  // default text types
  var textTypes = [
    'text/plain',
  ];

  // default xml types
  var xmlTypes = [
    'text/xml',
    'application/xml',
  ];

  // ...

  // 对Content-Type做扩展
  var extendTypes = opts.extendTypes || {};

  extendType(jsonTypes, extendTypes.json);
  extendType(formTypes, extendTypes.form);
  extendType(textTypes, extendTypes.text);
  extendType(xmlTypes, extendTypes.xml);

  // 返回koa-bodyparser中间件
  return async function bodyParser(ctx, next) {
    // ...
  };
};
```

可以看到，`koa-bodyparser`可以处理`form`、`json`、`text`、`xml`类型的请求，并且可以通过`extendTypes`选项扩展请求的`Content-Type`，在处理完选项配置后，返回一个中间件`bodyParser`。

## 解析消息

当收到来自客户端的请求时，就会执行`bodyParser`中间件函数，其代码如下所示：

```js
/* koa-bodyparser/index.js */
module.exports = function (opts) {
  // ...

  return async function bodyParser(ctx, next) {
    if (ctx.request.body !== undefined) return await next();
    if (ctx.disableBodyParser) return await next();
    try {
      // 根据请求的Content-Type解析消息
      const res = await parseBody(ctx);
      ctx.request.body = 'parsed' in res ? res.parsed : {};
      if (ctx.request.rawBody === undefined) ctx.request.rawBody = res.raw;
    } catch (err) {
      if (onerror) {
        onerror(err, ctx);
      } else {
        throw err;
      }
    }
    await next();
  };

  async function parseBody(ctx) {
    if (enableJson && ((detectJSON && detectJSON(ctx)) || ctx.request.is(jsonTypes))) {
      return await parse.json(ctx, jsonOpts);
    }
    if (enableForm && ctx.request.is(formTypes)) {
      return await parse.form(ctx, formOpts);
    }
    if (enableText && ctx.request.is(textTypes)) {
      return await parse.text(ctx, textOpts) || '';
    }
    if (enableXml && ctx.request.is(xmlTypes)) {
      return await parse.text(ctx, xmlOpts) || '';
    }
    return {};
  }
};
```

可以看到，在`bodyParser`方法中，首先会使用`parseBody`方法，根据请求的`Content-Type`，调用不同的`parse`方法，从上下文中解析数据，如果解析成功，会将结果保存在`ctx.request.body`上，然后执行下一个中间件。那么接下来，我们就来看看`parse`方法是如何解析`json`和`form`的。

### json

处理`json`的方法是在`co-body`模块中定义的，其代码如下所示：

```js
/* co-body/lib/json.js */
const raw = require('raw-body');
const inflate = require('inflation');

module.exports = async function(req, opts) {
  req = req.req || req;
  opts = utils.clone(opts);

  // defaults
  let len = req.headers['content-length'];
  const encoding = req.headers['content-encoding'] || 'identity';
  if (len && encoding === 'identity') opts.length = len = ~~len;
  opts.encoding = opts.encoding || 'utf8';
  opts.limit = opts.limit || '1mb';
  const strict = opts.strict !== false;

  // 从req中提取请求主体，inflate用来处理压缩报文，raw用来从可读流中提取内容
  const str = await raw(inflate(req), opts);
  try {
    // 使用JSON.parse解析请求消息
    const parsed = parse(str);
    return opts.returnRawBody ? { parsed, raw: str } : parsed;
  } catch (err) {
    err.status = 400;
    err.body = str;
    throw err;
  }

  function parse(str) {
    if (!strict) return str ? JSON.parse(str) : str;
    // strict mode always return object
    if (!str) return {};
    // strict JSON test
    if (!strictJSONReg.test(str)) {
      throw new Error('invalid JSON, only supports object and array');
    }
    return JSON.parse(str);
  }
};
```

可以看到，在该方法中，首先对配置选项进行处理，然后调用`inflate`方法，其代码如下所示：

```js
/* inflation/index.js */
function inflate(stream, options) {
  // ...

  options = options || {}

  // 从请求头中提取Content-Encoding
  var encoding = options.encoding
    || (stream.headers && stream.headers['content-encoding'])
    || 'identity'

  // 根据Content-Encoding，判断是否需要解压后使用，这里支持gzip和deflate
  switch (encoding) {
  case 'gzip':
  case 'deflate':
    break
  case 'identity':
    return stream
  default:
    var err = new Error('Unsupported Content-Encoding: ' + encoding)
    err.status = 415
    throw err
  }

  // no not pass-through encoding
  delete options.encoding

  // 使用zlib.Unzip，对数据进行解压
  return stream.pipe(zlib.Unzip(options))
}
```

可以看到，如果请求主体是压缩后的，`inflate`方法会使用`zlib.Unzip`，对其进行解压处理。然后调用`raw`方法，该方法就是用来从请求中提取数据的，其主要代码如下所示：

```js
/* raw-body/index.js */
function getRawBody (stream, options, callback) {
  // ...
  // 返回一个Promise
  return new Promise(function executor (resolve, reject) {
    readStream(stream, encoding, length, limit, function onRead (err, buf) {
      if (err) return reject(err)
      resolve(buf)
    })
  })
}

function readStream (stream, encoding, length, limit, callback) {
  // ...
  // 在可读流上注册相应的事件，提取数据
  // attach listeners
  stream.on('aborted', onAborted)
  stream.on('close', cleanup)
  stream.on('data', onData)
  stream.on('end', onEnd)
  stream.on('error', onEnd)
  // ...
}
```

可以看到，在`readStream`方法中，主要就是在`req`可读流上注册`data`和`end`事件，用来提取数据，具体方法如下所示：

```js
function readStream (stream, encoding, length, limit, callback) {
  // ...
  function onData (chunk) {
    if (complete) return

    received += chunk.length

    if (limit !== null && received > limit) {
      // ...
    } else if (decoder) {
      buffer += decoder.write(chunk)
    } else {
      buffer.push(chunk)
    }
  }

  function onEnd (err) {
    if (complete) return
    if (err) return done(err)

    if (length !== null && received !== length) {
      // ...
    } else {
      var string = decoder
        ? buffer + (decoder.end() || '')
        : Buffer.concat(buffer)
      done(null, string)
    }
  }

  function done () {
    // ...
    if (sync) {
      process.nextTick(invokeCallback)
    } else {
      invokeCallback()
    }

    function invokeCallback () {
      // ...
      callback.apply(null, args)
    }
  }
  // ...
}
```

在每次触发`data`事件时，就将数据保存在`buffer`中，当流中数据读取完毕后，可读流会触发`end`事件，此时就会调用`done`方法，将读取到的数据返回，最终，`raw`方法返回`Promise`的结果就是请求携带的数据。

回到上面的`json`方法，在得到数据后，就会调用`parse`方法，其内部使用`JSON.parse`将数据转换成`json`对象，然后将其返回，最后就可以通过`ctx.request.body`访问到请求的数据了。

### form

处理`form`的方法同样是在`co-body`模块中定义的，其代码如下所示：

```js
/* co-body/lib/form.js */
const qs = require('qs');
const raw = require('raw-body');
const inflate = require('inflation');

module.exports = async function(req, opts) {
  req = req.req || req;
  opts = utils.clone(opts);
  const queryString = opts.queryString || {};

  // keep compatibility with qs@4
  if (queryString.allowDots === undefined) queryString.allowDots = true;

  // defaults
  const len = req.headers['content-length'];
  const encoding = req.headers['content-encoding'] || 'identity';
  if (len && encoding === 'identity') opts.length = ~~len;
  opts.encoding = opts.encoding || 'utf8';
  opts.limit = opts.limit || '56kb';
  opts.qs = opts.qs || qs;

  const str = await raw(inflate(req), opts);
  try {
    // 使用qs模块，解析请求数据
    const parsed = opts.qs.parse(str, queryString);
    return opts.returnRawBody ? { parsed, raw: str } : parsed;
  } catch (err) {
    err.status = 400;
    err.body = str;
    throw err;
  }
};
```

可以看到，在处理`form`时，主体逻辑和`json`是一样的，唯一的区别是在获取请求数据后，使用`qs.parse`方法，对数据进行解析，最终，也将数据放到`ctx.request.body`上。

## 总结

`koa-bodyparser`就是在`req`上注册可读流对应的事件，从而获取请求的数据，然后根据请求的`Content-Type`，使用不同的方式对其进行解析，然后将数据绑定在`ctx.request.body`中，在接下来的中间件中，就可以直接通过`ctx.request.body`访问到这些数据了。
