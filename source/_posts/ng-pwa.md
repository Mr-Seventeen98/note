---
title: Angular Service Worker - PWA
---
[完整代码](https://github.com/Mr-Seventeen98/ngpwa)
使用 Angular Service Worker 和 Angular CLI 内置的 PWA 支持，将 Web 应用程序转换成可下载和安装的移动应用程序。

在此过程中，将了解 Angular Service Worker 的设计及其背后的工作原理，并了解它的工作方式与其他构建时生成 Service Worker 的不同。

## 目录

在这篇文章中，我们将讨论以下主题：

- 使用 Angular CLI 搭建 Angular PWA 应用程序
- 了解如何手动添加 Angular PWA 支持
- 了解 Angular Service Worker 运行时缓存机制
- 运行和理解 PWA 的构建
- 在生产环境下启动 Angular PWA
- 版本管理

### 搭建 Angular PWA
创建 Angular PWA 的第一步是将 Angular CLI 升级到最新版本：
``` bash
npm install -g @angular/cli@latest
```
搭建一个 Angular 应用程序并添加 Angular Service Worker 相关pacakge
``` bash
ng new angular-pwa-app
```
将 Angular Service Worker 添加到现有应用程序中
`这里一定是用ng add 不能用npm i或者yarn add，因为ng add @angular/pwa 是angular cli 扩展的命令，会修改整个项目对PWA支持的文件，如果要使用 npm i 或者 yarn add的话，直接安装@angular/service-worker包，后面所有的文件改动都需要手动去修改`
``` bash
ng add @angular/pwa
```

### 了解如何手动添加 Angular PWA 支持
执行完第一步所有的操作后PWA的支持几乎已经被CLI处理完了。但我们需要知道它都改变了些什么，以防以后需要手动升级Angular应用或者是手动配置webpackage

- @angular/service-worker包被添加到package.json. angular.json配置文件中多出了一项`"serviceWorker": true`
``` json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "cli": {
    "analytics": false
  },
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "angular-pwa-app": {
      "projectType": "application",
      "schematics": {
        ...
      },
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "options": {
            "outputPath": "dist/angular-pwa-app",
            "index": "src/index.html",
            "main": "src/main.ts",
            "polyfills": "src/polyfills.ts",
            "tsConfig": "tsconfig.app.json",
            "inlineStyleLanguage": "sass",
            "assets": [
              ...
            ],
            "styles": [
              "src/styles.sass"
            ],
            "scripts": [],
            "serviceWorker": true, // 添加项
            "ngswConfigPath": "ngsw-config.json"
          },
          "configurations": {
            ...
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          ...
        },
        "extract-i18n": {
          "builder": "@angular-devkit/build-angular:extract-i18n",
          "options": {
            "browserTarget": "angular-pwa-app:build"
          }
        },
        "test": {
          ...
        }
      }
    }
  },
  "defaultProject": "angular-pwa-app"
}
```
`serviceWorker` 节点有什么用途？
将导致 `build` 后的 `dist` 文件夹中包含几个额外的文件：
  - `ngsw-worker.js`: Angular Service Worker 文件
  -  `ngsw.json`: Angular Service Worker 的运行时配置

- 应用程序 `AppModule` 中添加了 `ServiceWorkerModule` 的注册
```typescript
@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    ServiceWorkerModule.register('ngsw-worker.js', {
      enabled: environment.production,
      // Register the ServiceWorker as soon as the app is stable
      // or after 30 seconds (whichever comes first).
      registrationStrategy: 'registerWhenStable:30000'
    })
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```
`ServiceWorkerModule` 有什么用途？
这个模块提供了几个可注入的服务：
  - `SwUpdate` 用于管理应用程序版本更新
  - `SwPush` 用于执行服务器 Web 推送通知
该模块通过调用 navigator.serviceWorker.register() 在用户浏览器中加载 ngsw-worker.js，在浏览器中注册 Angular Service Worker（浏览器支持Service Worker的情况下）。
调用 register() 会导致 ngsw-worker.js 文件被加载到单独的 HTTP 请求中。有了这个，将我们的 Angular 应用程序转变为 PWA 只缺少一件事。

- 构建配置文件 ngsw-config.json
CLI 还添加了一个名为ngsw-config.json的配置文件，用于配置 Angular Service Worker 运行时参数，并且生成的文件带有一些默认值。
```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|apng|webp|avif|gif|otf|ttf|woff|woff2)"
        ]
      }
    }
  ]
}
```
此文件包含默认缓存行为或 Angular Service Worker，它针对应用程序静态资产文件：index.html、CSS 和 Javascript。

### Angular Service Worker 运行时缓存机制
Angular Service Worker 可以在浏览器[CacheStorage](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage)中缓存各种内容

这是一种基于Javascript的key/value缓存机制，与标准浏览器Cache-Control机制无关，两种机制可以分开使用。

assetGroups配置文件部分的目标是准确配置 Angular Service Worker 在缓存存储中缓存的 HTTP 请求，并且有两个缓存配置条目：
  - 一个名为 的条目app，用于所有单页应用程序文件（所有应用程序 index.html、CSS 和 Javascript 包以及图标）
  - 另一个名为 的条目assets，用于也包含在 dist 文件夹中的任何其他资产，例如图像，但不一定是运行每个页面所必需的

缓存静态文件是应用程序本身
另一方面，资产文件仅在被请求时才被缓存（意味着是惰性的），但如果它们曾经被请求过一次，并且有新版本可用，那么它们将被提前下载（意味着这是预更新模式）。

同样，对于在单独的 HTTP 请求中下载的任何资产（例如图像）来说，这是一个很好的策略，因为根据用户访问的页面，它们可能并不总是需要。

但是如果他们需要一次，那么我们很可能也需要更新版本，所以我们不妨提前下载新版本。

同样，这些是默认值，但我们可以调整它以适合我们自己的应用程序。但是，在应用程序文件的特定情况下，我们不太可能想使用其他策略。

毕竟，应用缓存配置是我们正在寻找的下载和安装功能本身。也许我们使用 CLI 生成的包之外的其他文件？在这种情况下，我们希望调整我们的配置。

重要的是要记住，有了这些默认设置，我们已经准备好了一个可下载和可安装的应用程序。

### 运行和理解 PWA的构建
首先向应用程序添加一些可视化内容，以清楚地标识在用户浏览器中运行的给定版本。例如，我们可以用以下内容替换app.component.html文件的内容：
``` html
<h1>Version V1 is runnning ...</h1>
```
现在让我们构建这个 PWA 应用程序。Angular Service Worker 仅在生产模式下可用，所以让我们首先对我们的应用程序进行生产构建：
``` json
ng build --prod
```

#### 构建后的文件
让我们看看我们的 build 文件夹中有什么，这里是生成的所有文件：
![1](http://jsnext.icu/20211114085812.png)
构建配置文件中的serviceWorker标志angular.json导致 Angular CLI 包含几个额外的文件

#### ngsw-worker.js文件
这个文件就是 Angular Service Worker 本身。与所有 Service Worker 一样，它通过自己单独的 HTTP 请求进行交付，以便浏览器可以跟踪它是否发生了变化，并将其应用于 Service Worker 生命周期

它将ServiceWorkerModule通过调用间接触发此文件的加载navigation.serviceWorker.register()。

然后，此文件将保持不变，直到您升级到包含新版本 Angular Service Worker 的新 Angular 版本。

#### ngsw.json文件
这是Angular Service Worker 将使用的运行时配置文件。该文件基于该文件构建ngsw-config.json，并包含 Angular Service Worker 在运行时了解它需要缓存哪些文件以及何时缓存所需的所有信息。
```json
{
  "configVersion": 1,
  "timestamp": 1636811800992,
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "updateMode": "prefetch",
      "cacheQueryOptions": {
        "ignoreVary": true
      },
      "urls": [
        "/favicon.ico",
        "/index.html",
        "/main.9b1590cf6a3f05bd.js",
        "/manifest.webmanifest",
        "/polyfills.d2940b6b864d5b0c.js",
        "/runtime.e6e2f4954d860551.js",
        "/styles.ef46db3751d8e999.css"
      ],
      "patterns": []
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "cacheQueryOptions": {
        "ignoreVary": true
      },
      "urls": [
        "/assets/icons/icon-128x128.png",
        "/assets/icons/icon-144x144.png",
        "/assets/icons/icon-152x152.png",
        "/assets/icons/icon-192x192.png",
        "/assets/icons/icon-384x384.png",
        "/assets/icons/icon-512x512.png",
        "/assets/icons/icon-72x72.png",
        "/assets/icons/icon-96x96.png"
      ],
      "patterns": []
    }
  ],
  "dataGroups": [],
  "hashTable": {
    "/assets/icons/icon-128x128.png": "dae3b6ed49bdaf4327b92531d4b5b4a5d30c7532",
    "/assets/icons/icon-144x144.png": "b0bd89982e08f9bd2b642928f5391915b74799a7",
    "/assets/icons/icon-152x152.png": "7479a9477815dfd9668d60f8b3b2fba709b91310",
    "/assets/icons/icon-192x192.png": "1abd80d431a237a853ce38147d8c63752f10933b",
    "/assets/icons/icon-384x384.png": "329749cd6393768d3131ed6304c136b1ca05f2fd",
    "/assets/icons/icon-512x512.png": "559d9c4318b45a1f2b10596bbb4c960fe521dbcc",
    "/assets/icons/icon-72x72.png": "c457e56089a36952cd67156f9996bc4ce54a5ed9",
    "/assets/icons/icon-96x96.png": "3914125a4b445bf111c5627875fc190f560daa41",
    "/favicon.ico": "22f6a4a3bcaafafb0254e0f2fa4ceb89e505e8b2",
    "/index.html": "2dddde2e959d509b5875df1affbbdeeca0013ef5",
    "/main.9b1590cf6a3f05bd.js": "a40c908f209a2e5ded532ecb14c6125fd64c6772",
    "/manifest.webmanifest": "ccd3e1a5912adcc16f66473d4cba3e30f2bc0f04",
    "/polyfills.d2940b6b864d5b0c.js": "48b7ab4dcc659f5bffded964098491e3504c8d66",
    "/runtime.e6e2f4954d860551.js": "8e240156b422208604399acf6a5c744f5468a071",
    "/styles.ef46db3751d8e999.css": "da39a3ee5e6b4b0d3255bfef95601890afd80709"
  },
  "navigationUrls": [
    {
      "positive": true,
      "regex": "^\\/.*$"
    },
    {
      "positive": false,
      "regex": "^\\/(?:.+\\/)?[^/]*\\.[^/]*$"
    },
    {
      "positive": false,
      "regex": "^\\/(?:.+\\/)?[^/]*__[^/]*$"
    },
    {
      "positive": false,
      "regex": "^\\/(?:.+\\/)?[^/]*__[^/]*\\/.*$"
    }
  ],
  "navigationRequestStrategy": "performance"
}
```
此文件是`ngsw-config.json`该文件的扩展版本，其中所有通配符 url 都已应用并替换为与之匹配的任何文件的路径。

#### Angular Service Worker 如何使用ngsw.json
Angular Service Worker 将在安装模式预取的情况下主动加载这些文件，或者在安装模式延迟的情况下根据需要加载这些文件，并且还将这些文件存储在缓存存储中

此加载将在后台进行，因为用户首先加载应用程序。下次用户刷新页面时，Angular Service Worker 将拦截 HTTP 请求，并将提供缓存文件而不是从网络获取它们。

每个资产在`hashTable`都有一个对应的哈希值。如果我们对此处列出的任何文件进行任何修改（即使它只有一个字符），我们将在以下 Angular CLI 构建中获得完全不同的哈希值。

然后 Angular Service Worker 将知道该文件在服务器上有一个新版本，需要在适当的时间加载。

### 生产环境下启动 Angular PWA
然后让我们以生产模式启动应用程序，为此，我们需要一个小型 Web 服务器。一个不错的选择是http-server，让我们安装它：
``` json
npm install -g http-server
```
然后让我们进入`dist/angular-pwa-app`文件夹，并以生产模式启动应用程序：
``` json
cd dist
http-server -c-1 .
```
-c-1选项将禁用服务器缓存，服务器通常会在 port 上运行8080，为应用程序的生产版本提供服务。
如果你有口8080被占用，应用程序可能运行在8081，8082等等，所使用的端口记录在启动时的控制台。
如果您在另一台服务器上本地运行 REST API，例如在端口 9000 中，您还可以使用以下命令将任何 REST API 调用代理到它：
``` json
http-server -c-1 --proxy http://localhost:9000 . 
```
在服务器运行后，让我们转到，看看我们使用 Chrome 开发工具运行了什么：http://localhost:8080

![2](http://jsnext.icu/20211114134901.png)

正如我们所看到的，我们现在已经运行了 V1 版本，并且我们已经安装了一个带有源文件的 Service Worker ngsw-worker.js

#### 静态资源存储位置
所有的 Javascript 和 CSS 文件，甚至包括所有的 Javascript 和 CSS 文件index.html都已在后台下载并安装在浏览器中以备后用

这些文件都可以在缓存存储中找到，使用 Chrome 开发工具：
![3](http://jsnext.icu/20211114135245.png)
Angular Service Worker 将在您下次加载页面时开始提供应用程序文件。或者点击刷新，您可能会注意到应用程序启动得更快。

#### 脱机访问
为了确认应用程序确实被下载到用户浏览器中，让我们做一个测试：按下 Ctrl+C 来关闭服务器。

现在让我们在关闭http-server进程后点击刷新：您可能会惊讶于应用程序仍在运行，我们得到了完全相同的屏幕！

在控制台上，我们会发现以下消息：

``` bash
An unknown error occurred when fetching the script.
ngsw-worker.js Failed to load resource: net::ERR_CONNECTION_REFUSED
```

唯一尝试从网络获取的文件是 Service Worker 文件本身，这是正常的。

### 版本管理
应用程序将静态文件都缓存到本存在一定的风险，当更新了代码发布了新版本的时候还是会从本地读取。

假设我们对应用程序进行了一些小的更改，例如编辑styles.scss文件中的全局样式。在再次运行生产构建之前，让我们保留 之前的版本ngsw.json，以便我们可以看到发生了什么变化。

现在让我们再次运行生产构建，并比较生成的ngsw.json文件：
![4](http://jsnext.icu/20211114235245.png)

正如我们所看到的，构建输出中唯一改变的是 CSS 包，所有剩余的文件都没有改变，除了index.html（加载新包的地方）

#### Angular Service Worker 如何更新版本
每次用户重新加载应用程序时，Angular Service Worker 都会检查ngsw.json服务器上是否有可用的新文件。

这是为了与标准 Service Worker 行为保持一致，并避免应用程序的陈旧版本长时间运行。陈旧的版本可能包含错误甚至完全损坏，因此必须经常检查服务器上是否有新的应用程序版本可用。

在我们的例子中，ngsw.json将比较文件的旧版本和新版本，并在后台下载并安装新的 CSS 包。

下次用户重新加载页面时，将显示新的应用程序版本！

#### 通知用户有新版本可用

对于用户可能已打开数小时的长时间运行的 SPA 应用程序，我们可能希望定期检查服务器上是否有应用程序的新版本并将其安装在后台。

为了检查新版本是否可用，我们可以使用该SwUpdate服务及其checkForUpdate()方法。

但一般来说，checkForUpdate()手动调用不是必需的，因为 Angular Service Worker 会ngsw.json在每次完整的应用程序重新加载时寻找新版本，以与标准 Service Worker 生命周期保持一致。

我们可以通过使用availableObservable来要求在新版本可用时得到通知SWUpdate，然后通过对话框询问用户是否想要获取新版本：

```typescript
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent  implements OnInit {

    constructor(private swUpdate: SwUpdate) {
    }

    ngOnInit() {

        if (this.swUpdate.isEnabled) {

            this.swUpdate.available.subscribe(() => {

                if(confirm("New version available. Load New Version?")) {

                    window.location.reload();
                }
            });
        }        
    }
}
```
当新的应用程序版本部署在服务器上时，让我们分解一下这段代码会发生什么：
  - 新文件现在在服务器上可用，例如新的 CSS 或 Js 
  - ngsw.json服务器上有一个新文件，其中包含有关新应用程序版本的信息：加载哪些文件，何时加载等。

这是正常的，因为用户仍然有一个 Service Worker 在浏览器中运行，它仍然为缓存存储中的所有文件提供服务，并且完全绕过网络。

但是，Angular Service Worker 也会调用服务器以查看是否有新的ngsw.json，并ngsw.json在后台触发加载文件中提到的任何新文件。

加载新应用程序版本的所有文件后，Angular Service Worker 将发出该available事件，这意味着新版本的应用程序可用。然后用户将看到以下内容：
![5](http://jsnext.icu/20211120535245.png)

如果用户单击`Ok`，则将重新加载完整的应用程序并显示新版本。请注意，如果我们没有向用户显示此对话框，则用户仍然会在下次重新加载时看到新版本。