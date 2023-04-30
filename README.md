# self-use
猛学了半年开发后重新看了下afl，彻彻底底的搞懂了一些当时没看懂的东西和整体框架，网络上的源码分析大多数是将c代码翻译成自然语言，只告诉你afl这里做了什么，却没告诉你afl这里为什么这么做，所以我以源码笔记注释的形式重新读了一遍。为将来复习回看或者魔改提速。

# fuzz body

本质就是重复cull -> fuzz_one -> sync的循环
```c
  while (1) {

    cull_queue();// 根据top_rated 将没法触发新路径的testcase标记为redundant(选出最小集可以触发最大路径覆盖率的testcase)
    skipped_fuzz = fuzz_one(use_argv);// 返回1代表跳过了 0代表fuzz一次成功

      if (··· interval ···)
        sync_fuzzers(use_argv);


    queue_cur = queue_cur->next;
    current_entry++;

  }

```
# function hierarchy
- fuzz_one
    - 随机概率性跳过一些testcase(比如不favor的)
    - calibrate_case                // 无变异的跑几次目标程序 观察可变行为和跑完的bitmap效果
        - run_target                // fork-exec去跑一次目标程序 设置好相应的fd并通信(如果已经设置好forksrv了只需要发个ctl信号就行)
        - update_bitmap_score       // 更新top_rated
    - trim_case                     // 剪裁testcase 通过路径反馈观察剪裁是否被应用
    - common_fuzz_stuff             // 在反复的变异中调用common_fuzz_stuff
        - run_target
        - save_if_interesting
            - add_to_queue          // 加到queue末尾
            - calibrate_case        // 个人觉得这里的calibrate是多余的 前面已经run_target一次了
                - run_target        

# data struct 

## bitmap

```c
EXP_ST u8* trace_bits;                /* SHM with instrumentation bitmap  */

EXP_ST u8  virgin_bits[MAP_SIZE],     /* Regions yet untouched by fuzzing */
           virgin_tmout[MAP_SIZE],    /* Bits we haven't seen in tmouts   */
           virgin_crash[MAP_SIZE];    /* Bits we haven't seen in crashes  */

static u8  var_bytes[MAP_SIZE];       /* Bytes that appear to be variable */
```
这里就是afl的命名问题了，virgin_bits所代表的其实是一个比特图而不是位图，但是afl将virgin中都设置为1 代表着每个路径都没被探索过 在更新位图的时候 如果共享内存trace_bits中有探索过的路径 afl就会把virgin这些位置为0来表示已经探索过。但是这存在一个粒度的问题，对于一个byte，trace_bits表示的是hitcount 这个不能单纯的应用对应位置0的方法来调整virgin_bits。不过无伤大雅，真正的位图是queue_entry->trace_mini.这里虽然名字是_bits但其实是byte图.


## queue

```c
static struct queue_entry *queue,     /* Fuzzing queue (linked list)      */
                          *queue_cur, /* Current offset within the queue  */
                          *queue_top, /* Top of the list                  */

static struct queue_entry*
  top_rated[MAP_SIZE];                /* Top entries for bitmap bytes     */
```
刚开始看afl被它的名字误解了 其实queue是链表头 而queue_cur是遍历链表的链表指针 queue_top是链表尾.

top_rated表示这着一个路径会被哪个testcase触发 总是指向最优的那个

- virgin_bits

- common_fuzz_stuff


# sync同步机制
就是跑完了一组queue就去看别人queue上的testcase，拿过来测一下有没有新路径，就是这么个普通的东西，我还以为能根据共享内存去进行格外的调整呢，比如master探索过了这条路径slave就偏向于走另外一条. 所以这也是个优化点



# TODO
- persistent mode
- qemu mode