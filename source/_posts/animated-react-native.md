---
title: Animated in React Native
date: 2020-10-18 21:57:39
categories:
- code
- frontend
- native
---

## Motivation
在学习React Native动画的时候，看到了这段[使用Animated的基础代码](https://zhuanlan.zhihu.com/p/103293495)。可以看到这段代码并不使用`setState`来更新状态以实现动画效果。在本文中，笔者将会总结博客和源码中的相关资料，以期了解这段代码的实现原理。
```JavaScript
const FadeInView = (props) => {
  const fadeAnim = useRef(new Animated.Value(0)).current  // 透明度初始值设为0

  React.useEffect(() => {
    Animated.timing(                  // 随时间变化而执行动画
      fadeAnim,                       // 动画中的变量值
      {
        toValue: 1,                   // 透明度最终变为1，即完全不透明
        duration: 10000,              // 让动画持续一段时间
      }
    ).start()                      // 开始执行动画
  }, [fadeAnim])

  return (
    <Animated.View                 // 使用专门的可动画化的View组件
      style={{
        ...props.style,
        opacity: fadeAnim,         // 将透明度绑定到动画变量值
      }}
    >
      {props.children}
    </Animated.View>
  )
}
```

## Do Work in JS || Native Driver Animations
在[这篇文章](https://www.freecodecamp.org/news/how-react-native-animations-work/)里提到了Animated的两种计算动画的方式和对应的优劣。
1. 在JS线程中使用`requestAnimationFrame`计算动画参数，并使用Bridge将数据传输给原生代码
2. 在动画开始前通过Bridge直接将动画信息传输给原生代码，由UI线程来计算动画参数

可以看到使用[声明式的代码](https://ui.dev/imperative-vs-declarative-programming/)的方式定义动画便于使用UI线程来提高性能。
同时，不论是1还是2，都是由UI线程来渲染原生视图，因此可以跳过频繁的`setState`和`render`来提高性能。

## Source Code
### Animated.Value
在[react-native库](https://github.com/facebook/react-native)的`Libraries/Animated/nodes/AnimatedValue.js`文件中描述了Animated.Value是如何工作的：
Animated构建了一个依赖的有向无环图。view属性的计算包含了两个阶段：
1. Top Down Phase
   当Animated.Value被更新的时候，它会寻找并标记叶节点：即需要更新的view
2. Bottom Up Phase
   从被标记的叶节点回溯，以此得到它所需的value。（这么做的原因是某些view的属性可能是由多个value组合而成，如：transform）

### Animated.[Component]
在`Libraries\Animated\createAnimatedComponent.js`文件中，可以看到在mount/update的时候，使用`this._propsAnimated.setNativeView`为AnimatedProps绑定了component。同时在`Libraries\Animated\nodes\AnimatedProps.js`文件中，使用`__connectAnimatedView`函数通过native tag标识将动画节点与对应的native view相连。
