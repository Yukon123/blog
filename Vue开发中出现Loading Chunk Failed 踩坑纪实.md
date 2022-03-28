---
theme: v-green
highlight: a11y-light
---
### 报错现象:

  某天测试反应在点击页签的时候出现了 **Loading Chunk Failed** 的错误，经过本人百度分析后判断是异步组件在发包时旧资源被替换的问题，然后一通CV操作之后发现问题还是存在，于是便有如下探究。

### 发生原因

1. 用户在发包前进入了页面（也就是请求到了 **index.html** ），并且在 **index.html** 中可以得知将来要请求的异步组件的名字叫 **a.js** ，当服务器这时候发包，并且清空掉了 **a.js** 这个资源，改名叫 **a1.js** 。发包之后用户点击 **a.js** 对应的组件时，浏览器拿着先前在 **index.html** 得知的 **a.js** 这个名字去服务器请求资源就得到了以上的 **Loading Chunk Failed** 报错。

2. 正常的生产上线流程可能存在静态资源和页面分属不同服务器,应该是先全量部署静态资源（各种js，css，图片），不清空旧资源，然后再部署页面。但如果清空掉旧资源就可能导致报错。

3. 如果在测试环境中可能会采取清空覆盖掉旧资源，这个时候就必须要前端进行控制了。

### 解决思路

-   在监听到路由报错的时候，前端强制刷新页面，重新获取index.html和对应的静态资源路径。
-   设置**preFetch**，网络空闲的时候就请求资源，可以大幅降低报错的几率。

### 触发bug

想要解决问题首先就是得复现问题，涉及到发包上线的测试和验证都有点小尴尬，因此提供下个人思路

-   最直接的就是你开个页面，然后控制台网络禁用掉缓存，然后发包后进入其他异步组件触发bug。
-   如果想要触发 **onError** 这个钩子的话，直接断开 **devServer** 就可以了。
-   本地复现的话就是开个本地服务器，然后进入页面，把 **dist** 文件夹中对应的js文件删去即可触发。

### 代码实现

```js
 /* 正则使用'\S'而不是'\d' 为了适配写魔法注释的朋友，写'\d'遇到魔法注释就匹配不成功了。
  * 使用reload方法而不是replace原因是replace还是去请求之前的js文件，会导致循环报错。
  * reload会刷新页面， 请求最新的index.html以及最新的js路径。
  * 直接修改location.href或使用location.assign或location.replace，和router.replace同理， 
  * 在当前场景中请求的依然是原来的js文件，区别仅有浏览器的历史栈。因此必须采用reload.
  * reload()有个特点是当你在A页面试图进入B页面的时候报错，会在A页面刷新，因此在刷新后需要手动书写逻辑
  * 进入B页面，可以在router.onReady()方法里面书写
  * 为了避免在特殊情况下服务器丢失资源导致无限报错刷新，做了一步控制，仅尝试一次进入B页面，
  * 如果不成功就只刷新A页面，停留在当前的A页面。
  */
 
 
 router.onError((error) => {
   const jsPattern = /Loading chunk (\S)+ failed/g
   const cssPattern = /Loading CSS chunk (\S)+ failed/g
   const isChunkLoadFailed = error.message.match(jsPattern || cssPattern)
   const targetPath = router.history.pending.fullPath
   if (isChunkLoadFailed) {
     localStorage.setItem('targetPath', targetPath)
     window.location.reload()
   }
 })
 
 router.onReady(() => {
   const targetPath = localStorage.getItem('targetPath')
   const tryReload = localStorage.getItem('tryReload')
   if (targetPath) {
     localStorage.removeItem('targetPath')
     if (!tryReload) {
       router.replace(targetPath)
       localStorage.setItem('tryReload', true)
     } else {
       localStorage.removeItem('tryReload')
     }
   }
 })
 
 
```

### 一点思考

我之前提到过**异步组件**而不是**路由懒加载**导致了这个问题的发生，有趣的是，当你子组件是用**懒加载**方式进行，并且没有设置或者关闭了**preFetch**，且之前没有缓存。很可能也会报这种错。在网络中和路由懒加载一样是报 404 找不到对应资源，区别就是不会报 **loadingChunkError** 的错误。

设置缓存策略中页面一定要对页面( **inedx.html** )设置**no cache no store**，避免依然指向旧的已被删除的资源，只有Get方式获取的资源才可以设置缓存策略。
