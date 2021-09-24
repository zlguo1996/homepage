---
title: 埋点HOC
date: 2021-02-26 15:09:46
categories:
- code
- frontend
- log
---

在react中，对于公共组件，与组件职责无关的埋点参数需要通过props深入传递，影响代码可读性和耦合度。本文介绍了一种通过HOC在祖先组件中添加埋点信息，并在埋点组件中读取的方式。通过独立数据流，提高了代码的可读性，降低耦合度。

### 问题示例

#### 需求

两个业务组件：

1. 音乐列表组件 - MusicList
2. 音乐列表项组件 - MusicItem

有两个业务页面：

1. 播放列表页面 - PlayListPage

2. 收藏列表页面 - CollectionListPage

组织结构如下：

```jsx
////////// 组件 //////////

// 音乐列表
function MusicList({fetchList}) {
	const [list, setList] = useState([])
	useEffect(() => {
		fetchList.then(res => {setList{res}})
	}, [])
  
  const musicItems = list.map(item => <MusicItem key={item.id} item={item} />)
  
  return <div>{musicItems}</div>
}

// 音乐列表项
function MusicItem({item}) => {
  return <div>{item.name}</div>
}

////////// 页面 //////////

// 播放列表页面
function PlayListPage() {
  return <MusicList fetchList={fetchPlayList} />
}

// 收藏列表页面
function CollectionListPage() {
  return <MusicList fetchList={fetchCollectionList} />
}
```

现在我们需要对以上页面进行埋点，埋点信息格式如下：

```js
// 页面曝光埋点 - 播放列表页面的曝光和收藏列表页面的曝光
const pageImpressLog = {
  page: 'play_list' or 'collection_list',
  type: 'impress',
}

// 音乐列表项的点击埋点 - 分别对两个页面的每个元素进行埋点
const itemClickLog = {
  page: 'play_list' or 'collection_list',
  type: 'click',
  id: '5486234' // 音乐的id
}
```

#### 问题

对于页面曝光埋点，我们可以直接在对应的页面上进行埋点。但是对于公共业务组件，埋点就显得比较繁琐：我们需要在通过props把page的信息传入到MusicList中，再由它传至MusicItem。这会带来两个问题：

1. page信息实际上和MusicList / MusicItem 组件无关，这影响了组件的独立性。
2. 当公共组件内有多层嵌套的时候，这个无关信息需要深入传递，增加了组件的复杂度。

### 分析

对于以上例子透露的问题，以及对需求进行分析，我们可以发现：

1. 页面曝光和音乐列表项点击的埋点存在公共字段（如：`page`），可以复用，且使用公共字段的两个组件存在逻辑上的嵌套关系。
2. 公共组件可以使用props中的信息，对属于自己的特殊字段进行埋点（如：音乐列表项点击埋点中的`id`字段可以从通过`item.id`从props中提取）。

3. 我们希望使用独立于props之外的独立数据流来提供埋点信息，降低埋点代码对业务代码的污染。

### 解决方案

1. 在祖先组件中注入和该组件相关的埋点信息
2. 在后代组件中合并并使用所有祖先组件中注入的埋点信息

#### demo

```diff
////////// 组件 //////////

// 音乐列表
function MusicList({fetchList}) {
	const [list, setList] = useState([])
	useEffect(() => {
		fetchList.then(res => {setList{res}})
	}, [])
  
  const musicItems = list.map(item => <MusicItem key={item.id} item={item} />)
  
  return <div>{musicItems}</div>
}

// 音乐列表项
function MusicItem({item}) {
+  const log = useLog()
  const onClick = () => {
+   sendLog({
			...log,
			type: 'click',
			id: item.id
    })
  }
  return <div >{item.name}</div>
}

////////// 页面 //////////

// 播放列表页面
+ const PlayListPage = injectLog({
+  page: 'play_list'
+ })(
  () => {
    const log = useLog()
    useEffect(() => {
      sendLog({
        ...log,
        type: 'impress'
      })
    }, [])
  	return <MusicList fetchList={fetchPlayList} />
	}
)

// 收藏列表页面
function CollectionListPage = injectLog({
  page: 'collection_list'
})(
  () => {
    const log = useLog()
    useEffect(() => {
      sendLog({
        ...log,
        type: 'impress'
      })
    }, [])
    return <MusicList fetchList={fetchCollectionList} />
  }
)
```

从高亮行可以看到这种解决方案的示例：

1. 在页面中使用`injectLog`注入公共埋点信息。
2. 在埋点发送处使用`useLog`获取合并后的埋点信息，并发送。

#### 一次失败的尝试

由于react[使用DFS进行渲染](https://imkev.dev/react-rendering-order)，我的第一个想法是使用一个stack来存储所有祖先节点的渲染信息。`injectLog`HOC的简单版实现如下：

```js
const stack = []

function injectLog(log){
  return (Component) => {
    return (props) => {
     	stack.push(log)
    	const component = React.createElement(Component, props)
    	stack.pop(log)
    	return component 
    }
  }
}
```

但是这里执行stack pop操作的位置有问题。因为React.createElement并不会执行render函数。它返回的是一个Element，是一个所渲染的元素的描述，可以理解为一个plain object。React会通过render函数返回的描述自己控制渲染的执行。实际上，react profiler提供了onRender API。在render执行结束后会调用该回调函数。但是由于profiler会影响性能，官方不建议在生产环境使用。

#### 注入和读取

使用react context，独立与组件之外对埋点信息进行存储。通过provider和consumer实现了脱离props传参之外的数据流。
