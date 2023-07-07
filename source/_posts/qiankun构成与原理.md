---
title: qiankun构成与原理
date: 2023-02-02T10:12:49Z
url: https://github.com/gwuhaolin/blog/issues/36
tags:
    - 微前端
---

## 功能介绍&模块拆解

在聊原理之前先了解下qiankun提供的能力，一句话介绍qiankun功能：
**能根据路由自动调度子应用并实现沙箱（主子、子子应用之间的JS和CSS）隔离。**

举个例子：在主应用里注册两个子应用A&B，

```jsx
import { registerMicroApps } from 'qiankun';

registerMicroApps([
  {
    name: 'A',
    entry: 'https://baidu.com',
    container: '#yourContainer',
    activeRule: '/baidu',
  },
  {
    name: 'B',
    entry: 'https://google.com',
    container: '#yourContainer2',
    activeRule: '/google',
  },
]);
```

在pathname是_/baidu_时加载和运行子应用A，如果路由pushState到了_/google_时会unmount子应用A再加载和运行子应用B。

从上面的例子可以看出qiankun提供的能力可以划分为3大部分：
- [single-spa](https://single-spa.js.org/): 绑定路由的子应用调度，监听路由变化根据当前路由去调度对应的子应用
- [import-html-entry](https://github.com/kuitos/import-html-entry): 加载和运行子应用的HTML entry
- [sanbox](about:blank):子应用的隔离，又分为JS的隔离和CSS的隔离


<img width="681" alt="image" src="https://user-images.githubusercontent.com/5773264/216295142-ca8af020-a4c1-4944-8956-37f09cde5b63.png">

下面分别拆解来分析原理。

## Single-spa

一句话介绍single-spa：**根据路由变化做子应用调度（子应用生命周期管理）**

<img width="712" alt="image" src="https://user-images.githubusercontent.com/5773264/216295303-62b3d1d1-4c6c-4acd-ba5e-ae8218591b35.png">


以单个子应用的生命周期来看流程如下：

<img width="611" alt="image" src="https://user-images.githubusercontent.com/5773264/216295373-d1f8affc-b2dc-4616-a68b-59b97991ab56.png">


### 注册子应用

```js
singleSpa.registerApplication({ // 注册一个子应用,注册其他子应用同理
    name: 'taobao', // 子应用名,需要唯一
    app: () => System.import('taobao'), // 如何拿到子应用的生命周期，这里demo用的System，qiankun实际上不是基于System
    activeWhen: '/appName', // url 匹配规则，表示啥时候开始走这个子应用的生命周期
    customProps: {} // 自定义 props，从子应用的 bootstrap, mount, unmount 回调可以拿到
})
```


### 子应用调度

子应用在自己的入口 js导出了生命周期函数钩子，那在切换路由时子应用A的unmount会执行，子应用B的mount会执行。针对React体系的子应用通常在各生命周期中做如下事情：

```js
/**
* bootstrap 只会在微应用初始化的时候调用一次，下次微应用重新进入时会直接调用 mount 钩子，不会再重复触发 bootstrap。
* 通常我们可以在这里做一些全局变量的初始化，比如不会在 unmount 阶段被销毁的应用级别的缓存等。
*/
export async function bootstrap() {
  console.log('react app bootstraped');
}

/**
* 应用每次进入都会调用 mount 方法，通常我们在这里触发应用的渲染方法
*/
export async function mount(props) {
  ReactDOM.render(<App />, props.container ? props.container.querySelector('#root') : document.getElementById('root'));
}

/**
* 应用每次 切出/卸载 会调用的方法，通常在这里我们会卸载微应用的应用实例
*/
export async function unmount(props) {
  ReactDOM.unmountComponentAtNode(
    props.container ? props.container.querySelector('#root') : document.getElementById('root'),
  );
}
```


### Html-entry-loader

一句话介绍html-entry-loader：**把HTML当作子应用的manifest，加载和执行其中的JS拿到JS导出的模块，加载拿到其中的CSS**。

<img width="971" alt="image" src="https://user-images.githubusercontent.com/5773264/216295684-4de11506-b1bc-4c1c-bc2f-e60ffa075096.png">


特别说明：
- 外链的JS、CSS会被fetch下来以方便执行或提取
- 默认不带沙箱隔离

```js
importHTML('https://xxx.com/subApp.html')
    .then(res => {
        res.execScripts(proxy = window /*传入沙箱，默认全局window*/).then(exports => {
            console.log(exports); // 导出的JS模块，子应用生命周期从这里拿
        });
        res.getExternalStyleSheets() // Promise<string[]> 获取导出的CSS内容
});
```

### 为啥选择HTML作为子应用manifest的描述载体

主应用需要拿到运行一个子应用所需要的信息，包括：JS、CSS、mountId。
除能用HTML作为载体之外，还有一种方式是通过一个JSON来描述，类似这样：

```json
{
  "version": "1.3.1",
  "js": ["main.js", "common.js"],
  "css": ["main.css"],
  "publicPath": "https://cdn.cn/appXXX",
  "mountId": "root"
}
```

这两种方式各有千秋：

优点 | 缺点
-- | --
完全兼容原来就是以HTML方式输出的网页更灵活，支持HTML中内敛的JS、CSS等复用已有的公知的HTML规范作为协议，而不是新创造一种协议 | 存在信息冗余，传输体积大于JSON解析HTML耗时大于解析JSON
协议更简单，传输和解析更快 | 操作了一种新协议、规范，需要接入方按此新规范去改造适配，推广成本上升不够灵活


选择HTML最主要的原因是 “复用已有的公知的HTML规范作为协议，而不是新创造一种协议”，因为一个子应用可能需要在多个站点投放、或者需要独立运行。


### 如何提取HTML中的JS、CSS

通过正则表达式提取，源码里枚举了各种可能的情况下的正则提取式：

```js
const ALL_SCRIPT_REGEX = /(<script[\s\S]*?>)[\s\S]*?<\/script>/gi;
const SCRIPT_TAG_REGEX = /<(script)\s+((?!type=('|")text\/ng-template\3).)*?>.*?<\/\1>/is;
const SCRIPT_SRC_REGEX = /.*\ssrc=('|")?([^>'"\s]+)/;
const SCRIPT_TYPE_REGEX = /.*\stype=('|")?([^>'"\s]+)/;
const SCRIPT_ENTRY_REGEX = /.*\sentry\s*.*/;
const SCRIPT_ASYNC_REGEX = /.*\sasync\s*.*/;
const SCRIPT_NO_MODULE_REGEX = /.*\snomodule\s*.*/;
const SCRIPT_MODULE_REGEX = /.*\stype=('|")?module('|")?\s*.*/;
const LINK_TAG_REGEX = /<(link)\s+.*?>/isg;
const STYLE_TAG_REGEX = /<style[^>]*>[\s\S]*?<\/style>/gi;
const STYLE_TYPE_REGEX = /\s+rel=('|")?stylesheet\1.*/;
const STYLE_HREF_REGEX = /.*\shref=('|")?([^>'"\s]+)/;
```

[JS、CSS提取逻辑完整源码链接](https://github.com/kuitos/import-html-entry/blob/master/src/process-tpl.js)


### 如何执行子应用

```js
function getExecutableScript(scriptSrc, scriptText, proxy, strictGlobal) {
	const sourceUrl = isInlineCode(scriptSrc) ? '' : `//# sourceURL=${scriptSrc}\n`;

	// 通过这种方式获取全局 window，因为 script 也是在全局作用域下运行的，所以我们通过 window.proxy 绑定时也必须确保绑定到全局 window 上
	// 否则在嵌套场景下， window.proxy 设置的是内层应用的 window，而代码其实是在全局作用域运行的，会导致闭包里的 window.proxy 取的是最外层的微应用的 proxy
	const globalWindow = (0, eval)('window');
	globalWindow.proxy = proxy;
	// TODO 通过 strictGlobal 方式切换 with 闭包，待 with 方式坑趟平后再合并
	return strictGlobal
		? `;(function(window, self, globalThis){with(window){;${scriptText}\n${sourceUrl}}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`
		: `;(function(window, self, globalThis){;${scriptText}\n${sourceUrl}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`;
}
```


#### with语句

with的初衷是为了避免冗余的对象调用：

```js
foo.bar.baz.x = 1;
foo.bar.baz.y = 2;
foo.bar.baz.z = 3;

with(foo.bar.baz){
    x = 1;
    y = 2;
    z = 3;
}
```

[使用with语句有很多问题，详情见](https://swordair.com/javascript-with-statement-in-depth/)


#### 如何执行子应用的JS

```js
const evalCache = {};

export function evalCode(scriptSrc, code) {
	const key = scriptSrc;
	if (!evalCache[key]) {
		const functionWrappedCode = `window.__TEMP_EVAL_FUNC__ = function(){${code}}`;
		(0, eval)(functionWrappedCode);
		evalCache[key] = window.__TEMP_EVAL_FUNC__;
		delete window.__TEMP_EVAL_FUNC__;
	}
	const evalFunc = evalCache[key];
	evalFunc.call(window);
}
```

- 用eval去执行以字符串形式保存的子应用JS
- 借助window.__TEMP_EVAL_FUNC__记下子应用JS的执行结果，并缓存到evalCache中避免下次重复eval

## Sanbox

### JS隔离

JS隔离是qiankun最核心最复杂的部分。JS隔离需要实现的目标是：
- 隔离是指对window修改进行隔离，封装污染window
- 避免子应用A污染主应用的window
- 避免子应用A污染子应用B的window

`
问：沙箱会做避免主应用对子应用A的window污染么？
答：不会，启动一个子应用时，子应用的window继承自主应用
`

#### 三种JS沙箱实现： SnapshotSandbox、LegacySandbox 、ProxySandbox


不同的JS沙箱实现 | 原理简介 | 优点 | 缺点 | 开启方法
-- | -- | -- | -- | --
[ProxySandbox](https://github.com/umijs/qiankun/blob/master/src/sandbox/proxySandbox.ts) | 基于Proxy API实现 | 隔离性和性能较好 | 浏览器兼容性问题，依赖无法polyfill的Proxy API | sanbox = true
[SnapshotSandbox](https://github.com/umijs/qiankun/blob/master/src/sandbox/snapshotSandbox.ts) | 基于diff算法实现 | 性能低，只支持单例子应用隔离作用有限 | 浏览器兼容性好，支持IE11 | 用于不支持 Proxy 的低版本浏览器降级使用
[LegacySandbox](https://github.com/umijs/qiankun/blob/master/src/sandbox/legacy/sandbox.ts) | 基于Proxy API实现，现已废弃不推荐使用 | 中间产物 | 中间产物 | singular = true

- qiankun会优先使用ProxySandbox，对于不兼容Proxy的浏览器会降级到SnapshotSandbox
- ProxySandbox支持同时有多个子应用沙箱运行，SnapshotSandbox无法保证同时有多个子应用时的隔离
- LegacySandbox时历史中间产物，现在已经没有存在的价值，所以废弃不推荐使用

#### ProxySandbox核心思想

拦截对`window`上字段的读&写，每个子应用一个沙箱(一个fakewindow)，子应用对window读&写实际是对fakewindow的读写。

- 一个map去存储子应用对window的修改记录，对window的写都会记录在内
- get时优先去map中读，找不到就去外层真实的window上读

<img width="741" alt="image" src="https://user-images.githubusercontent.com/5773264/216528857-99ebbef7-5fce-4528-8d85-c4e82ee0c922.png">

##### 如何拦截对window的读写

```js
const fakewindow = new ProxySandbox(); // 给子应用分配的代理window变量

((window) => {
    with(window){
      子应用代码
    }
})(fakewindow);
```

子应用代码中对window的读写，实际上变成了对subAppProxy的读写。

#### SnapshotSandbox核心思想

把主应用的 window 对象做浅拷贝，将 window 的键值对存成一个 Hash Map。之后无论微应用对 window 做任何改动，当要在恢复环境时，把这个 Hash Map 又应用到 window 上就可以了。
微应用 mount 时：

1. 先把上一次记录的变更 modifyPropsMap 应用到微应用的全局 window，没有则跳过
2. 浅复制主应用的 window key-value 快照 = mainWindowKV，用于下次恢复全局环境

微应用 unmount 时

1. 将当前微应用 window 的 key-value = microWindowKV 和 mainWindowKV 进行 Diff，Diff 出来的结果就是 modifyPropsMap
2. 将上次快照 mainWindowKV 拷贝到主应用的 window 上，以此恢复环境


#### JS沙箱逃逸

你的JS里有诸如 `document.body.appendChild(scriptElement)` 这样的代码，会动态往DOM里面插入JS，如果不处理这些JS会在主应用的 window 上执行可能污染真正的window。
为此，沙箱还会拦截appendChild方法，凡是子应用中appendChild进去的JS都会被fetch下来去沙箱里面执行。
![image](https://user-images.githubusercontent.com/5773264/216525816-8847cee2-a7fc-434d-a8d8-53b27e6ec83b.png)

https://github.com/umijs/qiankun/blob/master/src/sandbox/patchers/dynamicAppend/common.ts#L396

### CSS隔离

qiankun提供以下三种隔离样式的方式


CSS隔离实现方式 | 原理简介 | 优点 | 缺点 | 开启方法
-- | -- | -- | -- | --
CSS生命周期管理 | 子应用之间切换时，是会自动做子应用CSS的加载和卸载的，防止子应用A的CSS代入到子应用B中 | 无额外性能开销兼容性好 | 只能做子应用之间切换时的隔离，无法做主子、并发子的隔离 | 内置逻辑全程开启无法关闭
Scopted Style | 给子应用套一层特殊选择器的div修改子应用CSS选择器前缀 | 能做到主子、并发子的隔离 | 提升CSS选择器复杂性，降低页面性能 | experimentalStyleIsolation
Shadow DOM | 用Shadow DOM包裹 | 能做到主子、并发子的隔离 | 浏览器兼容性问题，依赖无法polyfill的Shadow DOM API子应用需要做一些适配 | strictStyleIsolation

#### 1. CSS生命周期管理
子应用之间切换时，是会自动做子应用CSS的加载和卸载的，防止子应用A的CSS代入到子应用B中。

#### 2. Scopted Style
<img width="942" alt="image" src="https://user-images.githubusercontent.com/5773264/216528122-5b00d617-3847-4b72-89fd-2c832e9b1116.png">


#### 3. Shadow DOM

用https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM 包裹子应用DOM区域，防止子应用DOM里面的CSS作用范围跑到子应用之外。

