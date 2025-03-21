# Vite & Vue3 Typescript版的single-spa微前端实战 - 前端项目拆分实践

本文为Markdown格式，效果不好的话，请访问https://www.cnblogs.com/hualei/articles/18778924; 或https://github.com/topabomb/micro-frontends-single-spa-demo 查看；

前端项目由于功能繁复且多次迭代，单项目的模式会越来越大，严重影响打包效率和加载时间；将大的前端项目拆分为多个子项目app（本文称为remote项目），每个团队可以单独负责一个或多个子项目，通过root项目进行组合，实现逻辑及代码分离；详细的微前端论文和介绍请访问(https://micro-frontends.org/)

本文可能是最详细的通过single-spa实现微前端实战教程，single-spa支持多种异构的项目组合，但由于笔者比较熟悉vue环境，本位的root及remote均由vite+vue3+typescript实现，使用其他技术栈可参考本文的思路自行实践；

single-spa的呈现形式实际还是非iframe的单页应用程序（spa），故remote与root的分离还需考虑样式的隔离，部分依赖于全局的事件和方法也可能需要进行调整，请注意；

类似的微前端实现方式还有[vite-plugin-federation](https://github.com/originjs/vite-plugin-federation)，笔者同样进行了实践，single-spa更侧重异构的整合及remote的完整生命周期管理，vite-plugin-federation侧重于组件级的整合；css隔离及全局事件方法等依然需要开发者自行注意；笔者暂时选择single-spa的原因是root及各remote可以做到几乎完全的独立，同时通过root进行路由规划，亦可以完整继承全部项目的路由，而vite-plugin-federation偏向组件重用，在路由规划上需要额外的实现；将来有机会，也会分享vite-plugin-federation的实践过程；

本文及演示项目github仓库：
- 本文(https://github.com/topabomb/micro-frontends-single-spa-demo)
- root(https://github.com/topabomb/micro-frontends-demo-root)
- remote(app1)(https://github.com/topabomb/micro-frontends-demo-app1)
> 仓库代码是根据本文同步创建的，从vue的模板做的调整已极致简化了，亦可作为single-spa微前端的模板项目使用；

## Root 项目

root项目是前端的统一的入口，用于remote的加载并管理其生命周期，single-spa官方更加推荐原生的实践，但在实际项目中，root项目应该也会有一些通用或简单的内容呈现，故本文root项目同样采用vue，并在root项目中也有自己的路由和呈现内容；

### 创建VUE项目
```shell
npm create vue@latest
npm install
npm install single-spa -S
npm install vite-plugin-single-spa -D
```

依赖包介绍
- single-spa
微前端引擎，须在root项目中引入，用于加载remote项目并管理其生命周期；
- vite-plugin-single-spa
vite的single-spa插件，用于vite环境下的项目结构组织及vite打包的额外处理
### Vite配置
- ./vite.config.ts

>引入vite-plugin-single-spa并启用该vite插件，注意type必须设置为root；importMaps这里配置的是插件默认值，用于理解引入插件后的整体项目结构；

>imo用于指定single-spa官方的[import-map-overrides](https://github.com/single-spa/import-map-overrides)包的加载方式，这里使用imo:true使用JSDelivr托管的最新版本，实际项目中建议配置为特定url，例如
```ts 
imo: () => `https://my.cdn.example.com/import-map-overrides@4.2.0`
```
> 固定端口在root项目中不是必须的，但在remote项目中需要固化，后续我们可以看到在root项目中静态约定了remote的端口


```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueDevTools from 'vite-plugin-vue-devtools'
import viteSingleSpa from 'vite-plugin-single-spa'
const port = 4120
export default defineConfig({
  plugins: [
    vue(),
    vueDevTools(),
    viteSingleSpa({
      type: 'root',
      imo: true,
      importMaps: { dev: 'src/importMap.dev.json', build: 'src/importMap.json' },
    }),
  ],
  preview: { port },
  server: { port },
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})
```
- remote配置

    > 我们在前面指定了importMaps的两个不同版本，由于dev环境中main.js的路径在src下，所以需要分开配置；在开发环境中，我们使用本地端口来区分不同的remote项目加载点，而在生产环境中应由nginx或caddy分别路由到两个不同的目录或者分离两个站点

    - ./src/importMap.json

    ```json
    {
    "imports": {
        "@topabomb/app1": "http://localhost:4101/main.js",
        "@topabomb/app2": "http://localhost:4102/main.js"
        }
    }
    ```

    - ./src/importMap.dev.json

    ```json
    {
    "imports": {
        "@topabomb/app1": "http://localhost:4101/src/main.js",
        "@topabomb/app2": "http://localhost:4102/src/main.js"
        }
    }
    ```
### 启动Single-spa

前面都是在vite中的配置，实际需要在项目上下文代码中激活Single-spa的环境；
- ./src/single-spa.setup.ts
> 注意apps的配置，与上面importMaps中的命名需要对应；registerSpas方法将在main.ts中使用，启动Single-spa的上下文环境

> customProps中的basePath用于remote与root项目的路由分离，container将指定的容器元素传递给remote
``` typescript
import { registerApplication, start } from 'single-spa'
export const apps = {
  app1: '@topabomb/app1',
  app2: '@topabomb/app2',
}
export function registerSpas(container:HTMLElement) {
  for (const [route, moduleName] of Object.entries(apps)) {
    registerApplication({
      name: route,
      app: () => import(/* @vite-ignore */ moduleName),
      activeWhen: `/${route}`,
      customProps: {
        container,
        basePath: `/${route}`,
      },
    })
  }
  runSpas()
}
export function runSpas() {
  start()
}
```
- ./src/main.ts
> 此处几乎全部是vue模板的内容，额外registerSpas激活了single-spa环境，注意传递了remote_container作为remote渲染的容器;
``` typescript
import './assets/main.css'
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import { registerSpas } from './single-spa.setup.ts'
const app = createApp(App)
app.use(createPinia())
app.use(router)
app.mount('#app')
registerSpas(document.getElementById("remote_container")!)
```

### 在root项目的合适位置展示Remote的路由

由于我们直接使用了vue的默认模板，直接更改App.vue文件增加加载app1与app2的导航，通过前面的配置我们指定了remote将在remote_container中加载
- ./src/App.vue
```vue
...
<RouterLink to="/app1">App1</RouterLink>
<RouterLink to="/app2">App2</RouterLink>
...
<div id="remote_container"></div>
...
```

- 运行项目

导航处点击App1或App2，在开发者工具的网络请求中应该可以看到main.ts的加载失败了，这是因为我们还没有启动remote的app1和app2；
```
npm run dev
``` 

## Remote项目-被root所集成的app1项目

### 创建VUE项目
```shell
npm create vue@latest
npm install
npm install single-spa-vue -S
npm install vite-plugin-single-spa -D
```

依赖包介绍
- vite-plugin-single-spa
vite的single-spa插件，用于vite环境下的项目结构组织及vite打包的额外处理
- single-spa-vue
利用这个插件简化remote的代码实现；
- 这不是root项目，remote并没有的single-spa依赖
### Vite配置

- ./vite.config.ts

>引入vite-plugin-single-spa并启用该vite插件，注意type必须设置为mife(默认值),serverPort跟server、preview指定为同样的4101，避免动态端口的干扰，生产环境根据实际的架构配置serverPort可能需要调整；spaEntryPoints指向打包文件的入口点,默认值为src/spa.ts，这里Vue的模板项目入口点为src/main.ts，为了入口统一，故不使用默认值；

>注意projectId，主要用于区分不同的remote中的集成样式代码等，默认值为package.json中的name字段，此处单独指定是为了避免多个不同的remote中使用了同样的package.json的name；但此处的projectId跟root项目中的remote并没有相关性；
```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueDevTools from 'vite-plugin-vue-devtools'
import vitePluginSingleSpa from 'vite-plugin-single-spa'
const port = 4101
export default defineConfig({
  plugins: [
    vue(),
    vueDevTools(),
    vitePluginSingleSpa({
      type: 'mife',
      serverPort: port,
      spaEntryPoints: 'src/main.ts',
      projectId: 'app1',
    }),
  ],
  server: { port },
  preview: { port },
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
})

```

### Vue环境的代码调整
本文演示中Root与Remote都有自身的路由实现，Root中也针对自身及remote做了路由层面的整合，Remote中无需路由功能的情况可跳过下节;
- ./src/router/index.ts
> 此处路由实现直接保留了vue模板的路由实现，注释部分为模板的原有代码，可以看到修改点增加了basePath参数；
```typescript
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'
//const router = createRouter({
//history: createWebHistory(import.meta.env.BASE_URL),
const router = (basePath: string) =>
  createRouter({
    history: createWebHistory(basePath),
    routes: [
      {
        path: '/',
        name: 'home',
        component: HomeView,
      },
      {
        path: '/about',
        name: 'about',
        component: () => import('../views/AboutView.vue'),
      },
    ],
  })
export default router

```
- ./src/main.ts
> 注意，原有vue模板的逻辑封装到了mountVue方法，同时路由组件的使用也使用了跟模板相同的import.meta.env.BASE_URL参数，在development模式下，该项目跟往常一样独立运行，并不能被root集成；
```typescript
import './assets/main.css'
import { createApp, h } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import singleSpaVue, { type SingleSpaProps } from 'single-spa-vue'
type Props = {
  name: string
  container: HTMLElement
  basePath: string
}
const vueLifecycles = singleSpaVue({
  createApp,
  appOptions: {
    render() {
      const { name, container, basePath } = this as unknown as Props
      //h的额外参数非必须，这里演示把host的参数传递给remote的根元素
      return h(App, {
        remoteName: name,
        mountContainer:container.id,
        remotePath: basePath,
      })
    },
  },
  handleInstance: (app, props: Props) => {
    //sspa的入口，同样需要跟mountVue一样app.use相关的组件，但这里没有app.mount
    app.use(createPinia())
    app.use(router(props.basePath))
  },
})
//vue模板项目的原始逻辑被封装在这个函数
const mountVue = () => {
  const app = createApp(App)
  app.use(createPinia())
  app.use(router(import.meta.env.BASE_URL))
  app.mount('#app')
}
if (import.meta.env.MODE === 'development') {
  mountVue()
}
export const bootstrap = vueLifecycles.bootstrap
export const mount = async (prop: SingleSpaProps) => {
  //在这里指定domElement为root传递过来的元素
  vueLifecycles
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    .mount({ domElement: (prop as unknown as any).container, ...prop })
    .then((v) => {
      console.log(`mount event:`,v)
    })
    .catch((e) => {
      console.error(`mount error:`,e)
    })
}
export const unmount = vueLifecycles.unmount
```
### 运行方式修改
通过前面的修改，跟往常一样可以```npm run dev```来启动app1的开发环境；要该项目能够被root集成，我们需要增加新的启动脚本，在package.json的script增加dev:sspa和start两个指令：
```json
...
    "dev:sspa":"vite --mode staging",
    "start": "npm run build && npm run preview",
...
```
> root项目可以同样增加start指令方便在preview模式下启动项目；

现在我们可以试着启动remote项目，这时app1可以被root加载了；由于我们使用--mode staging，main.ts没有执行app.mount('#app')，所以直接访问该网站，会缺失ui渲染；
```shell
npm run dev:sspa
```

## 查看微前端的集成效果
root跟第一个remote(app1)都已经构建好了，我们可以查看实际的效果；
### 开发环境
- 启动root项目
```shell
npm run dev
```
- 启动remote(app1)项目
```shell
npm run sspa
```

打开root项目的站点，按本文的配置是：http://localhost:4120/； 点击App1路由，可以看到app1项目被整体加载到指定的RouterView容器中，你还可以点击app1项目中的导航，可以发现app1中的内联导航也能正常跳转；
> 如果你跟笔者一样root跟remote都是vue创建的新项目，没有做任何的界面调整，那么也不要奇怪页面的组织方式，微前端的集成，需要预先设计好root的布局方式，让remote在合适的地方进行渲染；

### Preview(本机模拟生产)环境

- 启动root项目
为root项目的package.json的script增加start指令
```json
...
"start": "npm run build && npm run preview",
...
```
```shell
npm run start
```
- 启动remote(app1)项目
```shell
npm run start
```

打开root项目的站点，可以看到跟开发环境一样的效果和实现

## Remote项目-被root所集成的app2项目

到这里，app2的实现跟app1一样，要注意vite配置上的定制，本文就省略了，请读者自行实现；


## 可无视的安利环节
安利下笔者的其他开源项目
- [电池大师(BatteryMaster)](https://batterymaster.weero.net/)
> 免费且开源的Windows笔记本电池管理软件，支持电池健康度、损耗度、充电功率、放电功率、电池电压等等关键电池信息展示；手工调节处理器功率限制；可以记录并查看历史的电池健康度变化；

> 本文在hx370笔记本撰写，开启[电池大师](https://batterymaster.weero.net/)限制12w功耗，用时4小时30分，耗电53%，平均放电9.9w。还是有点效果；

- [MarkText中文特别版/汉化版](https://marktext.weero.net/)
> 全面中文化的MarkText特别版，针对Ebook与本地笔记增加了一些实用化的功能；

> 本文撰写全部使用[MarkText中文特别版/汉化版](https://marktext.weero.net/)，还算好用；

- [Topabomb的Github](https://github.com/topabomb/)

> 除了本文，最近还有deepseek_api_call的[样板工程开源](https://github.com/topabomb/deepseek_api_call)

- [乱七八糟的个人项目](https://www.weero.net/)

