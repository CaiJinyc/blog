---
title: 移动端字母索引导航
date: 2018-04-23
tags:
- JavaScript
- 移动端
---
**vue** + **better-scroll** 实现移动端歌手列表字母索引导航。算是一个学习笔记吧，写个笔记让自己了解的更加深入一点。<!-- more -->

Demo： [list-view](http://caijin.tech/demo/list-view/index.html#/)，使用 chrome 手机模式查看。换成手机模式之后，不能滑动的话，刷新一下就 OK 了。

Github：[移动端字母索引导航](https://github.com/CaiJinyc/demo/tree/master/list-view)

### 效果图
![](http://hexo-image.oss-cn-shenzhen.aliyuncs.com/18-5-2/19433704.jpg)

## 配置环境
因为用到的是 vue-cli 和 better-scroll，所以首先要安装 vue-cli，然后再 npm 安装 [better-scroll](https://ustbhuangyi.github.io/better-scroll/doc/zh-hans/installation.html#npm)。

简单介绍一下 better-scroll：
> better-scroll 是一款重点解决移动端（已支持 PC）各种滚动场景需求的插件。它的核心是借鉴的 iscroll 的实现，它的 API 设计基本兼容 iscroll，在 iscroll 的基础上又扩展了一些 feature 以及做了一些性能优化。
>
> better-scroll 是基于原生 JS 实现的，不依赖任何框架。它编译后的代码大小是 63kb，压缩后是 35kb，gzip 后仅有 9kb，是一款非常轻量的 JS lib。

除了这两，还使用 scss、vue-lazyload。scss 预处理器，大家都懂，用别的也一样。lazyload 实现懒加载，不用也可以，主要是优化一下体验。

数据直接使用了网易云的歌手榜单。

*CSS 样式我就不贴了，直接看源码就可以了。*
## 实现基本样式
直接使用 v-for 和 双侧嵌套实现歌手列表、以及右侧索引栏。

HTML 结构：
```html
<ul>
  <li v-for="group in singers" 
  class="list-group" 
  :key="group.id" 
  ref="listGroup">
    <h2 class="list-group-title">{{ group.title }}</h2>
    <ul>
      <li v-for="item in group.items" 
      class="list-group-item" :key="item.id">
        <img v-lazy="item.avatar" class="avatar">
        <span class="name">{{ item.name }}</span>
      </li>
    </ul>
  </li>
</ul>
<div class="list-shortcut">
  <ul>
    <li v-for="(item, index) in shortcutList"
    class="item"
    :data-index="index"
    :key="item.id"
    >
      {{ item }}
    </li>
  </ul>
</div>
```
shortcutList 是通过计算属性得到的，取 title 的第一个字符即可。

```js
shortcutList () {
  return this.singers.map((group) => {
    return group.title.substr(0, 1)
  })
}
```

## 使用 better-scroll
使用 better-scroll 实现滚动。对了，使用的时候别忘了用 import 引入。
```js
created () {
  // 初始化 better-scroll 必须要等 dom 加载完毕
  setTimeout(() => {
    this._initSrcoll()
  }, 20)
},
methods: {
  _initSrcoll () {
    console.log('didi')
    this.scroll = new BScroll(this.$refs.listView, {
      // 获取 scroll 事件，用来监听。
      probeType: 3
    })
  }
}
```
使用 created 方法进行 better-scroll 初始化，使用 setTimeout 是因为需要等到 DOM 加载完毕。不然 better-scroll 获取不到 dom 就会初始化失败。

*这里把方法写在两 methods 里面，这样就不会看起来很乱，直接调用就可以了。*

> 初始化的时候传入两 probeType: 3，解释一下：当 probeType 为 3 的时候，不仅在屏幕滑动的过程中，而且在 momentum 滚动动画运行过程中实时派发 scroll 事件。如果没有设置该值，其默认值为 0，即不派发 scroll 事件。

## 给索引添加点击事件和移动事件实现跳转
首先需要给索引绑定一个 touchstart 事件（当在屏幕上按下手指时触发），直接使用 v-on 就可以了。然后还需要给索引添加一个 data-index 这样就可以获取到索引的值，使用 `:data-index="index"`。
```html
<div class="list-shortcut">
  <ul>
    <li v-for="(item, index) in shortcutList"
    class="item"
    :data-index="index"
    :key="item.id"
    @touchstart="onShortcutStart"
    @touchmove.stop.prevent="onShortcutMove"
    >
      {{ item }}
    </li>
  </ul>
</div>
```
绑定一个 onShortcutStart 方法。实现点击索引跳转的功能。再绑定一个 onShortcutMove 方法，实现滑动跳转。

```js
created () {
  // 添加一个 touch 用于记录移动的属性
  this.touch = {}
  // 初始化 better-scroll 必须要等 dom 加载完毕
  setTimeout(() => {
    this._initSrcoll()
  }, 20)
},
methods: {
  _initSrcoll () {
    this.scroll = new BScroll(this.$refs.listView, {
      probeType: 3,
      click: true
    })
  },
  onShortcutStart (e) {
    // 获取到绑定的 index
    let index = e.target.getAttribute('data-index')
    // 使用 better-scroll 的 scrollToElement 方法实现跳转
    this.scroll.scrollToElement(this.$refs.listGroup[index])

    // 记录一下点击时候的 Y坐标 和 index
    let firstTouch = e.touches[0].pageY
    this.touch.y1 = firstTouch
    this.touch.anchorIndex = index
  },
  onShortcutMove (e) {
    // 再记录一下移动时候的 Y坐标，然后计算出移动了几个索引
    let touchMove = e.touches[0].pageY
    this.touch.y2 = touchMove
    
    // 这里的 16.7 是索引元素的高度
    let delta = Math.floor((this.touch.y2 - this.touch.y1) / 18)

    // 计算最后的位置
    // * 1 是因为 this.touch.anchorIndex 是字符串，用 * 1 偷懒的转化一下
    let index = this.touch.anchorIndex * 1 + delta
    this.scroll.scrollToElement(this.$refs.listGroup[index])
  }
}
```
这样就可以实现索引的功能了。

当然这样是不会满足我们的对不对，我们要加入炫酷的特效呀。比如索引高亮什么的~~

## 移动内容索引高亮
emmm，这个时候就有点复杂啦。但是有耐心就可以看懂滴。

我们需要 better-scroll 的 on 方法，返回内容滚动时候的 Y轴偏移值。所以在初始化 better-scroll 的时候需要添加一下代码。对了，别忘了在 data 中添加一个 scrollY，和 currentIndex （用来记录高亮索引的位置）因为我们需要监听，所以在 data 中添加。
```js
_initSrcoll () {
  this.scroll = new BScroll(this.$refs.listView, {
    probeType: 3,
    click: true
  })

  // 监听Y轴偏移的值
  this.scroll.on('scroll', (pos) => {
    this.scrollY = pos.y
  })
}
```

然后需要计算一下内容的高度，添加一个 calculateHeight() 方法，用来计算索引内容的高度。

```js
_calculateHeight () {
  this.listHeight = []
  const list = this.$refs.listGroup
  let height = 0
  this.listHeight.push(height)
  for (let i = 0; i < list.length; i++) {
    let item = list[i]
    height += item.clientHeight
    this.listHeight.push(height)
  }
}

// [0, 760, 1380, 1720, 2340, 2680, 2880, 3220, 3420, 3620, 3960, 4090, 4920, 5190, 5320, 5590, 5790, 5990, 6470, 7090, 7500, 7910, 8110, 8870]
// 得到这样的值
```

然后在 watch 中监听 scrollY，看代码：

```js
watch: {
  scrollY (newVal) {
    // 向下滑动的时候 newVal 是一个负数，所以当 newVal > 0 时，currentIndex 直接为 0
    if (newVal > 0) {
      this.currentIndex = 0
      return
    }
    
    // 计算 currentIndex 的值
    for (let i = 0; i < this.listHeight.length - 1; i++) {
      let height1 = this.listHeight[i]
      let height2 = this.listHeight[i + 1]

      if (-newVal >= height1 && -newVal < height2) {
        this.currentIndex = i
        return
      }
    }
    
    // 当超 -newVal > 最后一个高度的时候
    // 因为 this.listHeight 有头尾，所以需要 - 2
    this.currentIndex = this.listHeight.length - 2
  }
}
```

得到 currentIndex 的之后，在 html 中使用。

```html
给索引绑定 class -->  :class="{'current': currentIndex === index}"
```

最后再处理一下滑动索引的时候改变 currentIndex。

因为代码可以重复利用，且需要处理边界情况，所以就把

```js
this.scroll.scrollToElement(this.$refs.listGroup[index])
```

重新写了个函数，来减少代码量。

```js
// 在 scrollToElement 的时候，改变 scrollY，因为有 watch 所以就会计算出 currentIndex
scrollToElement (index) {
  // 处理边界情况
  // 因为 index 通过滑动距离计算出来的
  // 所以向上滑超过索引框框的时候就会 < 0，向上就会超过最大值
  if (index < 0) {
    return
  } else if (index > this.listHeight.length - 2) {
    index = this.listHeight.length - 2
  }
  // listHeight 是正的， 所以加个 -
  this.scrollY = -this.listHeight[index]
  this.scroll.scrollToElement(this.$refs.listGroup[index])
}
```

## lazyload
lazyload 插件也顺便说一下哈，增加一下用户体验。

**使用方法**  
1. 先 npm 安装
2. 在 main.js 中 import，然后 Vue.use

```js
import VueLazyload from 'vue-lazyload'

Vue.use(VueLazyload, {
  loading: require('./common/image/default.jpg')
})
```

添加一张 loading 图片，使用 webpack 的 require 获取图片。

3. 然后在需要使用的时候，把 `:src=""` 换成 `v-lazy=""` 就实现了图片懒加载的功能。

## 总结
移动端字母索引导航就这么实现啦，感觉还是很有难度的哈（对我来说）。

主要就是使用了 better-scroll 的 on 获取移动偏移值（实现高亮）、scrollToElement 跳转到相应的位置（实现跳转）。以及使用 touch 事件监听触摸，来获取开始的位置，以及滑动距离（计算最后的位置）。

### 收获
- 对 touch 事件有了了解
- 对 better-scroll 的使用熟练了一点
- vue 也熟练也一点啦，emmm
- 以后再写这样的东西就有经验啦