---
title: 从 electron-updater 学习如何实现应用更新和发布
date: 2022-02-07 00:16:23
tags:
categories:
---
# 前言
最近倒腾了一段时间的 electron 应用，略有收获。接下来，我主要想给大家分享的是我从 **electron-updater** 中学到的东西。

# 自动更新

- 下载新安装包更新
- 局部文件更新

使用新安装包更新分为两种，全量更新和增量更新。在对比了 electron 官方提供的方案和开源社区提供的方案后，最终选择了 [electron-updater](https://www.npmjs.com/package/electron-updater)。**electron-updater** 同时支持全量更新和增量更新。

不管是全量更新，还是增量更新，基本流程都是下载新的安装包等待重新安装，只不过后者只需要下载部分数据就能形成一个新的安装包。

除了这种方式，还有一种就是局部文件更新。我们知道，通常 electron 应用需要更新的部分都集中在 **app.asar** 这个文件中，有的时候可能是在 **app.asar.unpacked** 中。

有些同学可能没接触过 app.asar.unpacked，这个是使用 asarUnpack 特性后生成的目录，所有指定不需要打包到 app.asar 压缩文件的文件都会存储在这个目录中。electron 应用在启动时，会先把这个目录中的文件拷贝到 app.asar 中，然后才真正启动。社区里有  **[electron-asar-hot-updater](https://github.com/yansenlei/electron-asar-hot-updater)** ，大体工作流程是检测是否有新版本的 app.asar 压缩文件，有的话就下载覆盖应用安装目录里的 app.asar，然后重新启动。

## 全量更新

![全量更新.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d57b7f766d4f4653a3150516d440a2ac~tplv-k3u1fbpfcp-watermark.image?)
<center><div>图1</div></center>

全量更新的大体逻辑就是 **图1** 所示。在打包成安装程序时，就一并生成 update.yml 文件。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5085355891f04b0cb28e9cd5977e7173~tplv-k3u1fbpfcp-watermark.image?)

<center>图2</center>

如 **图2** 所示，*update.yml* 记录着一些新版本安装程序的信息，其中最重要的三个字段是 **version**、**path** 和 **sha512**。*.yml* 文件是很常见的文件格式，我们可以使用 [js-yaml](https://www.npmjs.com/package/js-yaml) 可以用来解析 [yaml](https://baike.baidu.com/item/YAML/1067697?fr=aladdin) 文件。

```javascript
const yaml = require('js-yaml');
const fs   = require('fs');

// Get document, or throw exception on error
try {
  const doc = yaml.load(fs.readFileSync('/home/ixti/example.yml', 'utf8'));
  console.log(doc);
} catch (e) {
  console.log(e);
}
```

*update.yml* 和新安装程序一起上传到服务器上，之后就可以按照 **图1** 的逻辑来处理。在对比远程最新版号和本地版本号时，可以使用 [semver](https://www.npmjs.com/package/semver) 。

```javascript
const semver = require('semver')

semver.valid('1.2.3') // '1.2.3'
semver.valid('a.b.c') // null
semver.clean('  =v1.2.3   ') // '1.2.3'
semver.satisfies('1.2.3', '1.x || >=2.5.0 || 5.0.0 - 7.2.3') // true
semver.gt('1.2.3', '9.8.7') // false
semver.lt('1.2.3', '9.8.7') // true
semver.minVersion('>=1.0.0') // '1.0.0'
semver.valid(semver.coerce('v2')) // '2.0.0'
semver.valid(semver.coerce('42.6.7.9.3-alpha')) // '42.6.7'
```

## 增量更新

增量更新是指只需要请求部分数据就可以实现版本更新，下面我只讲 *electron-updater* 中采用的增量更新方案。

*electron-updater* 采用的是基于内容可变长度分块（*Content-Defined Chunking*, *CDC*）的的软件增量更新方式，这种方式解决了其他增量更新中不相邻版本需要单独打patch的问题，同时仅需要下载部分文件内容就可以生成新版本的安装包。

### 内容可变长度分块——CDC

内容分块采用滑动窗口技术对文件进行分块，其核心操作是确定每个窗口的边界，它应用数据指纹如 **Rabin** 指纹算法将文件分割成长度大小不等的内容块。在算法执行过程中，CDC 使用一个固定大小的滑动窗口对文件数据计算数据指纹。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a867ef4c67fb4d8fba6cd4f8d961ecbb~tplv-k3u1fbpfcp-watermark.image?)
<center>图3</center>

如 **图3** 所示，一般来说会有相应程序使用指纹算法将安装程序分割成长度不等的数据块，然后计算不同数据块的 **hash** 值和 **size**，并将其记录到一个 **blockmap** 文件中。因此每个版本的安装程序都有对应的 *blockmap* 文件。之后只需要对比两个版本的 *blockmap* 文件，计算出哪些数据块不同，然后使用分段请求（*range：bytes=x-y*）请求最新安装程序的这部分数据，写入到旧安装程序就可以生成新的安装程序。

*electron-updater* 的更新是使用 [app-builder-bin](https://www.npmjs.com/package/app-builder-bin)  编译生成 *blockmap* 文件。该 *blockmap* 文件使用了 **gzip** 压缩，实际文件内容如下所示。
```json
{
    "version": "2",
    "files": [
        {
            "name": "file",
            "offset": 0,
            "checksums": [
                "VxFZSGxhDYz5FXgMiOUk5oCc"
            ],
            "sizes": [
                32768
            ]
        }
    ]
}
```

## 局部更新

![局部文件更新.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66ca0d35f506498395f7f944a7a82671~tplv-k3u1fbpfcp-watermark.image?)
<center>图4</center>

如 **图4** 所示，局部文件更新的逻辑相比全量更新就比较简单了。这里，*electron-asar-hot-update* 网络请求没有使用 *electron* 提供的 *net* api，而是使用了 *request* 这个库。

需要注意的一点是，在 *windows* 环境中替换 *app.asar* 时，如果直接在应用的主进程中通过 *nodejs* 去替换，会因为文件占用而报错，因为主进程的代码也是在 *app.asar* 中的。所以这个库单独写了个 *update.exe* 程序，调用该程序异步处理这个文件的替换。

ps： 类 *Unix* 的系统，通常不存在文件占用的问题，你可以试着在 mac 系统下写个 nodejs 脚本，让它自己删除自己，看是否会报错。

# 灰度发布
全量、增量、局部，3种类型的更新基本都讲到了，处理完更新，我们就可以发布应用了。

等等！发布应用是直接就把应用程序安装包的下载链接放到官网上么？当然不是。首次安装我们当然希望所有用户都能使用上我们的应用，但是过了一段时间，我们又新增了一个功能，又该如何发布这个新版本？

先回答这个问题再讲原因。我们肯定不是直通过之前讲的更新逻辑直接将旧版本更新到最新版本。虽然，新版本经过了内部的各种测试，内部觉得已经很稳定了，但是发布给用户还是要谨慎对待的。我们希望一部分用户可以先使用到新版本，收集一些意见反馈，及时修复一些未被发现的 bug，再逐步扩大体验新版本的用户范围。这就是灰度发布。

> 灰度发布（又名**金丝雀发布**）是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行A/B testing，即让一部分用户继续用产品特性A，一部分用户开始用产品特性B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。

这是*百度百科*上对灰度发布的解释。

如 **图5** 所示，这里面关键的地方在于首先要生成一个不变的 **guid**，然后对这个 *guid* 进行哈希，取得 *hashCode* 后对 100 取模得到一个 0-100 的数字，根据灰度发布的进度来确定当前应用是否需要更新到新版本。

[electron-updater](https://www.npmjs.com/package/electron-updater) 本身也包含了灰度发布的逻辑。它会生成一个不变的 **stagingUserId** 写到本地文件中，作为唯一 *id*，然后计算出一个 0-100 的数字后，和服务器上的 *update.yml* 中的 **stagingPercentage** 字段对比来确定是否要更新新版本。

至于如何推进灰度发布，大家可以有自己的实现。

# 结束语
春节期间断断续续写了几天才完成的，希望这篇总结对大家实际开发 electron 应用有帮助，感谢大家的阅读～
