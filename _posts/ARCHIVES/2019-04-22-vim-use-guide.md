---
layout: post
title: Vim使用总结
category: ARCHIVES
description: 常见Vim操作案例
---

#### 跳到文件第一行
```
Esc状态
键盘按下 小写 gg
```

#### 跳到文件最后一行
```
Esc状态
键盘按下 大写 G或者shift + g
```

#### 跳到文件指定一行
```
Esc状态
键盘按下 行号 + gg
```

#### 打开文件并指定光标位置
```
vim <filename> +n(行数)
```

#### 跳到Line第一个字符
```
任意状态
键盘按下 fn + >(右移方向键)
```

#### 跳到Line最后一个字符
```
任意状态
键盘按下 fn + <(左移方向键)
```

#### 文件行号展示
```
Esc状态
键盘按下 :set number

:set nonumber 或 :set nonu 取消行号
```

#### 搜索翻动
```
Esc状态
向下搜索  /pattern 按n继续搜索下一个
向上搜索  ?pattern 按#继续搜索上一个
```

