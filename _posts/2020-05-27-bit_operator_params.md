---
layout:     post
title:      "位运算技巧——灵活控制返回内容"
subtitle:   "\"基本功系列：位运算小技巧\""
date:       2020-05-27 12:00:00
author:     "wangyapu"
header-img: "img/post-bit-operator-params.jpeg"
tags:
    - 基本功
---

# 位运算技巧——灵活控制返回内容

## 前言

位运算的实战技巧有非常多，在 Linux 操作系统以及各大开源框架里都有它的身影。你可能平时基本用不到位运算，但是作为基本功还是需要去积累的，书到用时方恨少，厚积薄发。后续我会把自己想到的、看到的一些位运算小技巧持续分享给大家。

今天恰巧碰到一个场景，接口需要按需返回数据内容，主要有两个原因：

- 数据字段特别多，并不是每个都是必须的。
- 对某类用户一些数据无权限查看。

来看看使用位运算的技巧如何只使用一个参数来控制接口内容的按需返回。

## 实战分析

首先来直观地看一下一个典型案例，Linux mmap 接口的使用方法：

```c
#define	PROT_READ	0x04	/* pages can be read */
#define	PROT_WRITE	0x02	/* pages can be written */
#define	PROT_EXEC	0x01	/* pages can be executed */

mmap(NULL, sizeof(foo)*10, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

接口指定共享内存的访问权限为 PROT_READ（可读）和 PROT_WRITE（可写），为什么简单的位或运算可以实现这样的效果？我们一点点来看。

- 二进制表示

    | 模式 | 二进制 |
    | --- | --- |
    | PROT_EXEC | 0000 0001 |
    | PROT_WRITE | 0000 0010 |
    | PROT_READ | 0000 0100 |

- 位或运算

    ```
    PROT_EXEC | PROT_WRITE = 0000 0001 | 0000 0010 = 0000 0011
    PROT_EXEC | PROT_READ = 0000 0001 | 0000 0100 = 0000 0101
    PROT_EXEC | PROT_WRITE | PROT_READ = 0000 0001 | 0000 0010 | 0000 0100 = 0000 0111
    ```

- 位与运算

    ```
    (PROT_EXEC | PROT_WRITE) & PROT_WRITE = 0000 0011 & 0000 0010 = 0000 0010
    (PROT_EXEC | PROT_WRITE | PROT_READ) & PROT_WRITE = 0000 0111 & 0000 0010 = 0000 0010
    ```

由此可以看出规律，`多个 2 次方数的位或结果与其中一个数（目标数）再进行位与操作，得到的结果不等于 0 且等于目标数，与其他不同的 2 次方数位与结果都是 0。`

## 解决实际问题

借鉴以上的思路，我们可以采用同样的方法灵活地控制接口的返回内容，例如 A 用户需要字段 1、4，用户 B 需要字段1、2、4。

附上简单的示例代码：

```java
public class BitParams {
    public static void main(String[] args) {
        BitParams bitParams = new BitParams();
        bitParams.getData(1 | 4);
        bitParams.getData(1 | 2 | 4);
    }

    private void getData(int flag) {
        System.out.println("return result:");
        for (DataEnum dataEnum : DataEnum.values()) {
            if ((flag & dataEnum.getIndex()) != 0) {
                System.out.println(dataEnum.getDataFiled());
            }
        }
    }

    public enum DataEnum {
        DATA_FILED_1(1, "DATA_FILED_1"),
        DATA_FILED_2(2, "DATA_FILED_2"),
        DATA_FILED_4(4, "DATA_FILED_4");

        private final int index;
        private final String dataFiled;

        DataEnum(int index, String dataFiled) {
            this.index = index;
            this.dataFiled = dataFiled;
        }

        public int getIndex() {
            return index;
        }

        public String getDataFiled() {
            return dataFiled;
        }
    }
}
```


