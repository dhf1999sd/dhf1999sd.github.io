---
layout: post
title: SystemVerilog
date:   2024-11-28
header-img: img/tiao.jpg
author:     DHF1999SD
tags: 
    - Technology
    - FPGA
catalog: true
comments: true
author: DHF1999SD
---





<!-- more -->
# SystemVerilog
## 基本概念
FPGA硬件描述语言
### 介绍

### 数据类型
#### 内建数据类型
##### 基本数据类型
    time        64位整数，默认单位为秒
    real        来自Verilog，就如C的double类型，64位
    shortreal   来自C的float类型，32位
    string      可变长度的字符数组
    void        空返回，用于函数
##### 2.按照有符号和无符号分类
    有符号类型：byte、shortint、int、longint、integer
    无符号类型：bit、logic、reg、wire等线网类型
    ※记忆：按位定义的变量是无符号的。
#### 结构体    
    语法：

    struct{
    list of different types of variables with sizes
    } structure name;

    示例：

    struct{
    string name;
    bit[15:0] salary;
    byte id;
    } employee_s;
## 常用语法






