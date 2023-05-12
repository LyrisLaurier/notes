[american fuzzy lop (coredump.cx)](https://lcamtuf.coredump.cx/afl/)

[[AFL README翻译]]

# 白皮书

[[AFL technical_details翻译]]

# AFL++

https://www.usenix.org/conference/woot20/presentation/fioraldi

## 先进fuzz介绍

是针对这篇论文来说当时比较出名的，现在来看其实已经过去好几年了。。。

### AFL

覆盖率反馈：结合边缘覆盖与执行次数结合，次数存于bucktes。利用甲醛最小集合覆盖（weighted minimum set cover）的近似计算来收集有用的测试用例，将速度和大小作为权重。还有一个 trimming 阶段，试图减少每个测试用例的大小但不改变覆盖率，来加快速度。

变异分为两种：  
* deterministic：对测试用例内容的修改，如位翻转、加减法操作
* havoc：变异是随机堆叠（stacked）的，包含对测试用例大小的修改。后期AFL在 splicing 阶段还可能会合并两个测试用例用于havoc

forkserver：每次执行新的测试用例时，需要重新启动新进程并初始化，很耗费资源和时间。利用IPC机制将forkserver注入到目标进程中，新测试用例需要启动时forkserver先自我复制以实现将子进程给这个测试用例执行，这样子进程会集成父进程的状态如文件、加载库等，可以减少开销

persistent模式：不同于forkserver，它不会每次都靠子进程，而是选择利用循环每次迭代执行一个测试用例

### AFLFast

智能调度（smart scheduling）：基于覆盖率引导的fuzzer会使用不同的优先级算法来调度 fuzzing 管道（pipeline）中的不同元素。调度的目标通常是通过智能选择测试用例来提高覆盖率和发现 bug 的概率

AFLFast提出“stress low-frequency paths”，重点关注较少被访问的路径。有两个主题
* 合适的算法去stress low-frequency paths
* 是否可以调整每个 seed 生成输入的数量

### MOpt

引入变异调度（mutation scheduling），利用粒子群算法（Particle Swarm Optimization）

