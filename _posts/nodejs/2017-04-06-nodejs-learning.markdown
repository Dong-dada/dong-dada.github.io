---
layout: post
title:  "Node.js 基础知识"
date:   2017-04-06 11:54:30 +0800
categories: nodejs
---

 
 

本文参考自 [廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/001434502419592fd80bbb0613a42118ccab9435af408fd000)

Node.js 是一个 JavaScript 语言的运行时环境，你可以把它理解成是运行 JavaScript 的虚拟机。Node.js 是基于 Google V8 项目开发的。它对 V8 进行了一些优化，使得 V8 能够在非浏览器环境下运行得更好。

Node.js 的安装非常简单，从 [官网](https://nodejs.org/) 下载对应的安装包安装即可。

另外，Node.js 的安装包还附带了一个包管理工具 npm(nodejs package manager)。Node.js 的开发者社区共享了许多有用的包，使用 npm 可以方便地安装这些包，npm 会帮你解决模块间的依赖问题，把所有依赖的包都下载下来。避免了手工操作的繁琐。

要通过 node.js 来运行 js 代码，有两种方式，一种是在命令行中直接输入 node 命令，这样做会打开 node 的命令行模式，你可以在命令行中直接输入并执行 js 代码。另一种是将代码写在 js 文件里面，然后通过 `node xxx.js` 命令去执行这个 js 文件。

我们之前介绍 JS 的时候已经说够，编写 js 代码时要在 js 文件的开头写上 `'use strict'` 表示使用严格模式。你也可以在执行 node 命令时加上 `--use_strict` 参数，让 node 以严格模式加载我们的 js 代码。

## 模块

为了编写可维护的代码，我们把很多函数分组，分别放到不同的文件里，很多编程语言都采用这种组织代码的方式。在 Node 环境中，一个 .js 文件就称之为一个模块 (module)。

为了导出模块中定义的函数，我们需要使用 `module.exports = xxx;` 语句：

```js
// hello.js 文件

'use strict';

var s = 'Hello';

function greet(name) {
    console.log(s + ', ' + name + '!');
}

module.exports = greet;
```

这样就可以在其他文件中导入 `hello` 模块中导出的 `greet()` 函数了：

```js
// test.js 文件

'use strict';

// 引入hello模块:
var greet = require('./hello');

var s = 'Michael';

greet(s); // Hello, Michael!
```

注意，我们在 `hello.js` 中使用 `module.exports = greet;`, 其结果是将 `greet()` 这个函数对象导出。因此我们在 `test.js` 中使用 `require('./hello')` 将直接获取到 `greet()` 这个函数对象。

注意 `require('./hello')` 中使用相对路径来指定模块的位置，如果你直接写 `require('hello')`, 那么 node 将会依次在内置模块、全局模块、当前模块下查找 `hello.js`；

小结：
- 使用 `module.exports = variable;` 来对外导出变量；
- 使用 `var variable = require('other_module');` 来导入其他模块中的变量；

### 实现原理

这种模块加载机制被称为 CommonJS 规范。在这个规范下，每个 `.js` 文件都是一个模块，它们内部使用的变量名和函数名都互不冲突，例如 `hello.js` 和 `test.js` 中都声明了全局变量 `s`, 但互不影响。

你可能会奇怪，在浏览器环境下，多个 js 文件中的全局变量是会互相污染的，在 `a.js` 中声明的全局变量，其作用域会影响到 `b.js`。为什么 node.js 不会出现这种情况呢？

Node.js 实现这一效果的关键在于 JavaScript 提供的闭包机制。如果我们把一段代码用一个函数包裹起来，这段代码中的所有全局变量就变成了函数内部的局部变量。按照这种思路，实际上我们的 `hello.js` 会变成：

```js
(function () {
    // 读取的hello.js代码:
    var s = 'Hello';
    var name = 'world';

    console.log(s + ' ' + name + '!');
    // hello.js代码结束
})();
```

那么 `module.exports = variable;` 是如何将函数内部的局部变量导出的呢？

很简单，node 可以事先准备一个对象 `module`：

```js
// 准备module对象:
var module = {
    id: 'hello',
    exports: {}
};

var load = function (exports, module) {
    // 读取的hello.js代码:
    function greet(name) {
        console.log('Hello, ' + name + '!');
    }

    module.exports = greet;
    // hello.js代码结束
    return module.exports;
};
var exported = load(module);

// 把 module 保存到某个地方
save(module, exported);
```

可以看到，我们之所以可以在 js 文件中直接使用 `module.exports = variable;` 是因为 `module` 是作为函数的一个参数传递进来的。

当你调用 `require('./hello');` 时，require 会执行上述操作，将 `module` 记录到 node 环境的某个地方，这样下次 `require` 时就不会再次加载了。

### module.exports 和 exports

除了使用 `module.exports = variable;` 这种写法，你还可以用下述方法来导出变量：

```js
function hello() {
    console.log('Hello, world!');
}

function greet(name) {
    console.log('Hello, ' + name + '!');
}

exports.hello = hello;
exports.greet = greet;
```

之所以可以直接使用 `exports` 来赋值，是因为包裹这段代码的函数里，把 exports 作为参数传了进来，道理和 modules 是一样的：

```js
// 准备module对象:
var module = {
    id: 'hello',
    exports: {}
};

var load = function (exports, module) {
    // ...

    // 使用 export 或者 module.exports 都可以，因为它们俩都作为参数传进来了
    exports.hello = hello;
    module.exports.greet = greet;
}

var exported = load(module.exports, module);
```

但这里有一个需要注意的地方，你不能直接给 exports 赋值：

```js
// 正确，可以给 module.exports 赋值
module.exports = {
    hello : hello,
    greet : greet
};

// 错误，不能直接给 exports 赋值
exports = {
    hello : hello,
    greet : greet
};
```

其原因非常简单，因为直接给 `exports` 赋值相当于让 `exports` 指向了一个新的对象，这时候 `exports` 就不是原来的 `module.exports` 了。


## 基本模块

我们知道，在浏览器端 js 的执行受到了很大限制，不能直接处理网络请求、读写文件、处理二进制内容等。而 Node.js 是运行在服务端的，显然不能缺少这些功能，所以 Node.js 内置了一些模块让 js 能够执行这些浏览器环境中执行不了的操作。

### global

我们知道，浏览器中的 js 有且仅有一个全局模块 `window`, 类似的，Node.js 中也有这样一个全局模块 `global`. 在 node 交互环境中可以看到 `global` 的信息：

```
> global.console
Console {
  log: [Function: bound ],
  info: [Function: bound ],
  warn: [Function: bound ],
  error: [Function: bound ],
  dir: [Function: bound ],
  time: [Function: bound ],
  timeEnd: [Function: bound ],
  trace: [Function: bound trace],
  assert: [Function: bound ],
  Console: [Function: Console] }
```

### process

`process` 也是 Node.js 提供的一个对象，它代表当前的进程。

Node.js 程序是由事件驱动执行的单线程模型。它会不断执行响应事件的 JS 函数，直到没有任何响应事件的函数可以执行时， Node.js 就退出了。

如果我们想要在下一次事件响应中执行代码，可以调用 `process.nextTick()`：

```js
// test.js

// process.nextTick()将在下一轮事件循环中调用:
process.nextTick(function () {
    console.log('nextTick callback!');
});
console.log('nextTick was set!');

// 输出：
// nextTick was set!
// nextTick callback!
```

可以看到，`process.nextTick()` 中的函数不是立刻执行，而是在下一次事件循环中执行。

Node.js 进程本身的事件就由 `process` 来处理，如果我们希望在进程退出时执行一些代码，可以监听 `process` 的 `exit` 事件：

```js
// 程序即将退出时的回调函数:
process.on('exit', function (code) {
    console.log('about to exit with code: ' + code);
});
```

### fs

`fs` 应该是 `file system` 的缩写，用于文件操作。和所有其他 JS 模块不同的地方在于，它同时提供了 异步 和 同步 方式。

#### 异步读取文件

```js
'use strict';

var fs = require('fs');

// 读取文本
fs.readFile('sample.txt', 'utf-8', function (err, data) {
    if (err) {
        console.log(err);
    } else {
        console.log(data);
    }
});

// 读取二进制
fs.readFile('sample.png', function (err, data) {
    if (err) {
        console.log(err);
    } else {
        console.log(data);
        console.log(data.length + ' bytes');
    }
});
```

如果不指定文件编码，就以二进制的方式来读取文件。读取到的结果是一个 `Buffer` 对象，它是一个包含若干字节的数组(和 `Array` 不同)，`Buffer` 可以和 `String` 互相转换：

```js
// Buffer -> String
var text = data.toString('utf-8');
console.log(text);

// String -> Buffer
var buf = new Buffer(text, 'utf-8');
console.log(buf);
```

#### 同步读文件

```js
'use strict';

var fs = require('fs');

// 使用 try catch 捕获错误
try {
    var data = fs.readFileSync('sample.txt', 'utf-8');
    console.log(data);
} catch (err) {
    // 出错了
}
```

#### 写文件

```js
'use strict';

var fs = require('fs');

var data = 'Hello, Node.js';
fs.writeFile('output.txt', data, function (err) {
    if (err) {
        console.log(err);
    } else {
        console.log('ok.');
    }
});
```

如果传入的 `data` 是 `String` 对象，则默认以 `utf-8` 形式写入文本文件，如果 `data` 是 `Buffer` 对象，则以二进制方式写入文件。

同步方式也提供了一个 `fs.writeFileSync()` 函数。

```js
'use strict';

var fs = require('fs');

var data = 'Hello, Node.js';
fs.writeFileSync('output.txt', data);
```

#### stat

`fs.stat()` 函数可以获取文件的信息，它返回一个 `Stat` 对象：

```js
'use strict';

var fs = require('fs');

fs.stat('sample.txt', function (err, stat) {
    if (err) {
        console.log(err);
    } else {
        // 是否是文件:
        console.log('isFile: ' + stat.isFile());
        // 是否是目录:
        console.log('isDirectory: ' + stat.isDirectory());
        if (stat.isFile()) {
            // 文件大小:
            console.log('size: ' + stat.size);
            // 创建时间, Date对象:
            console.log('birth time: ' + stat.birthtime);
            // 修改时间, Date对象:
            console.log('modified time: ' + stat.mtime);
        }
    }
});
```

### stream

`stream` 也是 Node.js 提供的一个模块，其作用是支持流式传输。

#### 读取文件流

```js
'use strict';

var fs = require('fs');

// 打开一个流:
var rs = fs.createReadStream('sample.txt', 'utf-8');

rs.on('data', function (chunk) {
    console.log('DATA:')
    console.log(chunk);
});

rs.on('end', function () {
    console.log('END');
});

rs.on('error', function (err) {
    console.log('ERROR: ' + err);
});
```

#### 写入文件流

```js
'use strict';

var fs = require('fs');

var ws1 = fs.createWriteStream('output1.txt', 'utf-8');
ws1.write('使用Stream写入文本数据...\n');
ws1.write('END.');
ws1.end();

var ws2 = fs.createWriteStream('output2.txt');
ws2.write(new Buffer('使用Stream写入二进制数据...\n', 'utf-8'));
ws2.write(new Buffer('END.', 'utf-8'));
ws2.end();
```

所有可以读取数据的流都继承自 `stream.Readable`, 所有可以写入的流都继承自 `stream.Writable`。

#### pipe

就像可以把两个水管串成一个更长的水管一样，两个流也可以串起来。一个 `Readable` 流和一个 `Writable` 流串起来后，所有的数据自动从 `Readable` 流进入 `Writable` 流，这种操作叫 `pipe`。

在 Node.js 中，`Readable` 流有一个 `pipe()` 方法，其作用就是和另外一个 `Writable` 流连接起来。

```js
'use strict';

var fs = require('fs');

var rs = fs.createReadStream('sample.txt');
var ws = fs.createWriteStream('copied.txt');

rs.pipe(ws);
```

上述代码将实现把 `sample.txt` 中的数据拷贝到 `copied.txt` 中。

### http

要开发 HTTP 服务器程序，从头处理 TCP 连接，解析 HTTP 是不现实的。这些工作实际上已经由 Node.js 自带的 http 模块完成了。应用程序并不直接和 HTTP 协议打交道，而是操作 `http` 模块提供的 `request` 和 `response` 对象。
- `request` 对象封装了 HTTP 请求，我们调用 `request` 对象的属性和方法就可以拿到所有 HTTP 请求的信息；
- `response` 对象封装了 HTTP 响应，我们操作 `response` 对象的方法，就可以把 HTTP 响应返回给浏览器；

下面的代码实现了一个最简单的 Web 程序，它对于所有请求都返回 `Hello world!`

```js
'use strict';

// 导入http模块:
var http = require('http');

// 创建http server，并传入回调函数:
var server = http.createServer(function (request, response) {
    // 回调函数接收request和response对象,
    // 获得HTTP请求的method和url:
    console.log(request.method + ': ' + request.url);
    // 将HTTP响应200写入response, 同时设置Content-Type: text/html:
    response.writeHead(200, {'Content-Type': 'text/html'});
    // 将HTTP响应的HTML内容写入response:
    response.end('<h1>Hello world!</h1>');
});

// 让服务器监听8080端口:
server.listen(8080);

console.log('Server is running at http://127.0.0.1:8080/');
```

运行上述代码后，在浏览器中导航 `http://127.0.0.1:8080/` 即可看到效果。

#### 文件服务器

让我们扩展一下之前写的程序，我们可以设定一个目录，然后让 web 程序变成一个文件服务器。要实现这一点，只需要解析 `request.url` 中的路径，然后在本地找到对应的文件，把文件内容发送出去就可以了。

```js
'use strict';

var
    fs = require('fs'),
    url = require('url'),
    path = require('path'),
    http = require('http');

// 从命令行参数获取root目录，默认是当前目录:
var root = path.resolve(process.argv[2] || '.');

console.log('Static root dir: ' + root);

// 创建服务器:
var server = http.createServer(function (request, response) {
    // 获得URL的path，类似 '/css/bootstrap.css':
    var pathname = url.parse(request.url).pathname;
    // 获得对应的本地文件路径，类似 '/srv/www/css/bootstrap.css':
    var filepath = path.join(root, pathname);
    // 获取文件状态:
    fs.stat(filepath, function (err, stats) {
        if (!err && stats.isFile()) {
            // 没有出错并且文件存在:
            console.log('200 ' + request.url);
            // 发送200响应:
            response.writeHead(200);
            // 将文件流导向response:
            fs.createReadStream(filepath).pipe(response);
        } else {
            // 出错了或者文件不存在:
            console.log('404 ' + request.url);
            // 发送404响应:
            response.writeHead(404);
            response.end('404 Not Found');
        }
    });
});

server.listen(8080);

console.log('Server is running at http://127.0.0.1:8080/');
```

上述代码使用 `url` 模块来解析用户传入的 url, 使用 `path` 模块来构造目录，使用 `fs` 模块创建一个 `Readable` 流，并使用 `pipe()` 方法将文件内容写入到 `response` 中。

### crypto

Node.js 的 `crypto` 模块提供了通用的加解密和哈希算法。Node.js 用 C/C++ 实现这些算法后注册给了 JS 引擎。

#### MD5 和 SHA1

```js
const crypto = require('crypto');

const hash = crypto.createHash('md5');

// 可任意多次调用update():
hash.update('Hello, world!');
hash.update('Hello, nodejs!');

console.log(hash.digest('hex')); // 7e1977739c748beac0c0fd14fd26a544
```

使用 md5 进行 hash 时，需要注意文件大小的问题，如果文件比较大，不要一次性读入到内存，而是通过多次调用 `update()` 来逐步计算 md5 值。

如果要使用 SHA1 算法来计算，只需要把 `crypto.createHash('md5')` 改成 `crypto.createHash('sha1')` 就可以了。还可以使用更安全的 `sha256` 和 `sha512`。

#### Hmac

Hmac 也是一种哈希算法，但它可以指定一个密钥：

```js
const crypto = require('crypto');

const hmac = crypto.createHmac('sha256', 'secret-key');

hmac.update('Hello, world!');
hmac.update('Hello, nodejs!');

console.log(hmac.digest('hex')); // 80f7e22570...
```

只要密钥 `secret-key` 发生变化，同样的数据也会产生不同的 hash 值，你可以把它理解为用随机数 “增强” 的哈希算法。

#### AES

AES 是一种常用的对称加密算法，加解密都使用同一个密钥，`crypto` 模块也提供了支持：

```js
const crypto = require('crypto');

function aesEncrypt(data, key) {
    const cipher = crypto.createCipher('aes192', key);
    var crypted = cipher.update(data, 'utf8', 'hex');
    crypted += cipher.final('hex');
    return crypted;
}

function aesDecrypt(encrypted, key) {
    const decipher = crypto.createDecipher('aes192', key);
    var decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
}

var data = 'Hello, this is a secret message!';
var key = 'Password!';
var encrypted = aesEncrypt(data, key);
var decrypted = aesDecrypt(encrypted, key);

console.log('Plain text: ' + data);
console.log('Encrypted text: ' + encrypted);
console.log('Decrypted text: ' + decrypted);
```

需要注意的是，AES 有许多不同的算法，例如 `aes192`, `aes-128-ecb`, `aes-256-cbc` 等，除了密钥意外，AES 还可以指定 IV(Initial Vector), 不同的系统 IV 不同，加密结果也是不同的。加密的结果通常有两种形式：`hex`, `base64`。上述这些算法和结果格式 Node.js 都是支持的，但在应用中要注意，如果加解密的一方使用的是 Node.js 另外一方使用的是 java, PHP 等语言，需要仔细测试一下，如果无法正确解密，要确认双方使用的 AES 算法是否一致，字符串密钥和 IV 是否相同，加密后的格式是否一致。

#### Diffie-Hellman

DH 算法是一种密钥交换协议，它可以让双方在不泄露密钥的情况下协商出一个密钥出来。简单的过程如下： 
- 小红选择一个素数和一个底数，例如 素数 `p=23`, 底数 `g=5`(可以任选)，再自己随机生成一个整数 `a=6`, 然后根据公式 `A=g^a mod p` 计算出一个值 `A=8`。接着把 `p=23, g=5, A=8` 这件事告诉小明。如果想通过 `p, g, A` 这几个数字计算出 `a` 的话，需要利用公式 `a = log(g, (A*n)+p)`, 为了计算出 a, 必须不断带入 n, 如果 a 和 p 都是比较大的数那么计算将更加困难。因此第三方很难得知 `a` 的值是多少
- 小明也安装同样的方式，自己随机生成一个整数 `b=15`, 然后计算 `B=g^b mod p`, 把 `B=19` 传给小红；
- 小红拿到 `B=19` 之后，就可以使用 `s = B^a mod p` 计算出一个密钥 `s=2`, 同时小明也可以使用 `s = A^b mod p` 计算出同样的密钥 `s=2`；
- 小红和小明都拿到了密钥 `s=2`, 可以用这个密钥加密接下来的通信内容了。

Node.js 中实现 Diffie-Hellman 算法的接口如下：

```js
const crypto = require('crypto');

// xiaoming's keys:
var ming = crypto.createDiffieHellman(512);
var ming_keys = ming.generateKeys();

var prime = ming.getPrime();
var generator = ming.getGenerator();

console.log('Prime: ' + prime.toString('hex'));
console.log('Generator: ' + generator.toString('hex'));

// xiaohong's keys:
var hong = crypto.createDiffieHellman(prime, generator);
var hong_keys = hong.generateKeys();

// exchange and generate secret:
var ming_secret = ming.computeSecret(hong_keys);
var hong_secret = hong.computeSecret(ming_keys);

// print secret:
console.log('Secret of Xiao Ming: ' + ming_secret.toString('hex'));
console.log('Secret of Xiao Hong: ' + hong_secret.toString('hex'));
``` 
