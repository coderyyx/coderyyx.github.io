---
title: Service Worker助力前端性能优化
date: 2022-04-15 10:09:40
tags: 性能优化
categories: 前端
---

![题图](https://storage.googleapis.com/seoviu_wordpress/blog/2020/11/def3dc08-serviceworker-darstellung-1.png)

> 图片来源：<https://www.searchviu.com/en/service-worker-what-seos-need-to-know/>

> 本文作者：杨运心

# 1. 前言
`Service Worker` 对于前端开发工程师们已经不是一个新的概念了，依稀记得前两年还比较火主要的应用场景是 `PWA`，现在好像没有那么火热了，主要还是业务场景不太适合，比如我们的移动端大部分都是跑在 app 内。`Service Worker` 目前在我们移动端应用主要场景是用来缓存 js、图片、第三方 js 等一些静态资源。最近做了一个 feature，就是利用 `Service Worker` 缓存 html，提升应用性能 & 站点性能。接下来就跟大家分享一下我们的技术方案以及过程中遇到的一些问题和解决思路还有最后优化的效果。

# 2、Service Worker 简介
为了防止大家有些许遗忘，这里还是回顾一下 Service Worker 的相关知识点，熟悉的同学可以跳过

Service Worker 是浏览器在后台独立于网页运行的脚本，它本质上是一种 `web worker`，我们知道如果有一些耗时的任务如果需要优化我们可能首先想到的方案就是放到 `web worker` 中执行，因为它完全独立于 js 主线程，不会产生长任务从而导致一些用户体验问题。它相对于普通的 `web worker` 有了离线缓存的功能，也就是拦截处理网络请求，包括通过程序来管理缓存中的响应。

这个 API 之前之所以火热，是因为它可以支持离线体验，让开发者能够全面控制请求和响应，所以这个能力的可操作空间很大，有丰富的想象力。

使用 Service Worker 有一些需要注意的点：
 - 生产环境必须在 `https` 下才能生效，因为 SW 的能力过于强大，所以必须要保证安全；
 - 它是一种 `web worker` 无法访问 DOM。 Service Worker 通过响应 [postMessage](https://html.spec.whatwg.org/multipage/workers.html#dom-worker-postmessage) 接口发送的消息来与其控制的页面通信，页面可在必要时对 DOM 执行操作。
 - 可以拦截作用域内页面发出的请求以及处理响应
 - 异步实现，大量使用了 `promise`

#### Service Worker 的生命周期
 简单介绍一下 Service Worker 的生命周期，它主要经过如下几个阶段：`installing` -> `installed` -> `activating` -> `activated` -> `redundant`

 <div align=center>
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/12396286292/29c6/a37d/a934/b3291a35422f3f68ab952440d2ef5ced.png"/>
</div>

> `INSTALLING`： 这个钩子表示 Service Worker 正在注册，**`我们一般会在这个阶段预缓存资源`**
`INSTALLED`： 这个阶段表示 Service Worker 注册成功，**`预缓存的资源也都成功缓存到本地`**
`ACTIVATING`： 当前页面以及没有其他 Service Worker 控制，**`我们一般在这个阶段清理过期的缓存`**
`ACTIVATED`： 激活态，在这个阶段 Service Worker 正在接管了页面，可以拦截页面请求等一些操作。
`REDUNDANT`： 废弃，当前 Service Worker 被新的替换

#### 如何借助 Service Worker 优化前端性能
Service Worker 对前端性能优化有什么帮助，到底能有多大的效果，我相信也是大家比较关注的。目前我们的方案是缓存静态资源在客户端来优化站点的性能，当然这也是业内比较常见的利用 Service Worker 优化前端性能的常见方案。我们都知道一个页面最终呈现在用户侧主要会经历 `资源下载` 以及 `浏览器渲染`，用下图简单呈现。

 <div align=center>
<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12468507922/ff68/e82f/fd98/5d54a28d37dd394130457cc255cb3dde.png"/>
</div>

从上图我们可以肯定的是为了让页面快速呈现在用户侧，缩短资源加载时间是一个可行的方案。下面的对比也直观反应出了使用了 Service Worker 对页面加载的提升。


# 3、Service Worker 缓存静态资源流程
下面我们先通过一个简单的 demo 梳理一下 Service Worker 缓存静态资源的大致流程。

使用 create-react-app 脚手架初始化的 PWA 模板初始化，因为这个模板已经集成了 Service Worker

```js
npx create-react-app serviceworker-demo --template cra-template-pwa
```

- 在入口文件 `index.js` 中注册 `Service Worker`
```js
serviceWorkerRegistration.register();
```

- Service Worker 的核心逻辑
```js

import { precacheAndRoute, createHandlerBoundToURL } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';

const precachelist = self.__WB_MANIFEST;

console.log('===========');
console.log(precachelist);

precacheAndRoute(precachelist);
```
我们可以看到使用了 [Workbox](https://developers.google.com/web/tools/workbox) 库实现相关逻辑，这个库是 `Google` 实验室推出的，它集成了 Service Worker 相关的最佳实践，也是现在比较推荐的开发 Service Worker 的库。

它主要干了两件事：
- 获取 `precachelist`，这个是我们应用在构建阶段生成的静态资源列表

- 通过 `precacheAndRoute` 这个方法把静态资源下载到本地并且缓存到本地 [Caches](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 中；然后注册路由，当 `Service Worker` 生效后会拦截页面发出的请求，当页面访问的资源发现已经缓存在 `Caches` 中后就会返回本地缓存的资源。

这个 demo 已经放到了 `github pages`。 [项目地址](https://github.com/coderyyx/serviceworker-demo)、[线上 demo 地址](https://coderyyx.github.io/serviceworker-demo/)

- 缓存的资源列表

控制台打出的日志即为 Service Worker 将预缓存到本地的静态资源列表，如下所示：
 <div align=center>
<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13048600064/a830/bc2d/93cc/70770c6aa62f4bd977246f1d4d2e5930.png"/>
</div>

- 资源列表在本地的存储格式

Service Worker 把静态资源缓存到本地缓存空间即 [Caches](https://developer.mozilla.org/en-US/docs/Web/API/Cache)，他会缓存请求的 `Request` 和 `Response` 对象，如下图所示：
 <div align=center>
<img src="https://p6.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13048690215/4f19/effc/5a24/96f44423c9280ee0afd86dce5a6c0ee2.png"/>
</div>

- 实际表现
 <div align=center>
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13048716240/6bac/341e/f617/1feedb8210f12e37437bc9b96780abc5.png"/>
</div>

 <div align=center>
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13050316024/4c29/558b/0fbe/ffdb494ffc70f310bf0b12773b646bdd.png"/>
</div>

可以看到页面访问的所有被预缓存的资源都被 Service Worker 处理过，是从本地缓存空间返回的。图中 `logo192.png` 之所以发起了2次请求是因为对图片资源应用了 `StaleWhileRevalidate` 缓存策略： 即首次从本地返回随后发起请求更新资源。关于缓存策略这里就不展开了，这里对各种[缓存策略](https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-strategies)有详细的介绍。

上面通过一个简单的 demo 介绍了如何应用 Service Worker；当然 demo 和实际的业务场景中应用差距还是很大的，下面就介绍我们是如何在业务中落地的。


# 4、缓存静态资源方案
为了便于大家理解，在开始介绍我们的优化方案之前我觉得有必要先跟大家介绍一下目前我们移动站的业务场景以及系统架构。

#### 业务场景
由于我们业务的特殊性，我们一个工程内的页面分布在**云音乐**以及**音街**两个 App 内，还有剩下一部分是站外的落地页，通常在各类浏览器以及微信里传播，如下图所示：
 <div align=center>
<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12480977220/0b6f/63d8/9aa3/baf83a116fd0fc7d7a53521cf8ecc3e5.png"/>
</div>

#### 应用架构
我们系统架构有如下几个特点：
  - React 同构应用
  - `SSG`
  - 支持 `SSR` 降级 `SSG`
  - 静态资源 CDN 缓存 (包括html)


#### 构建流程
由于上述说的业务场景的特殊性且也是为了方便管理，我们对路由也进行了拆分，最后对每个路由进行了 `SSG` 处理，生成对应的 html 资源，主要给 CSR 的页面以及 SSR 降级的页面使用。

 <div align=center>
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12472053927/afc1/d109/0cac/f80245951ea130781c96afd45bee51a4.png"/>
</div>

上面简单介绍了我们的业务场景以及应用架构，我们的场景其实是比较适合应用 Service Worker 的，因为我们有 `SSG`，所以每个 html 都有页面基本的结构，所以如果缓存在本地收益是比较好的，用户会更快看到页面内容。

在介绍完业务场景、系统架构以及构建流程后，接下来介绍我们基于当前的场景设计缓存静态资源的方案。可能有的同学会疑问缓存静态资源需要这么麻烦吗？一股脑全都预缓存不就完事了，如果你的应用很简单，几个路由，构建资源也不是很多我觉得问题不大。但是我们业务场景比较复杂，页面也很多。如果一次性缓存会带来两个问题：
 - 一是如果我只是云音乐用户缓存其他 app 里面的页面和资源是没有意义的，同时也会导致用户本地缓存过大；
 - 二是预缓存资源体积大会增加 Service Worker 的安装失败率，所以这是不可接受的。

针对这个问题我们的方案是对不同场景注册不同的 **`sw.[env].js`**，它只预缓存当前环境相关的静态资源，相关代码如下：

#### 分环境注册 SW
```js
// registerSw.js
const registerSw = () => {
    const register = async () => {
        // 不支持 sw，直接退出
        if (!('serviceWorker' in navigator)) {
            log('sw-notsupport');

            return;
        }

        let scriptName;

        // 当前环境云音乐
        if (Env.isInNEMapp()) {
            scriptName = '/sw.music.js';
        // 当前环境音街
        } else if (Env.isInKSapp()) {
            scriptName = '/sw.ksong.js';
        // 站外
        } else {
            scriptName = '/sw.h5.js';
        }

        try {
            await navigator.serviceWorker.register(scriptName);
            log('sw-success');
        } catch (error) {
            log('sw-error', error);
        }
    };

    if (/loaded|complete/.test(document.readyState)) {
        register();
    } else {
        window.addEventListener('load', register);
    }
};
```


#### 依赖收集
上面明确了根据当前不同的环境缓存相关的静态资源，所以我们需要收集不同环境下依赖的所有静态资源，目前我们只缓存 js 以及 SSG 输出的 html 资源。首先我们需要对路由文件改造，因为我们需要知道某个路由是在哪个环境中使用的。

```js
const routes = [
    // 只在音街 app 中使用
    {
        path: '/app/user/taskcenter/v1',
        exact: true,
        // 标识当前路由使用环境
        app: ['ksong'],
        component: loadable(() => import('@views/app/task-center')),
    },

    // 只在云音乐 app 中使用
    {
        path: '/app/picksong/musicplaylist',
        exact: true,
        app: ['music'],
        component: loadable(() => import('@views/app/pick-song/music-playlist')),
    },

    // 在两个 app 中都使用
    {
        path: '/app/picksong/playlist',
        exact: true,
        app: ['ksong', 'music'],
        component: loadable(() => import('@views/app/pick-song/playlist')),
    },
]
```
如上图我们在每个路由中都增加了一个字段 `app` 标识这个路由是在哪个环境中使用。接下来我们看看如何收集路由依赖的 js。

- 收集 js

收集 js 其实比较简单，我们可以遍历 route 文件，构建出每个路由的 html，然后从中提取出 js 然后合并去重（因为 vender 和 runtime 是页面间公用的）就可以了。

收集页面依赖 js 代码如下：
```js
// collect-deps.js
const getDeps = async (url, app) => {
    /**
    * 省略一些代码
    */
    renderToString(
        extractor.collectChunks(
            Component({
                url,
            })
        )
    );

    const scriptTags = extractor.getScriptTags();

    return scriptTags.match(/(?<=src=")(.+)(?=")/g).map((item) => item.replace(publicPath, ''));
};

const collectDeps = async (routes, app) => {
    const result = [];

    try {
        for (const route of routes) {
            const url = route.path;

            // eslint-disable-next-line no-await-in-loop
            const deps = await getDeps(url, app);

            result.push(...deps);
        }
    } catch (error) {
        console.log(error);
        process.exit(1);
    }

    return Array.from(new Set(result));
};
```

- 收集 html

缓存 html 会有点麻烦，可能大家会说直接使用构建流程中生成的 html 就可以。但是这样有个问题就是我们缓存的 html 可能是旧版本的，因为我们除了 SSR 的路由都是上了 CDN 缓存的。举个例子，`https://domain/download`，这个页面是 CSR 渲染的，所以平时我们访问他的内容应该是从 CDN 节点返回的，然后我们缓存 `https://domain/download` 页面的 html 到本地，这个时候是没什么问题的，但是一旦这个页面在某个版本中被修改了上线后，Service Worker 重新预缓存这个路由的 html 到本地，它这个时候很有可能就是从 CDN 节点中拿到旧版本的内容，这样就导致了不能及时缓存最新版本的页面。
用下图简单说明这个问题：
 <div align=center>
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12502251563/b0e3/d28f/1beb/afce09348a3db54df385f65b239f678f.png"/>
</div>

- 1 这个流程表示首次方案页面，页面安装 Service Worker 然后预缓存 /download/ 页面的 html 到本地缓存，同时 CDN 节点也会缓存 html；
- 2 这条流程表示 Service Worker 安装成功、激活后用户访问页面，然后 /download/ 路由的请求被 Service Worker 拦截使用本地缓存的 html 响应，这一步没有问题。
- 3 这条流程表示当新版本发布后，新版的 Service Worker 重新预缓存 /download/ 页面的 html，但是这时候拿到的内容很有可能就是 CDN 节点上缓存的 html，导致 Service Worker 缓存到本地的 html，一直是旧的！

我们知道缓存 js 文件不会有这样的问题，因为 js 的路径中都包含了 **`contenthash`**，所以 Service Worker 缓存的永远都是最新的文件，那么最容易想到的解决方案就是给预缓存的 html 路径中也带上 **`contenthash`**。我们现在只需要在已经构建出来的 static html 资源基础上，再生成一份带 **`contenthash`** 的 html 提供给 Service Worker 预缓存使用。如下图所示：

 <div align=center>
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12509990106/2536/8da9/1c3b/5ef7d67cd08f920a9d1d9b9eff4edb74.png"/>
</div>

所以 Service Worker 缓存 html 到本地 `Caches` 就应该是:
```js
{
    [pathname.[contenthash].html] : content,
}
``` 

但是 Service Worker 从本地 Caches 查询的时候需要额外做一层映射，因为我们访问的路由是 `https://domain/download`, 缓存到本地的请求是 `https://domain/download.[contenthash].html`，所以需要有一层请求的路由到实际缓存的路由的映射关系，这层关系我们可以在生成预缓存 html 的时候处理好，然后注入到 Service Worker 文件中。

```js
// generate-precache-html.js
const writeCacheHtml = async (url) => {
    const content = await readFile(url);

    if (!content) {
        return '';
    }

    // 生成 hash
    const hash = crypto.createHash('md4').update(content).digest('hex').substr(0, 8);

    // 输出到 cache 目录提供给 sw 预缓存
    const filePath = path.join(distDir, `sw-cache/${url}`, `${hash}.html`);

    await fs.outputFile(filePath, newContent);

    // 返回相对路径
    return path.relative(distDir, filePath);
};

/**
 * 前端路由与预缓存 html 的对应关系, 提供给 sw 用来映射缓存的 html
 */
const generatePrecacheHtml = async (routes) => {
    // 存储映射关系
    const result = {};

    try {
        for (const route of routes) {
            const url = route.path;

            // eslint-disable-next-line no-await-in-loop
            const cacheHtmlPath = await writeCacheHtml(url);

            if (cacheHtmlPath) {
                result[url] = cacheHtmlPath;
            }
        }
    } catch (error) {
        process.exit(1);
    }

    return result;
};
```

#### Service Worker 构建流程
把上面的步骤整合在一起就是 Service Worker 缓存静态资源的核心流程如下：

```js
(async () => {
    try {
        for (const app of ['music', 'ksong', 'h5']) {
            const swFilePath = path.join(__dirname, `../../ksongmobile/sw.${app}.js`);

            /**
             * 根据 app 拿到全部路由
             */
            const routes = getRoutesByApp(app);

            /**
             * js 文件列表
             */
            const assets = await collectDeps(routes, app);

            /**
             * 文件的路由映射关系
             */
            const htmlRouteMap = await generatePrecacheHtml(routes);

            /**
             * 文件修改
             */
            await inject({
                swSrc: swFilePath,
                injectionPoint: 'self.__ROUTE_TO_CACHE_PATH__',
                content: routeMap,
            });

            const manifest = [...Object.values(htmlRouteMap), ...assets];

            await inject({
                swSrc: swFilePath,
                injectionPoint: 'self.__ASSETS_MANIFEST__',
                content: manifest,
            });
        }
    } catch (error) {
        console.log(error);
        process.exit(1);
    }

    process.exit(0);
})();
```

整个流程看起来是非常清晰的，主要是获取预缓存的 js、html，以及 html 资源的路由与缓存的映射关系，最后向 Service Worker 文件注入上述两类资源。

#### Service Worker 内部逻辑

可以看到 Service Worker 内部的逻辑十分简单，我们只需要注入预缓存资源列表以及 html 路由与本地预缓存路由的映射关系。
```js
import { precacheAndRoute } from 'workbox-precaching';

/**
 * 根据环境注入需要缓存的资源列表，从 PRECACHES_FILE_LIST 过滤出最终需要缓存的资源
 */
const PRECACHES_FILE_LIST = self.__ASSETS_MANIFEST__;

/**
 * 访问的 html 路由与本地预缓存路由的映射关系
 */
const ROUTE_TO_CACHE_PATH = self.__ROUTE_TO_CACHE_PATH__;

/**
 * 预缓存资源
 */
precacheAndRoute(PRECACHES_FILE_LIST, {
    directoryIndex: null,
    urlManipulation: ({ url }) => {
        const pathname = url.pathname;

        const cachePath = ROUTE_TO_CACHE_PATH[pathname];

        // 命中用本地预缓存的 html 响应
        if (cachePath) {
            return [new URL(cachePath, location.href)];
        }

        return [url];
    },
});
```

通过上面的构建流程最后我们拿到三份 `sw.[env].js`，每份的区别就是 `PRECACHES_FILE_LIST` 、`ROUTE_TO_CACHE_PATH`, 最后在应用入口根据环境注册相应的 `Service Worker` 就可以了。
# 5、实际效果
#### 单个页面效果对比
##### 使用前
- html 文档加载时间
 <div align=center>
<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12469003111/a34b/5d79/44b9/5b278ff92ea13e9388423953ce6217a4.png"/>
</div>

- 整个页面加载时间
 <div align=center>
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12469128645/746e/54fc/a920/bf83fc9cd3ce7365caca6ffb049ded41.png"/>
</div>


##### 使用后
- html 文档加载时间
 <div align=center>
<img src="https://p6.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12469002381/f59a/9501/05eb/31dd0ac3e4cc64d6cd2816d14607a472.png"/>
</div>

- 整个页面加载时间
 <div align=center>
<img src="https://p5.music.126.net/obj/wo3DlcOGw6DClTvDisK1/12469213740/77e4/1d13/ed64/27ea0403f1298cee36f5ab9b66b2b7e1.png"/>
</div>

从上面的对比中我们可以看到 html 文档的加载时间提升了 `96%`，整体页面的加载时间提升了 `69.4%`。

#### 整体容器启动耗时对比

- 使用前
 <div align=center>
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13200277262/24b1/4fe4/61f2/431442767bc26af68c421e211205dc03.png"/>
</div>

- 使用后
 <div align=center>
<img src="https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13200277264/f28b/fa5c/9864/d74c3014b6cdd30e2f11c99b29aff1d2.png"/>
</div>

从数据上看其中 【DNS、SSL、请求响应】 能看到有明显降低。

# 6、总结与思考

#### WKWebView 对 Service Worker 的支持
从 iOS14 开始如果要在 WKWebView 中使用 Service Worker 需要配置域名白名单，在名单内的域名才可以启用 Service Worker，可参考下列文章：[app-bound-domains](https://webkit.org/blog/10882/app-bound-domains/)、[service-workers](https://firt.dev/ios-14/#service-workers)
#### 关于使用场景
如果大家是想使用 Service Worker 来做性能方面的优化这里有几个小提醒看看自己的站点是不是真的适合：
- 多页应用
- `SSG`
- 常规的前端优化手段都已经做过了且当前站点性能已经比较不错

之所以这样说是因为如果缓存的 html 里面没有任何内容，只有一个根节点，页面内容都要在客户端构建我觉得用户体验也不会很好；从我们做前端性能优化的经验来看，Service Worker 的优先级不是最高的，因为还有很多其他的前端优化技巧比 Service Worker 来的更有效果，所以说常规的性能优化手段都做过了可以尝试 Service Worker。

#### 关于离线包
现在很多公司内部都有[离线包](https://juejin.cn/post/6844904031773523976)的方案，常规方案的实现是 app 启动后下载、更新离线包资源到客户端本地，然后 webview 拦截对静态资源的请求从而返回本地缓存的资源。这个过程跟 Service Worker 离线访问的原理是一致的而且 Service Worker 是一个 Web 标准，所以随着标准的不断完善、各浏览器厂商的跟进，传统的离线包是不是可以退出历史舞台。

# 感谢阅读

> 本文发布自 [网易云音乐大前端团队](https://github.com/x-orpheus)，文章未经授权禁止任何形式的转载。我们常年招收前端、iOS、Android，如果你准备换工作，又恰好喜欢云音乐，那就加入我们 grp.music-fe(at)corp.netease.com！
