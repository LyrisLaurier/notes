上一篇简单介绍了fuzz，并展现了AFL的安装、使用过程，后面会对AFL进行详细分析

参考文章：

从使用的角度分析：[【AFL（二）】AFL 工具分析 afl-cmin、afl-tmin - 未配妥剑，已入江湖 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wayne-tao/p/11889718.html) 

很粗略，是初期对AFL整体分析：[AFL-fuzz工具分析_Chen_zju的博客-CSDN博客](https://blog.csdn.net/Chen_zju/article/details/80791268) 

写的算是有内容，但是有点乱，且逻辑上也不太清楚：[AFL内部实现细节小记 - 记事本 (rk700.github.io)](http://rk700.github.io/2017/12/28/afl-internals/)

师兄发的：[AFL 源代码速通笔记 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/624286070)

# AFL工作流程

## 整体流程

```C
// main函数初始化、选项处理
// 执行input文件夹下预先准备的test cas，生成初始化的queue和bitmap
// 通过cull_queue对queue进行精选，减小input的量
while(1){
	// cull_queue()对queue进行精选，选出favored
	// 判断queue_cur是否为NULL，是则表示完成对队列的遍历
	queue_cycle++;
	// 初始化相关参数，重新开始遍历队列
	// fuzz queue_cur对应的input文件，fuzz_one()
	// 判断是否结束，并更新queue_cur和current_entry
}
```

## 预料删选 cull_queue()

`cull_queue()` 遍历 `top_rated[]` 中的queue，发现新边缘的入口就标记为 favored，下次遍历时这些被标记的就有更多的执行机会。本质上是一种贪婪算法在筛选
关键代码在 line 1309 - 1328

这个过程中用 `trace_bits[]` 记录执行的queue的路径情况、访问哪些边缘。 `top_rated[]` 中存储每条边的访问情况

## 测试用例变异 fuzz_one()

```C
// 判断是否有 pending_favored 和 queue_cur ，选择是否跳过分析
// 加载test case
// CALIBRATION
trim_case();// 修剪，减小input大小
// case打分performance score()，如果该queue已经完成deterministic阶段，则直接跳到havoc阶段
// bitfilp位翻转，同时检测token
// arithmetic算术变异，加/减一个整数
// interest对比特位进行替换，替换为interest的数据（提前定义好的，通常为数据类型的边界）
// dictionary字典替换或插入，来源用户提供的token、自动检测生成的token
// havoc多种变异方式结合
// splice拼接两个seed并进行havoc
```

第一步的 queue 跳过规则：
* 有 pending_favored：99%跳过 fuzzed /不是 favaored 的 queue；
* 无 pending_favored：
	* 95%跳过 fuzzed 的不是 favored 的 queue
	* 75%跳过没有 fuzzed 不是 favored 的 queue
	* 不跳过 favored 的 queue

trim_case()：输入文件修剪。内置的修剪器按可变长度和 stepover 顺序删除数据块
还有有一个独立的 afl-tmin 工具可以进行修剪：如果初始输入崩溃了目标二进制文件，afl-tmin将以非插装模式运行，只需保留任何能产生更简单文件但仍然会使目标崩溃的调整。如果目标是非崩溃的，那么这个工具使用一个插装的模式，并且只保留那些产生完全相同的执行路径的微调

```C
// 尝试将大块数据与大量的步骤归零。从经验上看，这可以通过在以后的时间内抢先进行更细粒度的工作来减少执行的次数。
// 执行块删除过程，减少块大小和stepovers，二进制搜索式(binary-search-style)。
// 通过计算唯一字符并尝试用零值批量替换每个字符来执行字母表规范化。
// 作为最后的结果，对非零字节执行逐字节标准化。
```

calculate_score()：对 case 进行打分，根据 case 的执行速度、bitmap大小、case产生时间、路径深度等因素对 case 打分，用来调整 havoc 阶段

bitflip 阶段按位翻转
* bitflip 1/1，每次翻转1个bit，按照每1个bit的步长从头开始
* bitflip 2/1，每次翻转相邻的2个bit，按照每1个bit的步长从头开始
* bitflip 4/1，每次翻转相邻的4个bit，按照每1个bit的步长从头开始
* bitflip 8/8，每次翻转相邻的8个bit，按照每8个bit的步长从头开始，即依次对每个byte做翻转
* bitflip 16/8，每次翻转相邻的16个bit，按照每8个bit的步长从头开始，即依次对每个word做翻转
* bitflip 32/8，每次翻转相邻的32个bit，按照每8个bit的步长从头开始，即依次对每个dword做翻转

自动检测token：在进行bitflip 1/1变异时，对于每个byte的最低位(least significant bit)翻转还进行了额外的处理：如果连续多个bytes的最低位被翻转后，程序的执行路径都未变化，而且与原始执行路径不一致(检测程序执行路径的方式可见上篇文章中“分支信息的分析”一节)，那么就把这一段连续的bytes判断是一条token

effector map：bitflip 8/8变异时会生成。在对每个byte进行翻转时，如果其造成执行路径与原始路径不一致，就将该 byte 在 effector_map 中标记为1有效，否则记为0无效的

arithmetic 阶段：根据目标大小的不同，分为多个子阶段
* arith 8/8，每次对8个bit进行加减运算，按照每8个bit的步长从头开始，即对文件的每个byte进行整数加减变异
* arith 16/8，每次对16个bit进行加减运算，按照每8个bit的步长从头开始，即对文件的每个word进行整数加减变异
* arith 32/8，每次对32个bit进行加减运算，按照每8个bit的步长从头开始，即对文件的每个dword进行整数加减变异

interest阶段
* interest 8/8，每次对8个bit进替换，按照每8个bit的步长从头开始，即对文件的每个byte进行替换
* interest 16/8，每次对16个bit进替换，按照每8个bit的步长从头开始，即对文件的每个word进行替换
* interest 32/8，每次对32个bit进替换，按照每8个bit的步长从头开始，即对文件的每个dword进行替换

替换的内容是提前设定好的一些特殊数

```C
/* List of interesting values to use in fuzzing. */

# define INTERESTING_8 \ -128, /* Overflow signed 8-bit when decremented */ 
	\ -1, /* */ \ 0, /* */ \ 1, /* */ \ 16, /* One-off with common buffer size */ 
	\ 32, /* One-off with common buffer size */ 
	\ 64, /* One-off with common buffer size */ 
	\ 100, /* One-off with common buffer size */ 
	\ 127 /* Overflow signed 8-bit when incremented */
# define INTERESTING_16 \ -32768, /* Overflow signed 16-bit when decremented */ 
	\ -129, /* Overflow signed 8-bit */ 
	\ 128, /* Overflow signed 8-bit */ 
	\ 255, /* Overflow unsig 8-bit when incremented */
	 \ 256, /* Overflow unsig 8-bit */ 
	 \ 512, /* One-off with common buffer size */ 
	 \ 1000, /* One-off with common buffer size */ 
	 \ 1024, /* One-off with common buffer size */ 
	 \ 4096, /* One-off with common buffer size */ 
	 \ 32767 /* Overflow signed 16-bit when incremented */
# define INTERESTING_32 \ -2147483648LL, /* Overflow signed 32-bit when decremented */ 
	\ -100663046, /* Large negative number (endian-agnostic) */ 
	\ -32769, /* Overflow signed 16-bit */ 
	\ 32768, /* Overflow signed 16-bit */ 
	\ 65535, /* Overflow unsig 16-bit when incremented */ 
	\ 65536, /* Overflow unsig 16 bit */ 
	\ 100663045, /* Large positive number (endian-agnostic) */ 
	\ 2147483647 /* Overflow signed 32-bit when incremented */

static s8  interesting_8[]  = { INTERESTING_8 };
static s16 interesting_16[] = { INTERESTING_8, INTERESTING_16 };
static s32 interesting_32[] = { INTERESTING_8, INTERESTING_16, INTERESTING_32 };
```

dictionary阶段
* user extras (over)，从头开始，将用户提供的tokens依次替换到原文件中
* user extras (insert)，从头开始，将用户提供的tokens依次插入到原文件中
* auto extras (over)，从头开始，将自动检测的tokens依次替换到原文件中

对于上述的这些变异，会自动判断是否需要进行这些阶段，若效果相似是会被跳过的

havoc 阶段结合以上各种变异，随机数量、随机变异方式

最后是 splice 阶段：从队列中随机选一个 seed 与当前的 seed 组合，如果两者差别不大，就再重新随机选一个；如果两者相差比较明显，那么就随机选取一个位置，将两者都分割为头部和尾部。最后，将当前文件的头部与随机文件的尾部拼接起来

## 代码插桩

afl-gcc 对代码进行插桩编译，afl-gcc.c 文件其实是 gcc 的 wrapper

编译的过程由 gcc 和 as 共同完成
* gcc 负责：源代码 → 预编译后的代码 → 汇编代码
* as 负责：汇编代码 → 机器码 → 链接后的文件

afl-gcc

```C
int main(int argc, char** argv) {
	find_as(argv[0]); // 查找 as 这个二进制程序并用 afl-as 替换
	edit_params(argc, argv); // 修改参数
	execvp(cc_params[0], (char**)cc_params); //调用原生的 gcc 进行编译
	FATAL("Oops, failed to execute '%s' - check your PATH", cc_params[0]);
	return 0;
}
```

`edit_params()` 中主要有两点：给 gcc 添加额外的参数、根据参数设置一些 flag

完成汇编后调用 afl-as 对汇编进行插桩：仍用 `edit_params()` 编辑参数，用 `add_instrumentation` 对汇编代码进行插桩，完成插桩后用 fork 生成子进程并调用原生的 as 进行编译

`add_instrumentation`：插桩的逻辑是循环读取每行汇编代码，然后对特定的汇编代码进行插桩。确保代码位于 text 内存段，如果是 main 函数或分支就进行插桩，如果是注释或强制跳转指令不插桩

