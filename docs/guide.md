## 基础知识
> 前置知识：建议提前阅读 [Fastify 官网文档](https://www.fastify.cn/docs/latest/)
> 
hoth 是基于 [Fastify](https://github.com/fastify/fastify) 实现的一个 Node.js 框架。Fastify 的特点（摘自官网）：

- 高性能： 据我们所知，Fastify 是这一领域中最快的 web 框架之一，另外，取决于代码的复杂性，Fastify 最多可以处理每秒 3 万次的请求，详见 [benchmark](https://www.fastify.io/benchmarks/)。

- 可扩展： Fastify 通过其提供的钩子（hook）、插件和装饰器（decorator）提供完整的可扩展性。
- 基于 Schema： 建议使用 JSON Schema 来做路由（route）验证及输出内容的序列化，Fastify 在内部将 schema 编译为高效的函数并执行。

- 对开发人员友好： 框架的使用很友好，帮助开发人员处理日常工作，并且不牺牲性能和安全性。

- 支持 TypeScript： 我们努力维护一个 TypeScript 类型声明文件，以便支持不断成长的 TypeScript 社区。


**常见框架性能对比：**

|                          | Version | Router | Requests/s | Latency | Throughput/Mb |
| :--                      | --:     | --:    | :-:        | --:     | --:           |
| 0http                    | 3.1.2   | ✓      | 58164.8    | 16.70   | 10.37         |
| fastify                  | 3.27.4  | ✓      | 56779.2    | 17.13   | 10.13         |
| connect                  | 3.7.0   | ✗      | 55113.6    | 17.66   | 9.83          |
| koa                      | 2.13.4  | ✗      | 38181.8    | 25.69   | 6.81          |
| restify                  | 8.6.1   | ✓      | 35393.4    | 27.74   | 6.38          |
| hapi                     | 20.2.1  | ✓      | 32892.4    | 29.89   | 5.87          |
| egg.js                   | 2.35.0  | ✓      | 19266.0    | 51.40   | 6.89          |
| express                  | 4.17.3  | ✓      | 12636.4    | 78.60   | 2.25          |


## 框架设计
![](https://psstatic.cdn.bcebos.com/basics/hoth/image2021-1-25_13-48-12_1649155172000.png)


hoth 以 Fastify 为基础，内置多个核心插件（[插件列表](/plugin/index)），降低业务方调研成本。通过 node-base 基础模块，提供 node runtime、fastify api、core plugins，业务层各 app 拆分上线。

## 业务配置

### 基础配置
这些配置项会在使用 hoth-cli 初始化项目时自动加上。
```
{
    dir: '/home/work/search/nodeserver/app/myapp', // 当前 app 所部署的根目录，指 app.js 所在的目录
    prefix: '/myapp', // 当前 app 的 uri prefix
    name: 'myapp', // 当前 app 名称
    rootPath: '/home/work/search/nodeserver' // 当前 node 服务根目录
}
```

### 添加配置
配置功能基于 [node-config](https://github.com/node-config/node-config)

app全局配置，新增 src/config/default.ts 文件
```javascript
export const port = 8080;
export const dbUrl = 'xxx';
```
插件配置，新增 src/config/plugin.ts 文件，框架会自动将该文件的配置参数传递给指定插件
```javascript
export default {
    '@hoth/app-autoload': {
        prefix: '/someotherprefix'
    }
};
```
同时也支持 default.json 等

>1. 配置项名称不要与基础配置冲突，否则会覆盖基础配置
>2. 多个 app 之间配置是隔离的，无法获取到另一个 app 的配置

#### 开发/生产环境
添加 src/config/development.ts 、src/config/production.ts ，根据 NODE_ENV 环境变量的值，会选择加载不同的配置，并与 default 配置合并，例如：

src/config/default.ts 
```javascript
export const env = 'default';
export const example = 1;
```
src/config/development.ts 
```javascript
export const env = 'development';
```
NODE_ENV=deveopment，最终的配置为：
```javascript
{
    env: 'development',
    example: 1
}
```

### 获取配置

```javascript
const foo = req.$appConfig.get('foo');
```

## 日志

### 日志分类
所有日志级别，从上往下依次提高

- trace：线下使用
- debug：线下debug使用
- info：一般打点
- notice：请求日志，每次请求有且仅有一个
- warn：警告，不影响主体业务
- error：一般报错
- fatal：影响服务稳定性的报错

> 其中 trace 和 debug 日志只有在线下会输出；notice 和 fatal 日志在 hoth 框架级别进行了打点，不需要业务特别添加；

### 日志文件
每个 app 都会产出三份业务相关的日志文件：

- log/appname/appname.log.ti：trace debug info日志
- log/appname/appname.log：notice 日志
- log/appname/appname.log.wf：warn error fatal 日志

### 日志指纹
一般指的是日志里的 logid 字段，只需要上游在请求node的时候，增加一个  logid 头信息，hoth框架会自动使用 req.headers.logid

### 内置打点
#### notice日志
用来记录请求级别的打点信息，一次Node请求有且仅有一条，各种监控依赖，日志示例：

```
NOTICE: 2021-05-11 12:35:26 [-:-] errno[-] status[200] logId[xxx] pid[13929] uri[/trends/pc?tab=homepage&page=rule] cluster[hoth-node] idc[fj] product[trends] module[pc] clientIp[127.0.0.1] ua[lightMyRequest] refer[-] cookie[-] parseTime[0.5] validationTime[0.2] tm[-] responseTime[48.7]
```
字段解释：

- parseTime：preParsing 到 preValidation 的耗时，主要是 req.body parse 的时间

- validationTime：preValidation 到 preHandler 的耗时，主要是 fatsify schema 检查的时间

- responseTime：请求处理总耗时

- cluster：集群名称

- idc：机房名称

- product：对应 hoth 的 app 名称

- module：对应子模块的名称，默认是 path 的第二级，也可以自定义

#### fatal 日志
使用 fastify.setErrorHandler，进行打点，日志示例：

```
FATAL: 2021-05-11 12:34:41 [-:-] errno[-] status[500] logId[xxx] pid[2550] uri[/member/index?r=31609] cluster[bdapp-nodeserver] idc[fj] product[member] module[index] clientIp[127.0.0.1] ua[curl/7.29.0] refer[-] cookie[-] Error: 随机出错啦 for test     at AppController.getApp (/home/work/pandora/nodeserver/app/src/controller/index/index.controller.ts:15:19)     at preHandlerCallback (/home/work/pandora/nodeserver/node_modules/fastify/lib/handleRequest.js:124:28)     at next (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:158:7)     at Object.<anonymous> (/home/work/pandora/nodeserver/node_modules/@hoth/app-autoload/src/hook/preHandlerFactory.ts:23:9)     at hookIterator (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:237:10)     at next (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:164:16)     at Object.preHandler (/home/work/pandora/nodeserver/node_modules/@hoth/logger/src/index.ts:115:5)     at hookIterator (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:237:10)     at next (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:164:16)     at hookRunner (/home/work/pandora/nodeserver/node_modules/fastify/lib/hooks.js:187:3)
```
字段与 notice 类似，只是最后增加了错误信息

## 插件机制

### 插件分类
- 框架内置插件：hoth 框架内自动加载的插件，用于框架初始化等所有 app 都依赖的功能，不需要业务额外引入或者安装
- 开源插件：内网或者外网开源的插件，安装在业务自己的npm依赖里，需要手动引入
- 自定义插件：纯业务相关的插件，放在 plugin 目录下

### 框架内置插件

目前的内置插件包括：

- @hoth/app-autoload：自动加载各个 app 的 controller、plugin，初始化配置等
- fastify-warmup：实现预热功能
- @hoth/molecule：molecule 插件
  
**@hoth/app-autoload 配置**

src/config/plugin.ts 
```javascript
export default {
    '@hoth/app-autoload': {
        prefix: '/someotherprefix'
    }
};
```
目前只支持修改 prefix。

### 开源插件
即用 npm 安装在项目里的插件，有两种注册的方式

app.ts
```javascript
import type {FastifyInstance} from 'fastify';
import view from '@hoth/view';
import swig from 'swig';

export default async function main(fastify: FastifyInstance, opts: AppConfig) {
    fastify.register(view, {
        engine: {
            swig
        },
        templatesDir: opts.dir + '/view'
    });
    return;
}
```
config/plugin.ts
```javascript
import swig from 'swig';
import {join} from 'path';

export default {
    '@hoth/view': {
        engine: {
            swig
        },
        templatesDir: join(__dirname, '../view')
    }
};
```
这种方式适合配置简单的插件，它有一个问题是不会进行显式的 import，如果插件内有一些 ts 定义，会导致无法引入

### 自定义插件

> 如何编写一个自定义插件，请参考 Fastify 官网[插件漫游指南](https://www.fastify.cn/docs/latest/Plugins-Guide/)。

自定义插件通过 [fastify-autoload](https://github.com/fastify/fastify-autoload) 插件进行加载，可查看其 readme，初始化时会自动加载 src/plugin 目录下的文件。

如果这个插件需要装载部分字段到 request 或者 reply 上，并且在 controller 里生效，需要添加 skip-override 属性，如下所示：
```javascript
function p(fastify, options, done) {
    fastify.decorateRequest('addRequest', 'ok');
    done();
}
p[Symbol.for('skip-override')] = true;
modole.exports = p;
```

具体可以查看官方文档：https://www.fastify.io/docs/master/Plugins/#Handle%20the%20scope