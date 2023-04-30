# self-use
猛学了半年开发后重新看了下afl，彻彻底底的搞懂了一些当时没看懂的东西和整体框架，网络上的源码分析大多数是将c代码翻译成自然语言，只告诉你afl这里做了什么，却没告诉你afl这里为什么这么做，所以我以源码笔记注释的形式重新读了一遍。为将来复习回看或者魔改提速。

# TODO
- run_target

- virgin_bits


- queues

- forkserver


# bitmap
afl采用一个byte代表一个路径 而这个byte的数值则代表hitcount
所以存在bitmap和bytemap
bitmap表示一个路径有没有走到过
而bytemap能表示命中次数

我个人觉得has_new_bits中有点问题 传入的是一个byte图

在virgin_map中 在一个u8里 用一个8位的bit代表着hitcount。
virgin开始会全置1 只要cur中哪些位走过了 就会将其那位清空 所以virgin表示没走到的位图


# sync同步机制


# 如果virgin不变了怎么办