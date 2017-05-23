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

…

### lib/units

#### app

#### device

...

### lib/util

...

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