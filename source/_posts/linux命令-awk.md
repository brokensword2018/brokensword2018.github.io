---
title: linux命令 awk
date: 2020-07-19 15:52:28
tags:
    - linux
  
categories: linux命令
---

### 简介
`awk`是一个用于处理文本的强大工具,取名为三位创始人`Alfred Aho，Peter  Weinberger`, 和 `Brian Kernighan` 的 `Family Name `的首字符。
<!-- more -->
### 基本语法
`akw [options] 'awk programa' inputfile`  
或  
` akw [options] -f 'awk programa file' inputfile`   

`awk programa`是指`awk`的命令,构成为  
`pattern {action}`,`pattern`是正则表达式使用`/`包裹起来,如`/regular-expression/`,然后以行为单位对满足条件的行执行`action`操作.

### 常用`option`   

`-F`:指定文件的分隔符,默认为空格或者`tab` ;`awk`将文本的一行以分隔符分割成多个域`$0`指整行,`$1,$2...`分别指第二列,第三列等.  

### 例子
假设文件`mail-list`内容为 ： 
```
Amelia       555-5553     amelia.zodiacusque@gmail.com    F
Broderick    555-0542     broderick.aliquotiens@yahoo.com R
Julie        555-6699     julie.perscrutabor@skeeve.com   F
Samuel       555-3430     samuel.lanceolis@shu.edu        A
```  

`awk '/Julie/ {print $1,$3}' mail-list`命令就是匹配存在`Julie`的行,然后打印出该行的第一列和第三列,注意要在`$1`和`$1`之间添加逗号,不然第一列和第三列的打印结果会连在一起.
