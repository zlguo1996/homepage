---
title: Eslint报错排查复盘
date: 2022-08-03 14:18:53
categories:
- code
- frontend
- framework
- eslint
---
## 背景

在实现赤兔脚手架（Modern.js的fork）URL Imports的接口定义Loader时，遇到了ESLint的报错：

```bash
0:0  error  Parsing error: "parserOptions.project" has been set for @typescript-eslint/parser.
The file does not match your project config: chitu-lock/https_music-ox.hz.netease.com_/xxx/index.ts.
The file must be included in at least one of the projects provided
```

本文希望通过记录排查时的思路，来梳理遇到此类：1. 网上无法直接检索到解决方案；2. 且官方文档对此类错误没有详细描述问题的解决思路。

## 问题表现

我们使用了三种eslint的执行方式：

1. 直接使用 eslint CLI 对指定文件进行 fix。执行成功。
2. 定义 format 函数，内部使用 eslint 的 node包，对指定文件进行fix。执行成功。
3. 在脚手架的 开始dev模式 & 文件变化 回调用调用 format 函数，在文件内容下载完成后直接对文本进行fix。执行失败。

## 排查过程

1. 初次遇到这个问题时，表现是 eslint 没有对文件生效。因此我首先尝试将eslint的报错信息打印出来，就得到了背景中的报错信息：
    
    ```jsx
    const formatter = await eslint.loadFormatter('stylish');
    const resultText = formatter.format(results);
    console.log(resultText);
    ```
    
2. 尝试Google这个报错问题，可以检索到这个[stackoverflow的高分回答](https://stackoverflow.com/questions/58510287/parseroptions-project-has-been-set-for-typescript-eslint-parser) ：即有可能我们提供了错误的 parserOptions.project 参数，使 eslint 使用了错误的 tsconfig.json 文件。为了判断 eslint 使用的配置，尝试将其打印出来：
    
    ```jsx
    const config = await eslint.calculateConfigForFile(filePath);
    console.log(config)
    ```
    
    通过打印的信息，我们可以看到 parserOptions.project 均为 ./tsconfig.json。顺便我们也对比了其他生成的eslint 配置，没有任何区别。
    
3. 尝试从源码分析
    1. 通过在 typescript-eslint 中搜索报错信息可以发现，错误源于文件 packages/typescript-estree/src/create-program/createProjectProgram.ts 的 createProjectProgram 方法 。Typescript的工程监听器（Program）无法获取待 lint 文件的信息（currentProgram.getSourceFile 返回 空）
    2. 通过检查 getProgramsForProjects 源码可以发现我们在创建 Program 时的输入参数是一致，但是输出的 Program 却在第三种执行方式时产生了不一致的运行结果。那么只可能是两次运行时某种环境的不同导致了结果的不同。
    3. 再次深入到 Typescript 对 getSourceFile 函数的实现，可以发现 Program interface 中提供了 getSourceFiles 方法来获取所有源文件。将 2、3 两种执行方式进行对比，可以发现执行方式3刚好少了 chitu-lock 文件夹中的 ts 文件

## 原因分析

最后可以发现原因在于，使用 eslint 对文本进行 lint 操作时，filePath实际上还没有真实的文件存在。导致 Typescript 的 WatchProgram 没有对该文件进行监听，使 eslint 认为我们指定需要 lint 的文件不包含在配置的 ts工程 中，导致报错。

```jsx
const results = await eslint.lintText(text, {
    filePath
});
```

## 复盘

在排查过程中，在最后发现两次执行 getSourceFiles 返回结果的不一致后，没有继续深入源码实现进行排查。因为想到了两次执行在文件是否存在上存在不一致。在这次长时间的排查中有以下教训：

1. step by step：功能实现需要一步步实现，对每一步的结果进行测试，这样在某个步骤出错时由于和上一步相差较小，可以较容易地缩小引起错误的范围。
2. 控制变量：排查过程中时刻关注成功和失败的执行环境之间有哪些不同，尽管深入源码最终还是可以解决问题，但是排查效率低；从变量出发的排查往往更加高效。