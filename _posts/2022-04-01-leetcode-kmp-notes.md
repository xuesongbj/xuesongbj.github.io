---
layout: post
title: 'KMP字符串查找算法'
tags: [code]
---

# KMP(Knuth-Morris-Pratt)字符串查找算法

KMP算法可在一个字符串`s`中查找一个词`W`的出现位置。一个词在不匹配时本身就包含足够的信息来确定下一个匹配可能的开始位置，此算法利用这一特征以避免重新检查先前配对的字符。

这种算法不太容易理解，我也是翻阅了很多资料，才理解。本篇通过一个例子解释什么是KMP。

&nbsp;

## KMP原理

举例来说，有一个字符串"BBC ABCDAB ABCDABCDABDE"，我想知道，里面是否包含另一个字符串"ABCDABD"？

* 首先，字符串"BBC ABCDAB ABCDABCDABDE"的第一个字符与搜索词"ABCDABD"的第一个字符，进行比较。因为B与A不匹配，所以搜索词后移一位。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_01.png)

* 因为B与A不匹配，搜索词再往后移。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_02.png)

* 就这样，直到字符串有一个字符，与搜索词的第一个字符相同为止。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_03.png)

* 接着比较字符串和搜索词的下一个字符，还是相同。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_04.png)

* 直到字符串有一个字符，与搜索词对应的字符不相同为止。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_05.png)

* 这时，最自然的反应是，将搜索词整个后移一位，再从头逐个比较。这样做虽然可行，但是效率很差，因为你要把"搜索位置"移到已经比较过的位置，重比一遍。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_06.png)

* 一个基本事实是，当空格与D不匹配时，你其实知道前面六个字符是"ABCDAB"。KMP算法的想法是，设法利用这个已知信息，不要把"搜索位置"移回已经比较过的位置，继续把它向后移，这样就提高了效率。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_07.png)

* 怎么做到这一点呢？可以针对搜索词，算出一张《部分匹配表》（Partial Match Table）。这张表是如何产生的，后面再介绍，这里只要会用就可以了。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_08.png)

* 已知空格与D不匹配时，前面六个字符"ABCDAB"是匹配的。 

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_09.png)

查表可知，最后一个匹配字符B对应的"部分匹配值"为2，因此按照下面的公式算出向后移动的位数：

```bash
移动位数 = 已匹配的字符数 - 对应的部分匹配值
```

因为 6 - 2 等于4，所以将搜索词向后移动4位。

* 因为空格与Ｃ不匹配，搜索词还要继续往后移。这时，已匹配的字符数为2（"AB"），对应的"部分匹配值"为0。所以，移动位数 = 2 - 0，结果为 2，于是将搜索词向后移2位。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_10.png)

* 因为空格与A不匹配，继续后移一位。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_11.png)

* 逐位比较，直到发现C与D不匹配。于是，移动位数 = 6 - 2，继续将搜索词向后移动4位。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_12.png)

* 逐位比较，直到搜索词的最后一位，发现完全匹配，于是搜索完成。如果还要继续搜索（即找出全部匹配），移动位数 = 7 - 0，再将搜索词向后移动7位，这里就不再重复了。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_13.png)

* 下面介绍《部分匹配表》是如何产生的。

首先，要了解两个概念："前缀"和"后缀"。 "前缀"指除了最后一个字符以外，一个字符串的全部头部组合；"后缀"指除了第一个字符以外，一个字符串的全部尾部组合。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_14.png)

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_15.png)

* "部分匹配值"就是"前缀"和"后缀"的最长的共有元素的长度。以"ABCDABD"为例，

- "A"的前缀和后缀都为空集，共有元素的长度为0；
- "AB"的前缀为[A]，后缀为[B]，共有元素的长度为0;
- "ABC"的前缀为[A, AB]，后缀为[BC, C]，共有元素的长度0；
- "ABCD"的前缀为[A, AB, ABC]，后缀为[BCD, CD, D]，共有元素的长度为0；
- "ABCDA"的前缀为[A, AB, ABC, ABCD]，后缀为[BCDA, CDA, DA, A]，共有元素为"A"，长度为1；
- "ABCDAB"的前缀为[A, AB, ABC, ABCD, ABCDA]，后缀为[BCDAB, CDAB, DAB, AB, B]，共有元素为"AB"，长度为2；
- "ABCDABD"的前缀为[A, AB, ABC, ABCD, ABCDA, ABCDAB]，后缀为[BCDABD, CDABD, DABD, ABD, BD, D]，共有元素的长度为0。

* "部分匹配"的实质是，有时候，字符串头部和尾部会有重复。比如，"ABCDAB"之中有两个"AB"，那么它的"部分匹配值"就是2（"AB"的长度）。搜索词移动的时候，第一个"AB"向后移动4位（字符串长度-部分匹配值），就可以来到第二个"AB"的位置。

![kmp](https://raw.githubusercontent.com/xuesongbj/xuesongbj.github.io/master/_posts/imgs/kmp/kmp_16.png)

&nbsp;

## 具体代码

```python
#!/usr/bin/env python
#-*- coding:utf-8 -*-

def kmp_match(s, p):
    match_idx = []
    m = len(s)      # 完整字符串
    n = len(p)      # 需要查询字串
    cur = 0         # 搜索起始指针
    table = partial_table(p)    # 字串匹配表
    while cur <= m-n:           # 只去匹配前m-n个
        for i in range(n):      # 按字串长度进行匹配，如果字串没有匹配上，向后位移
            if s[i+cur] != p[i]:
                cur += max(i - table[i-1], 1)  # 移动位数 = 已匹配的字符数 - 对应的部分匹配值
                break
        else:
            # 匹配上！！！
            match_idx.append(cur)
            cur+=1
    return match_idx


# 部分匹配表
def partial_table(p):
    prefix = set()
    postfix = set()
    ret = [0]   # 第一位无须匹配，从第二位开始
    for i in range(1, len(p)):
        prefix.add(p[:i])                                   # 根据匹配子串长度，依次更新前缀集合
        postfix = {p[j:i+1] for j in range(1, i+1)}         # 每次循环子串，更新后缀集合
        ret.append(len((prefix & postfix or {''}).pop()))   # 每循环一次更新字串同时出现在前缀和后缀的字串，返回其长度   
    return ret


def main():
    s = "ABCDEABC"
    p = "ABC"
    kmp_match(s, p)


if __name__ == "__main__":
    main()
```
