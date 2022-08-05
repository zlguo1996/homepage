---
title: rn-traps
tags:
---

- rn文本中，在安卓机上表情截断问题：https://superxlcr.github.io/2018/06/19/%E5%AD%97%E7%AC%A6%E6%88%AA%E6%96%AD%E5%BC%95%E5%8F%91%E7%9A%84emoji%E8%A1%A8%E6%83%85%E4%B9%B1%E7%A0%81%E9%97%AE%E9%A2%98/

- ScrollView scrollTo函数在useEffect中不生效

  - 不生效：

    ```js
    useEffect(() => {
      	scrollRef.current.scrollTo({ x });
    }, [list, punchCardDayCount]);
    ```

  - 生效：

    ```js
    useEffect(() => {
        setTimeout(() => {
          scrollRef.current.scrollTo({ x });
        }, 0);
    }, []);
    ```
  
- 为正常在安卓上使用`measureInWindow`和`measure`时需要设置View的collapsable属性为false，否则会返回undefined。详见：https://fantashit.com/measuring-a-view-without-an-onlayout-returns-an-empty-set-of-coordinates/
    
- rn中使用transform可能会导致像素级的视觉问题（如：圆不圆，因为单个像素渲染的误差），不论png还是svg均无法解决问题。例：注意圆的左右边缘
 ![](https://p5.music.126.net/obj/wonDlsKUwrLClGjCm8Kx/13580427389/5a65/d7f6/6077/13084c8984bdc6bceec37f4e3af2d61d.png)

 - rn输入，键盘上部的完成按钮： https://stackoverflow.com/questions/49249862/react-native-done-button-above-keyboard
   - `import { KeyboardAccessoryNavigation } from 'react-native-keyboard-accessory'` 放置在顶层View的最底部

    
- rn Text 多语言混排可能会出现文字被竖直截断的情况，如：᭓抖音热歌。对外文文字进行过滤：`str.replace(/([\u1000-\u109F]+)|([\u1B00-\u1B7F])/g, '')`

- rn transform 作用顺序。array中的操作是从末尾开始作用的，具体可以见：https://stackoverflow.com/a/60632809

