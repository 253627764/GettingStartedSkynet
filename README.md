# 我所理解的 Skynet

## 目录

* 安装
* Hello Skynet
* 服务
* 服务广大群众

## 安装
参照 [Skynet 官方文档](https://github.com/cloudwu/skynet)

## Hello Skynet
云风在 Blog 里经常提起，你可以把 Skynet 当成一个操作系统。以下的内容正是基于这一灵感，如果你认为有类比不对或表达不清的地方，欢迎你给我提 [issue](https://github.com/lintide/GettingStartedSkynet/issues/new)

在很久很久以前，那还是一个需要自己去电脑城组装电脑的时代，店里的小哥记下你要的配置，组装好后回到家你高高兴兴地开机验货。在 Skynet 里的世界也是一样的，首先你需要一个配置单。

### 第一台 Skynet 电脑配置单
[hello/config](hello/config)：

```lua
root = "./"
thread = 4
harbor = 0 -- 单点模式
bootstrap = "snlua helloSkynet"	-- 启动第一个服务
luaservice = root.."service/?.lua;"..root.."test/?.lua;"..root.."GettingStartedSkynet/hello/?.lua"
```

配置单的信息

* 这是一台四核的电脑: `thread = 4`，如果你预算比较多，搞成8核，16核都可以
* 单机模式，不跟其它电脑通讯：`harbor = 0`，所以也没有任何网络接口
* 告诉 Skynet 电脑启动后运行 `helloSkynet` 程序：`bootstrap = "snlua helloSkynet"`
* `helloSkynet` 这个程序放在 `GettingStartedSkynet/hello/?.lua` 目录下: `luaservice = ...`


### 第一个程序
[hello/helloSkynet.lua](hello/helloSkynet.lua):

 ```lua
 local skynet = require "skynet"

 skynet.start(function()
   skynet.error("Hello Skynet!")
   skynet.exit()
 end)
 ```

* 导入 `skynet`
* 任何__服务__必须有启动函数 `skynet.start` ，`skynet.start` 需要转入 `function() ... end` 参数
* 在启动函数里打印 __Hello Skynet!__ 字符串，然后退出 `skynet.exit()`

### 验证电脑是否有毛病
```
$ ./skynet GettingStartedSkynet/hello/config
```

如果输出
```
[:00000001] LAUNCH logger
[:00000002] LAUNCH snlua helloSkynet
[:00000002] Hello Skynet!
[:00000002] KILL self
```

恭喜你，我们进入下一个旅程

## 服务
电脑能给我们提供什么服务？我相信绝大多数人购买电脑目的还是很单纯滴：为了学习，个别同志会想办法下下片。说起下片，有个神器 __kuaipo__。

### 下片
__kuaipo__ 最重要的就是下载文件的功能了，注意：以下功能仅作模拟，无法真正下片。

[unzip/kuaipo.lua](unzip/kuaipo.lua)

```lua
local kuaipo = {}

-- 快播有下载的功能
kuaipo.download = function(file)
  for i=0,100,10 do
    skynet.error(file .. " downloading... %" .. i)
    skynet.sleep(10)
  end
end
```

一般下载的文件都是 __.rar__ 格式的，下完片后还需要解压缩，祭出盗版软件 __WinRAR__ （突然发现好久没用过这个鬼东西了，满满都是回忆）。

### 自动解压
下完片，一般都需要我们自己手动解压缩。片下多了，一个个点开解压缩也很浪费时间，机智如你，想起了一个好办法，下载完后自动解压缩。那么问题来了，这是两个不同的软件，怎么让他们沟通起来呢？

```
kuaipo 对 WinRAR 说：片下好了，麻烦你解压一下。
```

转换为代码如下 [unzip/kuaipo.lua](unzip/kuaipo.lua)：
```lua
local winRAR = skynet.newservice("winRAR")
skynet.send(winRAR, "lua", "upzip", filename)
```

* `skynet.newservice` 创建一个新服务，类似我们在电脑上打开 __winRAR__ 这个软件
* `skynet.send` 告诉 __winRAR__ 解压文件 `filename`

现实生活中，我们是如何知道 __winRAR__ 可以解压缩文件的呢？大多应该都是听朋友讲的，口口相传，而在 skynet 的世界里你必须告诉 skynet 谁有解压缩文件的能力。

[unzip/winRAR.lua](unzip/kuaipo.lua)
```lua
skynet.dispatch("lua", function(session, source, cmd, filename, ...)
...
end)
```

通过调用 `skynet.dispatch` 函数来告诉 skynet 有一个叫 __winRAR__ 软件提供了解压缩文件的__能力__。

读完这一小节，你只要好好理解这三个函数即可：`newservice`, `send`, `dispatch`

注意 [config](upzip/config) 文件有变化哦，我也还没弄明白其中的道理，你先这样用着吧 :)

## Ping Pong
在进入下一个章节之前，我们先来一个小点心，实现一个非常简单的 c/s 程序，客户端发送`ping`字符串到服务端，服务端收到信息后，回复`pong`。

先来看看服务端代码
1. 建立服务，并监听`8888`端口

  ```lua
  gate = skynet.newservice("gate")

  skynet.call(gate, "lua", "open" , {
    address = "127.0.0.1", -- 监听地址 127.0.0.1
    port = 8888,    -- 监听端口 8888
    maxclient = 1024,   -- 最多允许 1024 个外部连接同时建立
    nodelay = true,     -- 给外部连接设置  TCP_NODELAY 属性
  })
  ```

2. 建立连接，接收信息
  ```lua
  skynet.dispatch("lua", function(session, source, cmd, subcmd, ...)
    skynet.error("cmd: "..cmd.."  subcmd: "..subcmd)
    if cmd == "socket" then
      if subcmd == "open" then
        newClient(...)

        ....

  ```

  在`newClient`


客户端我是用 nodejs 写的[pingpong/ping.js](pingpong/ping.js)。

1. 建立 socket

  ```javascript
  const client = net.connect({'host': 'localhost', 'port': 8888}, function(){
    ...
  })
  ```

2. 构造消息，并发送

  ```javascript
  var message = "ping";

  var pack = new Buffer(2+message.length);
  pack.writeUInt16BE(message.length, 0);
  pack.write(message, 2);

  // pack 的数据如下
  // | 00 04 | 70 69 6e 67 |
  // | size  |   message   |

  client.write(pack);
  ```

## 服务广大群众

```
独乐乐,与人乐乐,孰乐?

-- 《孟子》
```

随着片越下越多，你在朋友中也建立起了小小名气，人称『毛哥』。正所谓能力越大，责任越大，朋友之间也会让你分享下资源，看完大家有点共同话题，彼此之间可以交流一些经验。
