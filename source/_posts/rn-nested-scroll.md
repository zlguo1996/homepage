---
title: RN嵌套滑动
date: 2021-09-24 15:16:20
categories:
- code
- frontend
- animation
- interaction
- react native
---

在嵌套滑动或其他复杂的交互场景时，我们需要对用户的手势进行识别。判断应该由哪个元素来响应这个手势事件。如在两个嵌套垂直滚动的ScrollView中，我们需要判断用户的意图是作用于内部还是外部的ScrollView。

## 手势识别能力

对于滑动手势的识别，有下面三种能力层级（具体例子可见`参考 - 3`的视频）：

1. 滑动中识别（*abbrev*. 能力1）
2. 滑动开始时识别（*abbrev*. 能力2）
3. 通过上一次滑动手势识别（*abbrev*. 能力3）

由上至下，识别延迟增加，用户体验降低。

## 仅使用react-native

### 一个简单的例子

#### 需求

以下面的[浮层](https://youtu.be/V8maYc4R2G0?t=826)为例。在地图app中，我们需要实现一个纵向滑动的浮层，内部嵌套了一个纵向滑动的地点列表。

![截屏2021-09-08 上午11.34.39](https://i.loli.net/2021/09/08/8jlE9BbvVgsUXzR.png)

在结构上，我们可以这样理解上面这个例子：

```xml
<App>
  <Map />
  <Drawer> // 浮层
    <Search />
    <Locations /> // 地点列表
  </Drawer>
</App>
```

#### 场景

设想我们在浮层展开的状态下向下拖动地点列表，对应`手势识别能力`分别会是如下的交互形式：

| 序号 | 识别能力               | 交互                                                         | 拖动次数 |
| ---- | ---------------------- | ------------------------------------------------------------ | -------- |
| 1    | 滑动中识别             | 用户下拉地点列表到底后，浮层开始下拉                         | 1        |
| 2    | 滑动开始时识别         | 用户下拉地点列表到底后，无法继续下拉。再次下拉时浮层下拉     | 2        |
| 3    | 通过上一次滑动手势识别 | 用户下拉地点列表到底后，无法继续下拉。再次下拉无响应。再次下拉时浮层下拉 | 3        |

#### 实现

对于上面这个简单的例子，我们可以通过一个ScrollView来实现**滑动中识别**的能力：

```xml
<ScrollView stickyHeaderIndices={[1]}>
  <TransparentPlaceholder /> // 透明占位符
  <Search /> // 搜索栏
  <Location-1/> // 地点1
  <Location-2/> // 地点2
  ...
</ScrollView>
```

通过`stickyHeaderIndices`配置，我们可以实现指定项目的吸顶功能。由于只使用了一个ScrollView，在用户的观感上，地点列表拖动到底之后，浮层就会开始下拉，体验流畅。

### 更加复杂的例子

#### 需求

在下面我的播客浮层中，我们添加了可以横向滚动的tab。这使得在横划过后，内部嵌套的ScrollView存在了不同的垂直滚动状态，因此我们不再能够仅用一个ScrollView来模拟内外两层的垂直滚动。

![截屏2021-09-24 下午2.20.51](https://i.loli.net/2021/09/24/CtSqVTBjAibH2rO.png)

#### 思路

这其实是一个响应者（[gesture handler](https://docs.swmansion.com/react-native-gesture-handler/docs/about-handlers)）的问题：当内部ScrollView滚动到底时，根据用户手势的方向，需要判断由内层列表还是外层抽屉对手势进行响应。

对于外层的抽屉，有两种实现方式：

1. 外层使用PanResponder（优点：更可控的外层浮层动画）

   事件拦截：通过`onMoveShouldSetPanResponderCapture`在需要时拦截用户的拖动事件，阻止内部列表的响应

   事件响应：通过`onPanResponderMove`和`onPanResponderRelease`对拖动事件进行响应，通过`Animated.ValueXY`将其绑定到抽屉的transitionY上
   ![5e96a0842dcade04fbfbc993a1y9cA7101](https://i.loli.net/2021/09/08/SaLKOICMtk7ZeUY.jpg)

2. 外层使用ScrollView（优点：更好的性能）

   事件拦截：通过`scrollEnabled`属性对事件进行拦截

   事件响应：使用`nestedScrollEnabled`实现嵌套的滚动，利用ScrollView原生的能力直接进行响应

#### 实现

在这个需求中，我们使用了PanResponder的方法进行实现（方案1）。核心的手势响应代码可以参照`参考 - 2.usePanResponder`。

不管PanResponder（方案1）还是ScrollView（方案2），都无法在滑动过程中切换响应的View，因此只能实现`能力2`。

#### 踩坑

在安卓端使用方案1进行实现时，由于ScrollView存在一些[奇特的表现](https://stackoverflow.com/questions/55082289/react-native-propagate-pan-responder-event-from-view-to-inner-scroll-view/55251301#55251301)，我们需要猜测用户的意图以实现部分操作的`能力2`，而有些时候，由于猜测不准确，只能实现`能力3`。

> 安卓端ScrollView的奇特表现：当ScrollView开始滑动的时候，`onMoveShouldSetPanResponderCapture`和`onPanResponderMove`无法被持续触发。（在测试中通过设置`nestedScrollEnabled`属性，我们的PanResponder可以拦截到更多事件，但仍然无法达到IOS上的效果）
>
> 用户意图的猜测：当用户下拉内部列表到底后，猜测用户下一步会收起浮层，提前锁定内部的`ScrollView`。

## 使用react-native-gesture-handler

从上一节可以看到，仅使用react-native提供的功能，无法实现能力1的嵌套滑动效果。而react-native-gesture-handler优化了手势响应的机制，使其具备更加复杂的手势识别能力。在[github issue的讨论](https://github.com/software-mansion/react-native-gesture-handler/issues/420)中我们可以看到针对抽屉这个场景，如何选择正确的响应者对手势进行响应。同时库中还提供了一个[demo](https://github.com/software-mansion/react-native-gesture-handler/blob/13053b92ac030e340099cdfee648623408bc8021/examples/Example/src/bottomSheet/index.tsx#L129-L132)，通过代码展示如何实现一个嵌套滚动的抽屉。

### 例子

#### 需求

以上面这个地图例子为例。设想我们需要实现更加复杂的功能：即无论内部列表滚动到何种状态，向下拉动浮层的上沿（区域1）可以直接收起浮层。当我们还是使用单个ScrollView时，下拉会先造成内部列表下拉，然后再造成浮层下拉。

![8jlE9BbvVgsUXzR](https://i.loli.net/2021/09/24/PFnmfrNkh3gMEt5.png)

#### 实现

针对这类需求，npm上已经有包利用[demo](https://github.com/software-mansion/react-native-gesture-handler/blob/13053b92ac030e340099cdfee648623408bc8021/examples/Example/src/bottomSheet/index.tsx#L129-L132)的原理进行了封装：[@gorhom/bottom-sheet](https://www.npmjs.com/package/@gorhom/bottom-sheet)。通过简单的代码，即可实现能力1的交互形式：

```jsx
<BottomSheet
  ref={bottomSheetRef}
  index={1}
  snapPoints={SNAP_POINTS}
  onChange={() => {}}>
  <View>
    <Search /> // 搜索栏
    <BottomSheetScrollView>
      <Location-1/> // 地点1
  		<Location-2/> // 地点2
      ...
    </BottomSheetScrollView>
  </View>
</BottomSheet>
```

>  @gorhom/bottom-sheet内部依赖了react-native-gesture-handler和react-native-reanimated，使用时切忌遗漏安装对应的客户端包
>

注：在安卓中，react-native包导出的Touchable和ScrollView无法在BottomSheet中正常响应事件，具体解决方案可见[官方文档的troubleshooting部分](https://gorhom.github.io/react-native-bottom-sheet/v2/troubleshooting)。

## 总结

本文描述了嵌套垂直滑动的三种手势识别能力。并针对抽屉组件中嵌套ScrollView的不同需求场景，简述了仅使用RN提供的API & 使用react-native-gesture-handler，实现不同能力的方法。可以看到，react-native-gesture-handler提供了更加完整的基础能力，以实现复杂的手势响应能力；但与此同时也增加了代码的复杂度。

## 展望

在我的播客例子中，目前仅使用原生的方法实现了能力2的手势响应机制。通过react-native-gesture-handler，是否能够实现能力1的手势响应机制，以及如何实现，还待研究。

## 参考

### 1. react native gesture handler

Document: https://docs.swmansion.com/react-native-gesture-handler/docs/

Video: https://www.youtube.com/watch?v=V8maYc4R2G0&ab_channel=ReactConferencesbyGitNation

#### RN手势检测的问题

1. gesture recognition logic is distributed between threads that run in parallel
2. lack of an API that would allow for defining interactions between native gesture recognizers
3. touch events recognized by JS responder system cannot be connected with native animated nodes

#### 解决方案

在UI线程（native）识别多个手势，按需激活正确的单个响应器，并使用Animated Native Driver实现流畅动画

### 2. usePanResponder

外层抽屉浮层的pan responder配置

```js
import {
    useCallback, useMemo, useRef
} from 'react';
import {
    PanResponder,
    Animated
} from 'react-native';

import {
    usePropsRef, closerToZero, flag, limitInBetween
} from './utils';

const stablePositions = {
    TOP: 0,
    BOTTOM: 1
};

export default function usePanResponder({
    // up为y减小，down为y增大
    lockDown, lockUp,
    captureUp, captureDown,
    thresholdUp, thresholdDown, // 滑动绝对距离的阈值
    // top为y最小值，down为y最大值
    onTopReached, onBottomReached,
    topBoundY, bottomBoundY,
    defaultY, // 默认开始位置
}) {
    const props = usePropsRef({
        lockDown,
        lockUp,
        captureUp,
        captureDown,
        thresholdUp,
        thresholdDown,
        onTopReached,
        onBottomReached,
        topBoundY,
        bottomBoundY,
        defaultY
    });

    // 获取手势前的位置
    const lastStablePosition = useRef(
        Math.abs(defaultY - topBoundY) < Math.abs(defaultY - bottomBoundY)
            ? stablePositions.TOP : stablePositions.BOTTOM
    );
    const getStablePositionY = useCallback(
        stablePosition => (
            stablePosition === stablePositions.TOP
                ? props.current.topBoundY : props.current.bottomBoundY
        ), []
    );

    // 位置动画
    const position = useRef(new Animated.ValueXY({ y: defaultY, x: 0 })).current;

    // 动画position到指定位置
    const transitionTo = useCallback((stablePosition) => {
        const isBottom = stablePosition === stablePositions.BOTTOM;
        Animated.spring(
            position,
            { toValue: isBottom ? props.current.bottomBoundY : props.current.topBoundY }
        ).start(() => {});

        lastStablePosition.current = stablePosition;
        if (isBottom) props.current.onBottomReached();
        else props.current.onTopReached();
    });

    // 手势监听器
    const panResponder = useMemo(() => PanResponder.create({
        onStartShouldSetPanResponder: () => true,
        onMoveShouldSetPanResponderCapture: (evt, gesture) => {
            const verticalCapture = getVerticalPanValue({
                upValue: props.current.captureUp,
                downValue: props.current.captureDown,
                elseValue: false
            }, gesture);

            return verticalCapture;
        },
        onPanResponderMove: (e, gesture) => {
            const lock = getVerticalPanValue({
                upValue: props.current.lockUp,
                downValue: props.current.lockDown,
                elseValue: false
            }, gesture);
            if (lock) return;

            const targetY = getStablePositionY(lastStablePosition.current) + gesture.dy;
            const limitedTargetY = limitInBetween(
                targetY, [props.current.topBoundY, props.current.bottomBoundY]
            );
            if (limitedTargetY === targetY) { // in bound
                position.setValue({ y: targetY });
            } else { // over bound (use rubberband)
                const overDistance = closerToZero(
                    targetY - props.current.topBoundY, targetY - props.current.bottomBoundY
                );
                position.setValue({
                    y: limitedTargetY + flag(overDistance) * calculateRubberband(
                        Math.abs(overDistance)
                    )
                });
            }
        },
        onPanResponderRelease: (e, gesture) => {
            const lock = getVerticalPanValue({
                upValue: props.current.lockUp,
                downValue: props.current.lockDown,
                elseValue: false
            }, gesture);
            if (lock) return;

            const { dy } = gesture;
            const absoluteDy = Math.abs(dy);
            const flagDy = flag(dy);
            if (lastStablePosition.current === stablePositions.TOP
                && absoluteDy > props.current.thresholdDown
                && flagDy > 0) { // to down
                transitionTo(stablePositions.BOTTOM);
            } else if (lastStablePosition.current === stablePositions.BOTTOM
                && absoluteDy > props.current.thresholdUp
                && flagDy < 0) { // to up
                transitionTo(stablePositions.TOP);
            } else { // reset position
                transitionTo(lastStablePosition.current);
            }
        }
    }), [position]);

    return {
        panResponder,
        position
    };
}

function calculateRubberband(overDistance) {
    return Math.sqrt(overDistance);
}

function getVerticalPanValue({ upValue, downValue, elseValue }, gesture) {
    const { dx, dy } = gesture;

    // 竖直滑动
    const panRatio = Math.abs(dx / dy);
    const verticalDrag = panRatio < 2;

    if (!verticalDrag) return elseValue;

    if (dy < 0) { // up
        return upValue;
    } if (dy > 0) { // down
        return downValue;
    }

    return elseValue;
}
```

## 3. 交互视频

1. 能力1，滑动中识别（理想情况，用户只需交互一次）

   <video src="https://vodkgeyttp9c.vod.126.net/vodkgeyttp8/8vdQyfjG_3902554063_uhd.mp4?ts=1949278835&rid=7859AE92117216E8AD0E345795682128&rl=0&rs=cWazXiGUycMdyAhsKhOXEkfeKHPGccOq&sign=add655382502b1a3a778fbade5022bf0&coverId=cxnrl57ADPi6eE9Tj47Y6w==/109951166509772321&infoId=1357303" style="max-width:300px;" />

2. 能力2，滑动开始时识别（用户需要拖动两次，第一次拖动内部列表，第二次拖动外部抽屉）
   
   <video src="https://vodkgeyttp9c.vod.126.net/vodkgeyttp8/79544nqW_3902564433_uhd.mp4?ts=1949279212&rid=7859AE92117216E8AD0E345795682128&rl=0&rs=aDZoqyiQtMxNyNsLqRJRjZoOJFXSQDkf&sign=a3f34774922993db2418e88d86434ce7&coverId=UX3CLuL7ikvtuQkTdxsZRw==/109951166509784344&infoId=1354343" style="max-width:300px;" />
   

