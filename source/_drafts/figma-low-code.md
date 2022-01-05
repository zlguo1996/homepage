---
title: 【学习笔记】低代码（Figma）
tags:
- code
- frontend
- low code
- figma
---


## 知识点

1. 设计稿 到 代码
   - HTML生成
     - 图片：图片上传、地址
   - CSS生成
     - 动态样式（e.g. 主题色）：变量声明，变量计算（e.g. calc）
     - 样式去重
   - JS生成
     - 动态参数（props）：参数声明
2. 代码 到 设计稿
   1. HTML处理
   2. CSS处理
      - 不支持的样式：使用lint
   3. JS处理
      - 逻辑呈现？

## 框架

figma -》 vue
luisa框架，把figma设计图解析成一个vue组件，然后在业务代码中引用这个设计图，传入数据和方法  
https://luisa.cloud/help/luisa_framework.html  

figma -》 react & vue  
blog：https://blog.prototypr.io/introducing-figma-to-react-86b0c064e984
官网：https://overlay-tech.com/?ref=article-figmaconcept

  

分享  
https://www.figma.com/file/ZW7L7l5zrUbyVyzmbHPEOR/Figma-To-Code-(Copy)
