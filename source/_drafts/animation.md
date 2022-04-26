---
title: animation
tags:
---

# 动画

## Animated
react & react native 使用的底层style动画库（如：react-spring有用到）
[github](https://github.com/animatedjs/animated) [文档](http://animatedjs.github.io/interactive-docs/)

### 特性
Animated.template（用于生成interpolate不支持的差值属性）   
[源码](https://github.com/animatedjs/animated/blob/master/src/AnimatedTemplate.js)
例子：linear-gradient动画   
```
export default function Background({
    tabAnimation,
}) {
    const colorAnimation = useMemo(() => tabAnimation.interpolate({
        inputRange: [0, 1],
        outputRange: [`${BG_COLOR[GENDER.MALE]}`, `${BG_COLOR[GENDER.FEMALE]}`]
    }), [tabAnimation]);
    const gradientAnimation = useMemo(() => Animated.template(['linear-gradient(', ', #00000000)'], colorAnimation), [colorAnimation]);
    const style = useMemo(() => ({
        background: gradientAnimation,
    }), [gradientAnimation]);

    return <Animated.div style={style} styleName="root" />;
}
```   
动画效果：   
<video src="./animation/linear-gradient-animation.mov" />
