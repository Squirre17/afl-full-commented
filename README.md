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

# qemu mode
主要就是加入了一个afl_forksrv_pid的pid
然后kill这个pid会被patch成kill掉自己qemu本身
```diff
@@ -11688,8 +11697,20 @@ abi_long do_syscall(void *cpu_env, int n
         break;
 
     case TARGET_NR_tgkill:
-        ret = get_errno(safe_tgkill((int)arg1, (int)arg2,
-                        target_to_host_signal(arg3)));
+        {
+          int pid  = (int)arg1,
+              tgid = (int)arg2,
+              sig  = (int)arg3;
+
+          /* Not entirely sure if the below is correct for all architectures. */
+
+          if(afl_forksrv_pid && afl_forksrv_pid == pid && sig == SIGABRT)
+              pid = tgid = getpid();
+
+          ret = get_errno(safe_tgkill(pid, tgid, target_to_host_signal(sig)));
+
+        }
+
```

然后在解析elf的时候会填入这三个指针
```c
abi_ulong afl_entry_point, /* ELF entry point (_start) */
          afl_start_code,  /* .text start pointer      */
          afl_end_code;    /* .text end pointer        */
```

在QEMU中，TranslationBlock（简称TB）是将一段机器码（即汇编指令）翻译成中间语言（TCODE）的基本单位。一个TB包含了执行一段连续的机器码所需的所有信息，例如起始地址、字节码、翻译后的代码、执行时需要的上下文等。

在运行QEMU时，首先要把原始的二进制机器码转换成可执行的中间语言（TCODE），这个过程称为“翻译”，每次翻译的结果就是一个TB。当CPU开始执行指令时，QEMU会把当前PC值所对应的TB载入到缓存中，然后以一种类似于解释执行的方式运行该TB。如果TB成功执行完毕，CPU就会跳转到下一个TB的起始位置，并重复以上操作；如果执行过程中发生异常、中断或者其他特殊情况，CPU会根据具体情况处理并跳转到相应的地址执行。


TB是按照一定的规则来划分的，这个规则通常由QEMU的翻译器（translator）来定义。在x86架构中，一个TB通常对应着原始二进制代码中的多条指令，它们具有以下特征：

1.  从TB起始地址开始执行不会受到任何控制流指令（如跳转、返回等）的影响，也就是说，TB内部的所有指令都可以顺序执行。    
2.  TB内部不存在条件分支或循环结构（例如if语句、while语句等），即TB内部的所有指令都是线性的。
3.  TB内部没有访问全局共享状态的指令，也就是说，TB内部的所有指令只涉及到CPU本地状态的修改和读取。
4.  TB内部不存在异常、中断或其他需要特殊处理的指令，也就是说，TB内部的所有指令都可以直接通过模拟CPU执行得到正确的结果。

先前我们已经获得了_start的地址 我们只需要在qemu执行TranslateBlock的时候去判断是不是起始地址就能启动forksrv了
```c
/* Execute a TB, and fix up the CPU state afterwards if necessary */
static inline tcg_target_ulong cpu_tb_exec(CPUState *cpu, TranslationBlock *itb)
{
    CPUArchState *env = cpu->env_ptr;
    uintptr_t ret;
    TranslationBlock *last_tb;
    int tb_exit;
    uint8_t *tb_ptr = itb->tc_ptr;

    AFL_QEMU_CPU_SNIPPET2;


-----------
#define AFL_QEMU_CPU_SNIPPET2 do { \
    if(itb->pc == afl_entry_point) { \
      afl_setup(); \
      afl_forkserver(cpu); \
    } \
    afl_maybe_log(itb->pc); \
  } while (0)

```

所以qemu里的插桩是以TB为单位 而不是以BB 而且插桩不再是以随机数xor 而是以pc地址做一些算术运算

在tb_find里有另外一个宏 
这个其实不需要很care 应该就是在产生一个新的TB的时候告诉父进程用我们这里的cache 不需要重新生成了（？
```c
             */
            tb = tb_htable_lookup(cpu, pc, cs_base, flags);
            if (!tb) {
                /* if no translated code available, then translate it now */
                tb = tb_gen_code(cpu, pc, cs_base, flags, 0);
                AFL_QEMU_CPU_SNIPPET1;
            }
```



# persistent mode
通过一些代码修改 变成in-process的模样 具体就是修改单次处理成直接在循环内向外`read(0, buf ...)` 子进程每一次运行后都会`raise(SIGSTOP)` forksrv会处理这个信号

```c
  cc_params[cc_par_cnt++] = "-D__AFL_LOOP(_A)="
    "({ static volatile char *_B __attribute__((used)); "
    " _B = (char*)\"" PERSIST_SIG "\"; "
    "__attribute__((visibility(\"default\"))) "
    "int _L(unsigned int) __asm__(\"__afl_persistent_loop\"); "
    "_L(_A); })";

```

在编译的时候会传入这个宏 `-D`是传入一个宏 会用`__afl_persistent_loop`函数替换源程序中的 `__AFL_LOOP`函数 实现persistent mode
