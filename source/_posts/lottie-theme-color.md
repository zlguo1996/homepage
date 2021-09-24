---
title: React Native 动画应用主题颜色实践
date: 2021-04-25 15:02:09
categories:
- code
- frontend
- lottie
- react native
---

## 最佳实践

使用lottie创建动画，修改lottie json中对应颜色的属性。

找到颜色属性的方法：让视觉提供两个颜色不同的json文件，作比较。

如需实现类似tintColor一样修改所有颜色的效果，可以使用Reference中的代码。

## 踩坑

**常规格式动画图片（gif, webp, apng）**

- ios：tintColor不会用到所有格式的动画图片（gif, webp, apng）上
- 安卓：对于asset中的webp图片，tintColor会生效

**Lottie**

- 方案1：使用colorFilter，不生效（不知道是不是配置有错，网上文档较少）
- 方案2：直接改json，让视觉提供色彩不同的两个json文件，compare后可以知道需要改哪些地方

## Reference

修改lottie中所有颜色的代码

```
import { useMemo } from 'react';

function replaceColor(object, color) {
    if (typeof object !== 'object') return; // 不是object

    for (const key of Object.keys(object)) {
        if (key === 'k' && object[key].length === 4 && typeof object[key][0] === 'number') { // 替换颜色
            // eslint-disable-next-line no-param-reassign
            object[key] = [color[0], color[1], color[2], object[key][3]]; // 保留alpha
        } else {
            replaceColor(object[key], color);
        }
    }
}

// 将Lottie文件中的所有颜色替换为给定颜色
// targetColor格式 rgba(x,x,x,x)
export default function useLottieThemeColor(json, targetColor) {
    return useMemo(() => {
        const res = JSON.parse(JSON.stringify(json));

        let colorVec;
        try {
            colorVec = JSON.parse(targetColor.replace(/(rgba\()([0-9., ]+)(\))/, '[$2]'));
            colorVec = colorVec.map((v, index) => {
                if (index >= 0 && index < 3) {
                    return v / 255;
                }
                return v;
            });
        } catch {
            console.error(`Failed to parse theme color: ${targetColor} to color vector`);
            return res;
        }

        replaceColor(res, colorVec);

        return res;
    }, []);
}
```
