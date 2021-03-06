# STF代码分析

## 目录概述

- .git

- .tx

- bin

  - 包含stf文件，通过`require('../lib/cli/please')`把cli目录下定义的命令引入。

  - `package.json`中包含如下代码

  - ```javascript
    "bin": {
        "stf": "./bin/stf"
      },
    ```

  - 启动stf的方式为执行命令`stf local`,实际上执行的是cli目录下定义的local命令。

- doc

- docker

- lib

  - cli：命令行相关逻辑。
  - db：数据库相关逻辑。
  - units：定义各个服务模块，api、app、reaper等；其中app是web相关的模块，给予express搭建。
  - util：各种工具类，logger、procutil、lifecycleutil等。
  - wire：**？？？**

- node_modules

  - stf依赖的包，在`package.json`的配置；通过`npm install`安装。

- res

  - web前端相关

- rethinkdb_data

- test

- tmp

- vendor

  - minirev
    - 针对不同架构编译好的文件
    - 架构包括：arm64-v8a, armeabi, armeabi-v7a, mips, mips64, x86, x86_64
  - minitouch
    - 针对不同架构编译好的文件
    - 架构包括：arm64-v8a, armeabi, armeabi-v7a, mips, mips64, x86, x86_64
  - STFService
    - STFService.apk
    - wire.proto

- .bowerrc

- .dockerignore

- .editorconfig

- .eslintrc

- .gitignore

- .npmignore

- .travis.yml

- bower.json

- Dockerfile

- gulpfile.js

- package.json：nodejs配置文件

- webpack.config.js

**注：.md文件和LICESEN文件没有列出**



## STF部署机启动方式

