# Drag,drag,drag！拽出哔哩哔哩侧边导航组件
![](https://user-gold-cdn.xitu.io/2020/4/3/1713ded20c3ac3cf?w=658&h=370&f=jpeg&s=50087)

------------------------------------------

## 一.前言
---------------
文章主要以宏观的形式来聊哔哩哔哩侧边导航拖拽组件，非常适合正在渐进式学习VUE的你，适当的模仿开发项目是前端学习必须要有的技能。大多数人都知道的是，面试需要有自己的作品，而作品最重要的不是切页面，而是：<font face="黑体" color=red>创新+用户体验+性能优化+技术展示</font> 。作者也是一个前端小白，正在摸索阶段，我今天讲解的是模仿我觉得做的不错的**侧边导航栏**，希望大家有收获。让我们一起来，淡黄的长裙，蓬松的头发，拽拽拽！

#### 组件展示
> 这是一个模仿老版哔哩哔哩的侧边导航栏组件，部分效果如下图：

![](https://user-gold-cdn.xitu.io/2020/4/3/1713e09e6e811639?w=1673&h=672&f=gif&s=3663992)

根据效果图可以看出，组件拥有以下功能：

1. 导航栏中的条目元素`item`可以进行拖拽，并且页面专题结构同步改变。
2. 点击任意条目元素`item`，可以立即到其对应的页面位置。
3. 当浏览页面时，移动的某个专题时，旁边的条目元素`item`也会与之对应。


---------------


## 二.具体讲解
- 根据需求：本文将简述对h5和css进行编写，重点是如何实现实时滚动导航和拖拽。

### 获取专题名称及其相关数据

 #### 1.首先我们要去vuex里面拿数据，完成显示专题名称，拖拽等功能，需要`sortValues`、`sortKeys`以及`sortIds`,vuex通过去请求哔哩哔哩官方提供的api进行拿取。具体过程暂且忽略，部分代码如下（因为这个是一个全栈项目，而这个组件和其他组件的关联程度最大，所以作者有点不好如何讲解，还望多多谅解，文末将会附上guthub地址）：
 ```
import { contentApi, contentrankApi } from '@/api'
import * as TYPE from '../actionType/contentType' //采用actionType便于开发与管理

const state = {
	// 默认排序
	sortKeys: ['douga', 'bangumi', 'music', 'dance', 'game', 'technology', 'life', 'kichiku', 'fashion', 'ad', 'ent', 'movie', 'teleplay'],
	sortIds: [1, 13, 3, 129, 4, 36, 160, 119, 155, 165, 5, 23, 11],
	sortValues: ['动画', '番剧', '音乐', '舞蹈', '游戏', '科技', '生活', '鬼畜', '时尚', '广告', '娱乐', '电影', 'TV剧'],
	rows: [],
	ranks: [],
	rank: {}
}

const getters = {
	rows: state => state.rows,
	sortKeys: state => state.sortKeys,
	sortIds: state => state.sortIds,
	ranks: state => state.ranks,
	rank: state => state.rank,
	sortValues: state => state.sortValues
}

const actions = {
	getContentRows({commit, state, rootState}) {
		rootState.requesting = true
		commit(TYPE.CONTENT_REQUEST)
		contentApi.content().then((response) => {
			rootState.requesting = false
			commit(TYPE.CONTENT_SUCCESS, response)
		}, (error) => {
			rootState.requesting = false
			commit(TYPE.CONTENT_FAILURE)
		})
	},
	getContentRank({commit, state, rootState}, categoryId) {
		console.log(categoryId)
		rootState.requesting = true
		commit(TYPE.CONTENT_RANK_REQUEST)
		let param = {
			categoryId: categoryId
		}
		contentrankApi.contentrank(param).then((response) => {
			rootState.requesting = false
			if (categoryId === 1) {
				console.log(response)
			}
			commit(TYPE.CONTENT_RANK_SUCCESS, response)
		}, (error) => {
			rootState.requesting = false
			commit(TYPE.CONTENT_RANK_FAILURE)
		})
	}
}
const mutations = {
	[TYPE.CONTENT_REQUEST] (state) {

	},
	[TYPE.CONTENT_SUCCESS] (state, response) {
		for (let i = 0; i < state.sortKeys.length; i++) {
			let category = state.sortKeys[i] 
			let rowItem = {
				category: category,
				categoryId: state.sortIds[i],
				name: state.sortValues[i],
				b_id: `b_${category}`,
				item: Object.values(response[category])
			}
			state.rows.push(rowItem)
		}
		},
	[TYPE.CONTENT_FAILURE] (state) {

	},

	// 排行榜信息
	[TYPE.CONTENT_RANK_REQUEST] (state) {

	},
	[TYPE.CONTENT_RANK_SUCCESS] (state, response) {
		state.ranks.push(response)
		state.rank = response
	},
	[TYPE.CONTENT_RANK_FAILURE] (state) {
	
	}
}

export default {
	state,
	getters,
	actions,
	mutations
}

 ```
 
 #### 2. 接下来，我们要做的事情就是就是对数据进行初始化。作者先上代码再来解释，代码如下：
 
```
import { mapGetters } from "vuex";
export default {
  mixins: [scrollMixin],
  data() {
    return {
      current: 0, //当前选中条目的序号
      data: [], //数据(name,element,offsetTop,height)
      time: 800, //动画时间
      height: 32, //单个元素的高度
      isSort: false, //排序模式
      scrollTop: 0, //距离页面的顶部距离
      dragId: 0, //拖拽元素序号
      isDrag: false, //当前是否在拖拽
      offsetX: 0, //鼠标在要拖拽的元素上的X坐标上的偏移
      offsetY: 0, //鼠标在要拖拽的元素上的Y坐标上的偏移
      x: 0, //被拖拽的元素在其相对的元素上的X坐标上的偏移
      y: 0 //被拖拽的元素在其相对的元素上的Y坐标上的偏移
    };
  },

```
首先我们将所有我们实现需求所需的数据，全部简答初始化写在`data`,如我们需要实现页面滚动时条目跟随专题，就需要获取这个条目的序号，名字，元素以及距离页面顶部的高度等等。要实现可以把条目进行拖拽，就需要获取是否参与拖拽状态，正在拖拽哪一个条目，所有需要获取拖拽的条目序号以及鼠标的一些数据。
> 仅仅向上面这样初始化数据是远远不够的，要实现需求就必须在兼容所有浏览器的情况下，获取整个网页的大小宽高数据以及对鼠标的操作有着实时的监听。作者先上代码：

```
methods: {
    /** 初始化 */
    init() {
      this.initData(); //初始化
      this.bindEvent();
      this._screenHeight = window.screen.availHeight; //返回当前屏幕高度(空白空间) 
      this._left = this.$refs.list.getBoundingClientRect().left;//方法返回元素的大小及其相对于视口的位置。
      this._top = this.$refs.list.getBoundingClientRect().top;
    },
    /** 绑定事件 */
    bindEvent() {
      document.addEventListener("scroll", this.scroll, false);
      document.addEventListener("mousemove", this.dragMove, false);//当指针设备( 通常指鼠标 )在元素上移动时, mousemove 事件被触发。
      document.addEventListener("mouseup", this.dragEnd, false);//事件在指针设备按钮抬起时触发。
      document.addEventListener("mouseleave", this.dragEnd, false);//指点设备（通常是鼠标）的指针移出某个元素时，会触发mouseleave事件。
      //mouseleave  和 mouseout 是相似的，但是两者的不同在于mouseleave 不会冒泡而mouseout 会冒泡。
      //这意味着当指针离开元素及其所有后代时，会触发mouseleave，而当指针离开元素或离开元素的后代（即使指针仍在元素内）时，会触发mouseout。
    },
    /** 初始化data */
    initData() {
      //将this.options.items转化成新的数组this.data
      this.data = Array.from(this.options.items, item => {
        let element = document.getElementById(item.b_id);
        if (!element) {
          console.error(`can not find element of name is ${item.b_id}`);
          return;
        }
        let offsetTop = this.getOffsetTop(element);
        return {
          name: item.name,
          element: element,
          offsetTop: offsetTop,//返回当前元素相对于其 offsetParent 元素的顶部的距离。
          height: element.offsetHeight//它返回该元素的像素高度，高度包含该元素的垂直内边距和边框，且是一个整数。
        };
      });
    },
    //获取元素距离顶部的距离
    getOffsetTop(element) {
      let top,
        clientTop,
        clientLeft,
        scrollTop,
        scrollLeft,
        doc = document.documentElement,//返回元素
        body = document.body;
      if (typeof element.getBoundingClientRect !== "undefined") {
        top = element.getBoundingClientRect().top;
      } else {
        top = 0;
      }
      clientTop = doc.clientTop || body.clientTop || 0;//表示一个元素的上边框的宽度.boder
      scrollTop = window.pageYOffset || doc.scrollTop;//返回当前页面相对于窗口显示区左上角的 Y 位置。浏览器兼容
      return top + scrollTop - clientTop;
    },
   }
```
- init()：在浏览器中打开可能是全屏或者是小窗，此时页面的大小高度都会改变，我们必须每次当浏览器窗口大小变化时，重新获取（初始化），当前屏幕的高度以及每个条目元素相对窗口的位置，只有这样才可以在不同的情况下，也不出错，实时变化。使用`screen.availHeight.availHeight`获取屏幕高度，使用`getBoundingClientRect()`方法来获取条目元素相对于视窗的位置，如下图所示。

![](https://user-gold-cdn.xitu.io/2020/4/3/1714094710cdb54d?w=240&h=238&f=jpeg&s=4821)

- bindEvent()：这个方法里面写了对鼠标操作以及滚动的行为进行事件绑定，也可说监听，这是实现实时变化的关键。这个方法里面我要特别说一下的是我们使用`mouseleave`，而不使用`mouseout`,的原因是我们需要实现进行拖拽时，当条目元素脱出侧边栏，这个元素将不会显示了(下面将放上展示动图)，因为触发了`mouseleave`，这个方法是当鼠标离开其父组件时触发。不使用`mouseout`是因为这个方法离开元素自己的位置就会触发离开其父级元素的时候也会触发，是冒泡触发的。这里我们使用一定要准确，如果你还是有点不理解，可以去试试MDN上的对比演示demo[演示demo文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/mouseleave_event)。

![](https://user-gold-cdn.xitu.io/2020/4/3/1714098deedd3dff?w=205&h=711&f=gif&s=269436)

- initData(): 将`this.options.items`转化成新的数组`this.data`,返回名字、元素本身、元素相对于其 offsetParent 元素的顶部的距离以及该元素的像素高度，高度包含该元素的垂直内边距和边框。

-  getOffsetTop（）：获取条目元素距离顶部的距离，这里作者不过多讲解推荐一篇文章[JavaScript之scrollTop、scrollHeight、offsetTop、offsetHeight等属性学习笔记](https://www.cnblogs.com/wenruo/p/9754576.html)。需要讲解的是`return top + scrollTop - clientTop;`**元素本身的高度加上滚动增加的高度减去一个重复的上边框高度才是实际的元素的高度**。
#### 3. 现在我们就要开始实现第一个功能，点击条目元素，网页移动到对应的位置，我们要实现这个功能很容易，只要获取对应条目元素的位置和`index`就可以实现，但是要实现平滑的滚动需要引入`smooth-scroll.js`代码如下：

```
        <div
          class="n-i sotrable"
          :class="[{'on': current===index && !isSort}, {'drag': isDrag && current === index}]"
          @click="setEnable(index)"
          @mousedown="dragStart($event, index)"
          :style="dragStyles"
          :key="index"
        >
          <div class="name">{{item.name}}</div>
        </div>
        
         <div class="btn_gotop" @click="scrollToTop(time)"></div>
         
         
    setEnable(index) {
      if (index === this.current) {
        return false;
      }
      this.current = index;
      let target = this.data[index].element;
      this.scrollToElem(target, this.time, this.offset || 0).then(() => {});
    },
```
> smooth-scroll.js

```
window.requestAnimationFrame = window.requestAnimationFrame || window.mozRequestAnimationFrame || window.webkitRequestAnimationFrame || window.msRequestAnimationFrame

const Quad_easeIn = (t, b, c, d) => c * ((t = t / d - 1) * t * t + 1) + b

const scrollTo = (end, time = 800) => {
	let scrollTop = window.pageYOffset || document.documentElement.scrollTop
	let b = scrollTop
	let c = end - b
	let d = time
	let start = null

	return new Promise((resolve, reject) => {
		function step(timeStamp) {
			if (start === null) start = timeStamp
			let progress = timeStamp - start
			if (progress < time) {
				let st = Quad_easeIn(progress, b, c, d)
				document.body.scrollTop = st
				document.documentElement.scrollTop = st
				window.requestAnimationFrame(step)
			}
			else {
				document.body.scrollTop = end
				document.documentElement.scrollTop = end
				resolve(end)
			}
		}
		window.requestAnimationFrame(step)
	})
}

const scrollToTop = (time) => {
	time = typeof time === 'number' ? time : 800
	return scrollTo(0, time)
}

const scrollToElem = (elem, time, offset) => {
	let top = elem.getBoundingClientRect().top  + ( window.pageYOffset || document.documentElement.scrollTop )  - ( document.documentElement.clientTop || 0 )
	return scrollTo(top - (offset || 0), time)
}

export default {
	methods: {
		scrollToTop,
		scrollToElem,
		scrollTo
	}
}

```
关于`smooth-scroll.js`,作者推荐自己查一下资料，有比较多。

#### 4. 实现页面滚动时条目元素跟随对应，代码如下：

```
     //  偏移值
    offset() {
      return this.options.offset || 100;
    },
     /** 滚动事件 */
    scroll(e) {
      this.scrollTop =
        window.pageYOffset ||
        document.documentElement.scrollTop + document.body.scrollTop;//浏览器兼容，返回当前页面相对于窗口显示区左上角的 Y 位置
      if (this.scrollTop >= 300) {
        this.$refs.navSide.style.top = "0px";
        this.init();
      } else {
        this.$refs.navSide.style.top = "240px";
        this.init();
      }
      // console.log("距离顶部" + this.scrollTop);
      //实时跟踪页面滚动
      for (let i = 0; i < this.data.length; i++) {
        if (this.scrollTop >= this.data[i].offsetTop - this.offset) {
          this.current = i;
        }
      }
    },

``` 
这里我们可以看到，我们使用了初始化里面的数据，然后滚动的关键就是获得元素到窗口的距离以及偏移值。需要注意的一个细节是**滚动时元素与窗口顶部的距离大于300px**时，整个组件将吸顶。


#### 5. 实现拖拽
> 1. 进入排序模式

```
  <div class="nav-side" :class="{customizing: isSort}" ref="navSide">  <!--默认不进行排序-->
    <transition name="fade">
      <div v-if="isSort">
        <div class="tip"></div>
        <div class="custom-bg"></div>
      </div>
    </transition>
 </div>
 //进入排序模式
    sort() {
      this.isSort = !this.isSort;
      this.$emit("change");
    },
    
    .fade-enter-actice, .fade-leave-active {
    transition: opacity 0.3s;
  }
  
  .fade-enter, .fade-leave-active {
    .tip {
      top: 50px;
      opacity: 0;
    }

    .custom-bg {
      top: 150px;
      left: -70px;
      height: 100px;
      width: 100px;
      opacity: 0;
    }
  }
}
```
通过上面的代码可知，进入排序模式的代码比较简单，主要是由css的动画来实现。


> 2.开始拖拽


```
/** 得到鼠标位置 */
    getPos(e) {
      this.x = e.clientX - this._left - this.offsetX;
      this.y = e.clientY - this._top - this.offsetY;
    },
/** 拖拽开始 */
    dragStart(e, i) {
      if (!this.isSort) return false;
      this.current = i;
      this.isDrag = true;
      this.dragId = i;
      this.offsetX = e.offsetX;
      this.offsetY = e.offsetY;
      this.getPos(e);
    },
```

开始拖拽时，需要判断是否进入了排序，进入了才允许可以进行拖拽，此时获得鼠标选中的位置，元素的位置以及对应id。

> 3.拖拽中


```
<template v-for="(item, index) in data" >
        <div
          v-if="isDrag && index === replaceItem && replaceItem <= dragId"
          class="n-i sotrable"
          :key="item.name"
        >
          <div class="name"></div>
        </div>
        <div
          class="n-i sotrable"
          :class="[{'on': current===index && !isSort}, {'drag': isDrag && current === index}]"
          @click="setEnable(index)"
          @mousedown="dragStart($event, index)"
          :style="dragStyles"
          :key="index"
        >
          <div class="name">{{item.name}}</div>
        </div>
        <div
          v-if="isDrag && index === replaceItem && replaceItem > dragId"
          class="n-i sotrable"
          :key="item.name"
        >
          <div class="name"></div>
        </div>
</template>
      
      
    // 拖拽的元素的position会变为absolute,dragStyles用来设置其位置,鼠标运动时会调用,从而实现跟随鼠标运动
    dragStyles() {
      return {
        left: `${this.x}px`,
        top: `${this.y}px`
      };
    },
    //当被拖拽的元素运动到其他元素的位置时,会使得replaceItem发送变化
    replaceItem() {
      let id = Math.floor(this.y / this.height);
      if (id > this.data.length - 1) id = this.data.length;
      if (id < 0) id = 0;
      return id;
    }
    
     /** 拖拽中 */
    dragMove(e) {
      if (this.isDrag) {
        this.getPos(e);
      }
      e.preventDefault();//该方法将通知 Web 浏览器不要执行与事件关联的默认动作（如果存在这样的动作）
    },
```

进入拖拽时，首要的是判断是否获取了要拖拽元素的鼠标位置，如果没有获取到，将无法进行拖拽，则使用`e.preventDefault()`通知浏览器不进行拖拽。然后使用`dragStyles()`获取元素拖拽的实时位置。最后元素拖拽时会改变其他元素的位置，位置改变了，其对应的id就会发生变化，我们通过`replaceItem()`来实现，在这个方法里面，我们奇妙的利用**元素的实时高度与元素本身的高度相除获得动态的id**。

> 4. 拖拽完成


```
    /** 拖拽结束 */
    dragEnd(e) {
      if (this.isDrag) {
        this.isDrag = false;
        if (this.replaceItem !== this.dragId) {
          this.options.items.splice(
            this.replaceItem,
            0,
            this.options.items.splice(this.dragId, 1)[0]
          );
        } else {
          this.setEnable(this.dragId, true);
        }
```
这段代码巧妙的是，首先判断是否还在进行拖拽如果有，则`this.isDrag = false;`停止拖拽，接着就是核心部分巧妙利用`splice`,如果`this.replaceItem !== this.dragId`,则在`this.replaceItem`后面添加` this.options.items.splice(this.dragId, 1)[0]`,即这个拖拽元素初始id,相当于拖拽不成功，回到原来的位置，否则拖拽成功。下面我用动图来演示一下。

![](https://user-gold-cdn.xitu.io/2020/4/4/171434182e434eae?w=288&h=625&f=gif&s=199437)

--------------------
### 最后今天是清明节，也是我们深切悼念新冠肺炎疫情牺牲的烈士和逝世同胞的日子，把网站变灰。

在全局中加上如下css就好，代码如下，参考文章[tuitui](https://juejin.im/post/5e86e221e51d4546ce27b99c)
```
  #app 
    filter grayscale(100%)
    -webkit-filter grayscale(100%)
    -moz-filter grayscale(100%)
    -ms-filter grayscale(100%)
    -o-filter grayscale(100%)
    filter url("data:image/svg+xml;utf8,<svg xmlns=\'http://www.w3.org/2000/svg\'><filter id=\'grayscale\'><feColorMatrix type=\'matrix\' values=\'0.3333 0.3333 0.3333 0 0 0.3333 0.3333 0.3333 0 0 0.3333 0.3333 0.3333 0 0 0 0 0 1 0\'/></filter></svg>#grayscale")
    filter progid:DXImageTransform.Microsoft.BasicImage(grayscale=1)
    -webkit-filter: grayscale(1)
```
效果图：

![](https://user-gold-cdn.xitu.io/2020/4/4/171434899cf86d2a?w=1917&h=916&f=png&s=1000424)


-----------------------
### 结束
 > 文章看到现在也结束啦，如果有错误的话就麻烦大家给我指出来吧！如果觉得不错的话别忘了点个赞👍再走噢！

### 最后附上Github地址
- 源码地址：[bilibili](https://github.com/zby-66/bilibili)

### 个人博客地址

- 博客地址：[小小洋的博客](http://182.92.70.96/)

### 期待

- 作者大三正在寻找春招实习中，期待大佬的青睐~