- STF基于以下工具，需要安装
  - [Node.js](https://nodejs.org/) >= 6.9 (latest stable version preferred)
  - [ADB](http://developer.android.com/tools/help/adb.html) properly set up
  - [RethinkDB](http://rethinkdb.com/) >= 2.2
  - [GraphicsMagick](http://www.graphicsmagick.org/) (for resizing screenshots)
  - [ZeroMQ](http://zeromq.org/) libraries installed
  - [Protocol Buffers](https://github.com/google/protobuf) libraries installed
  - [yasm](http://yasm.tortall.net/) installed (for compiling embedded [libjpeg-turbo](https://github.com/sorccu/node-jpeg-turbo))
  - [pkg-config](http://www.freedesktop.org/wiki/Software/pkg-config/) so that Node.js can find the libraries


- 通过git clone把STF下载到本地

```bash
git clone https://github.com/openstf/stf.git哈哈
```

- Build

```bash
npm install
npm link
```

- 打开rethinkdb数据库

```bash
rethinkdb
```

- 运行stf

```bash
stf local
stf local --allow-remote
stf local --public-ip <your_internal_network_ip_here>
```

- 打开浏览器访问[http://localhost:7100](http://localhost:7100/)
- 插上Android手机就可以远程




## stf入口分析

- /bin/stf——>/lib/cli/please.js——>/lib/cli/index.js——>/lib/cli/local/index.js



## lib目录代码分析

### lib/cli/

- #### 简介

cli/把stf的不同模块封装成命令行调用的形式，通过类似stf local的形式启动相应的模块的功能。每个模块的具体是现在在units/目录下，每个模块都运行在独立的进程中，通过`child_process.fork(modulePath[, args][, options])`的形式启动各自的进程。

cli/通过[yargs](https://www.npmjs.com/package/yargs)封装命令行工具，主要使用了`.command(module)`方法构建，构建方式如下

> #### Providing a Command Module
>
> For complicated commands you can pull the logic into a module. A module simply needs to export:
>
> `exports.command`: string (or array of strings) that executes this command when given on the command line, first string may contain positional args
>
> `exports.aliases`: array of strings (or a single string) representing aliases of `exports.command`, positional args defined in an alias are ignored
>
> `exports.describe`: string used as the description for the command in help text, use `false` for a hidden command
>
> `exports.builder`: object declaring the options the command accepts, or a function accepting and returning a yargs instance
>
> `exports.handler`: a function which will be passed the parsed argv.
>
> 
>
> ```javascript
> // my-module.js 
> exports.command = 'get <source> [proxy]'
>  
> exports.describe = 'make a get HTTP request'
>  
> exports.builder = {
>   banana: {
>     default: 'cool'
>   },
>   batman: {
>     default: 'sad'
>   }
> }
>  
> exports.handler = function (argv) {
>   // do something with argv. 
> }
> ```
>
> You then register the module like so:
>
> ```javascript
> yargs.command(require('my-module'))
>   .help()
>   .argv
> ```

- #### 命令封装举例：lib/cli/app

通过`module.export.command`设置命令的名称

```javascript
module.exports.command = 'app'
```

通过`module.export.describle`设置命令的描述

```javascript
module.exports.describe = 'Start an app unit.'
```

通过`module.export.builder`设置命令的具体形式、参数（option）、读取的环境变量、参数的描述等。	

```javascript
#.env()：设置读取的环境变量前缀
#.strict()：使用严格模式
#.option()：设置参数
#	alias：参数别名
#	describle：参数描述
#	type：参数类型
#	demand：是否必须
#.epilog()：该command输入错误是现实的信息

module.exports.builder = function(yargs) {
  return yargs
    .env('STF_APP')
    .strict()
    .option('auth-url', {
      alias: 'a'
    , describe: 'URL to the auth unit.'
    , type: 'string'
    , demand: true
    })
    .option('port', {
      alias: 'p'
    , describe: 'The port to bind to.'
    , type: 'number'
    , default: process.env.PORT || 7105
    })
    .option('secret', {
      alias: 's'
    , describe: 'The secret to use for auth JSON Web Tokens. Anyone who ' +
        'knows this token can freely enter the system if they want, so keep ' +
        'it safe.'
    , type: 'string'
    , default: process.env.SECRET
    , demand: true
    })
    .option('ssid', {
      alias: 'i'
    , describe: 'The name of the session ID cookie.'
    , type: 'string'
    , default: process.env.SSID || 'ssid'
    })
    .option('user-profile-url', {
      describe: 'URL to an external user profile page.'
    , type: 'string'
    })
    .option('websocket-url', {
      alias: 'w'
    , describe: 'URL to the websocket unit.'
    , type: 'string'
    , demand: true
    })
    .epilog('Each option can be be overwritten with an environment variable ' +
      'by converting the option to uppercase, replacing dashes with ' +
      'underscores and prefixing it with `STF_APP_` (e.g. ' +
      '`STF_APP_AUTH_URL`).')
}
```

通过设置  制定执行该命令的具体逻辑，通过require的方式引入lib/units/中的实现。

```javascript
module.exports.handler = function(argv) {
  return require('../../units/app')({
    port: argv.port
  , secret: argv.secret
  , ssid: argv.ssid
  , authUrl: argv.authUrl
  , websocketUrl: argv.websocketUrl
  , userProfileUrl: argv.userProfileUrl
  })
}
```

通过`.command()`的方式在lib/cli/index.js中引入。

```javascript
var _argv = yargs.usage('Usage: $0 <command> [options]')
  .strict()
  .command(require('./api'))
  .command(require('./app'))
  ......
  .demandCommand(1, 'Must provide a valid command.')
  .help('h', 'Show help.')
  .alias('h', 'help')
  .version('V', 'Show version.', function() {
    return require('../../package').version
  })
  .alias('V', 'version')
  .argv
```



- #### index.js、please.js为/lib/cli的入口

参考stf入口分析，通过执行stf local命令，启动stf各个模块的进程。

- #### lib/cli/local/index.js分析

通过`procutil.fork()`返回一个Promise，首先fork migrate，相当于执行`stf migrate ***`。完成后执行`log.warn(***)`,这部分是我自己加的log。最后执行`run()`方法通过`procutil.fork()`方式启动各个模块的子进程，通过`Promise.all(procs)`返回一个Promise。

```javascript
#以下代码有删减
module.exports.handler = function(argv) {
  ......
  function run() {
    var procs = [
      // app triproxy
      procutil.fork()
      // device triproxy
      , procutil.fork()
      // processor one
      , procutil.fork()
      // processor two
      , procutil.fork()
      // reaper one
      , procutil.fork()
      // provider
      , procutil.fork()
      // auth
      , procutil.fork()
      // app
      , procutil.fork()
      // api
      , procutil.fork()
      // websocket
      , procutil.fork()
      // storage
      , procutil.fork()
      // image processor
      , procutil.fork()
      // apk processor
      , procutil.fork()
      // poorxy
      , procutil.fork()
    ]

    return Promise.all(procs)
      .then(function() {
      process.exit(0)
    })
      .catch(function(err) {
      log.fatal('Child process had an error', err.stack)
      return shutdown()
        .then(function() {
        process.exit(1)
      })
    })
  ......
  return procutil.fork(path.resolve(__dirname, '..'), ['migrate'])
    .then(function() {
      log.warn("Wony's STF Begin Running.")
    })
    .done(run)
}
```

- #### 不设置任何参数，不配置任何信息的情况下，stf local启动至都执行了那些具体Command

整理中。。。

### lib/db

- #### 简介

/db文件夹下是数据库相关代码，该部分基于rethinkdb官方提供javascript driver实现，通过如下方式使用rethinkdb提供reQL接口。该部分基于Promise方式实现。

```javascript
var r = require('rethinkdb')
```

参考[rethinkdb](https://rethinkdb.com/docs/guide/javascript/)官方文档，熟悉ReQL语法。 

- #### 数据表样例

users表：

```json
{

    "adbKeys": [
        {
            "fingerprint": "cc:13:2a:ae:31:89:14:5d:69:32:db:6d:da:d2:c1:b4" ,
            "title": "wony_0213@wony-PC"
        }
    ] ,
    "createdAt": Tue Apr 18 2017 07:35:30 GMT+00:00 ,
    "email": gaolifa@caict.ac.cn, »
    "forwards": [ ],
    "group": "fYZFdT8PTBqE27gdCXMEYw==" ,
    "ip": "::ffff:127.0.0.1" ,
    "lastLoggedInAt": Thu Jun 15 2017 07:54:23 GMT+00:00 ,
    "name": "test" ,
    "settings": {
        "deviceListActiveTabs": {
            "details": false ,
            "icons": true
        } ,
        "lastUsedDevice": "0815f81a7ab83004" ,
        "platform": "web"
    }

}
```

accessTokens表：

```json
rethinkdb数据库中暂无数据
```

vncauth表：

```json
rethinkdb数据库中暂无数据
```

devices表：

```json
{

    "abi": "arm64-v8a" ,
    "airplaneMode": false ,
    "battery": {
        "health": "good" ,
        "level": 100 ,
        "scale": 100 ,
        "source": "usb" ,
        "status": "full" ,
        "temp": 33.9 ,
        "voltage": 4.328
    } ,
    "browser": {
        "apps": [
            {
                "id": "com.sec.android.app.sbrowser/.SBrowserLauncherActivity" ,
                "name": "Browser" ,
                "selected": false ,
                "system": true ,
                "type": "samsung-sbrowser"
            }
        ] ,
        "selected": false
    } ,
    "channel": "KYM7aigduTE5q6GqnQbQh3hUruU=" ,
    "createdAt": Tue Apr 18 2017 07:40:51 GMT+00:00 ,
    "display": {
        "density": 3.5 ,
        "fps": 59 ,
        "height": 2560 ,
        "id": 0 ,
        "rotation": 0 ,
        "secure": true ,
        "size": 5.659716606140137 ,
        "url": "ws://localhost:7404" ,
        "width": 1440 ,
        "xdpi": 515.1539916992188 ,
        "ydpi": 520.1920166015625
    } ,
    "manufacturer": "SAMSUNG" ,
    "model": "SM-N9200" ,
    "network": {
        "connected": false ,
        "failover": false ,
        "roaming": false ,
        "subtype": null ,
        "type": null
    } ,
    "operator": "," ,
    "owner": {
        "email": gaolifa@caict.ac.cn, »
        "group": "fYZFdT8PTBqE27gdCXMEYw==" ,
        "name": "test"
    } ,
    "phone": {
        "iccid": null ,
        "imei": "352575072139889" ,
        "network": "UNKNOWN" ,
        "phoneNumber": null
    } ,
    "platform": "Android" ,
    "presenceChangedAt": Thu Jun 15 2017 07:55:10 GMT+00:00 ,
    "present": true ,
    "product": "nobleltezc" ,
    "provider": {
        "channel": "H1PQ2iayRgyW8qzg2MUf9w==" ,
        "name": "wony-PC"
    } ,
    "ready": true ,
    "remoteConnect": true ,
    "remoteConnectUrl": "localhost:7405" ,
    "reverseForwards": [ ],
    "sdk": "23" ,
    "serial": "0815f81a7ab83004" ,
    "status": 3 ,
    "statusChangedAt": Thu Jun 15 2017 07:55:10 GMT+00:00 ,
    "version": "6.0.1"

}
```

logs表：

```json
rethinkdb数据库中暂无数据
```



- #### table.js


table.js中定义了各个表的primaryKey和建立的索引。

参考rethinkdb官网[secondary indexes](https://rethinkdb.com/docs/secondary-indexes/javascript/)部分的介绍。

users：

```javascript
primaryKey: 'email'
indexes: adbKeys
```

accessTokens:

```javascript
primaryKey: 'id'
indexes: email
```

vncauth:

```javascript
primaryKey: 'password'
indexes: response, responsePerDevice
```

devices:

```javascript
primaryKey: 'serial'
indexes: owner, present, providerChannel
```

logs:

```javascript
primaryKey: 'id'
```

- #### setup.js


setup.js中进行数据库的初始化设置，通过`module.exports = function(conn)`的形式导出数据库设置方法。通过Promise的形式执行创建数据库，创建数据表，创建索引的流程。

首先调用`createDatabase()`,完成后根据`var tables = require('./tables')`的值调用`createTable()`方法，完成后调用`createIndex()`方法。参考[bluebird](http://bluebirdjs.com/docs/api-reference.html)的文档，注意resolve（）方法和return（）方法。

`.return(any value) -> Promise`相当于`.then(function() {return value;});`

```javascript
module.exports = function(conn) {
  var log = logger.createLogger('db:setup')

  function alreadyExistsError(err) {
    return err.msg && err.msg.indexOf('already exists') !== -1
  }

  function noMasterAvailableError(err) {
    return err.msg && err.msg.indexOf('No master available') !== -1
  }

  function createDatabase() 

  function createIndex(table, index, options) 

  function createTable(table, options) 

  return createDatabase()
    .then(function() {
      return Promise.all(Object.keys(tables).map(function(table) {
        return createTable(table, tables[table])
      }))
    })
    .return(conn)
}
```



- #### index.js


/db模块的默认入口，提供了有关数据库操作的基础方法。

<u>该部分用到了util目录下的srv和lifecycle，目前还未完全理解清晰。</u>

```javascript
var db = module.exports = Object.create(null)

function connect() 

// Export connection as a Promise
db.connect = (function())()

// Verifies that we can form a connection. Useful if it's necessary to make
// sure that a handler doesn't run at all if the database is on a break. In
// normal operation connections are formed lazily. In particular, this was
// an issue with the processor unit, as it started processing messages before
// it was actually truly able to save anything to the database. This lead to
// lost messages in certain situations.
db.ensureConnectivity = function(fn)

// Close connection, we don't really care if it hasn't been created yet or not
db.close = function()

// Small utility for running queries without having to acquire a connection
db.run = function(q, options)

// Sets up the database
db.setup = function() {
  return db.connect().then(function(conn) {
    return setup(conn)
  })
}
```

- #### api.js


提供了操作数据库的接口，用于对数据库中各个数据表的GRUD操作。



### lib/units

#### app

#### device

...

### lib/util

#### lifecycle.js

生命周期相关.

![lifecycle](./images/lifecycle.png)



#### zmqutil.js

对`zmq`的封装，为了防止长时间空闲造成的TCP断开，对所有的zmq socket的TCP_KEEPALIVE的参数进行设置。

通过`var zmqutil = require('../../util/zmqutil');var push = zmqutil.socket('push')`的方式调用。

zeroMQ使用：

- [zeromq官网](http://zeromq.org/community)


- [zmq使用参考](https://www.npmjs.com/package/zmq)

Push/Pull：

```javascript
// producer.js 
var zmq = require('zmq')
  , sock = zmq.socket('push');
 
sock.bindSync('tcp://127.0.0.1:3000');
console.log('Producer bound to port 3000');
 
setInterval(function(){
  console.log('sending work');
  sock.send('some work');
}, 500);

// worker.js 
var zmq = require('zmq')
  , sock = zmq.socket('pull');
 
sock.connect('tcp://127.0.0.1:3000');
console.log('Worker connected to port 3000');
 
sock.on('message', function(msg){
  console.log('work: %s', msg.toString());
});
```



Pub/Sub:

```javascript
// pubber.js 
var zmq = require('zmq')
  , sock = zmq.socket('pub');
 
sock.bindSync('tcp://127.0.0.1:3000');
console.log('Publisher bound to port 3000');
 
setInterval(function(){
  console.log('sending a multipart message envelope');
  sock.send(['kitty cats', 'meow!']);
}, 500);

// subber.js 
var zmq = require('zmq')
  , sock = zmq.socket('sub');
 
sock.connect('tcp://127.0.0.1:3000');
sock.subscribe('kitty cats');
console.log('Subscriber connected to port 3000');
 
sock.on('message', function(topic, message) {
  console.log('received a message related to:', topic, 'containing message:', message);
});
```



#### srv.js

与DNS，域名解析相关。









### lib/wire

...

## res目录代码分析

### res/app

### res/auth

### res/bower_componects

### res/build

### res/common

### res/test

### res/web_modules