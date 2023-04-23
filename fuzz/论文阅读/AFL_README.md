AFL README翻译，原文件：[google/AFL: american fuzzy lop - a security-oriented fuzzer (github.com)](https://github.com/google/AFL)

## 1）fuzzing挑战

Fuzzing是识别现实世界软件中安全问题最强大且经过验证的策略之一。迄今为止，它已经发现了绝大部分安全关键软件中的远程代码执行和提权漏洞。

不幸的是，Fuzzing也相对浅显。盲目的随机变异使得很难到达被测试代码的某些代码路径，从而让一些漏洞无法通过这种技术来发现。

针对这个问题已经有很多尝试。早期的一种方法是语料库精选，由Tavis Ormandy首创。该方法依赖于覆盖率信号，从一个海量、高质量的候选文件语料库中选择有趣的种子，并通过传统方式进行Fuzzing。这种方法非常有效，但需要有可用的语料库。此外，块覆盖率度量只提供程序状态的非常简单的理解，对于长期指导Fuzzing努力来说帮助不大。

其他更复杂的研究集中在诸如程序流分析（“共同执行”）、符号执行或静态分析等技术上。所有这些方法在实验环境中极为有前途，但在实际使用中往往会遇到可靠性和性能问题，并且目前并没有提供“愚蠢”Fuzzing技术的可行替代方案。

## 2）AFL-Fuzz方法

American Fuzzy Lop（AFL）是一种暴力Fuzzing技术，配合一种非常简单但非常牢固的基于插装的遗传算法。它使用一种修改过的边界覆盖率来轻松捕捉程序控制流的微妙、局部变化。

简单来说，整个算法可以概括为：
* 将用户提供的初始测试用例加载到队列中，
* 从队列中取出下一个输入文件，
* 尝试将测试用例修剪为不会改变程序测量行为的最小大小，
* 使用平衡且经过深入研究的各种传统Fuzzing策略重复突变文件，
* 如果生成的任何变异导致插装记录了新的状态转换，则将变异后的输出作为队列中的新条目添加。
* 返回第2步。

发现的测试用例也会定期清除，以消除已经被新的、更高覆盖率的发现所取代的测试用例，并进行其他几个基于插装的工具优化步骤。

在Fuzzing过程的副产品中，该工具创建了一个小型、自包含的有趣测试用例语料库。这些对于向其他耗费劳动力或资源密集型的测试体系——例如测试浏览器、办公应用、图形套件或闭源工具等——提供有用帮助。

该Fuzzer经过完整测试，可提供比盲目Fuzzing或仅覆盖率工具更出色的开箱即用性能。

## 3）插桩程序

当源代码可用时，可以通过一个工具注入插桩（instrumentation），该工具在任何第三方代码的标准构建过程中可作为gcc或clang的替代品。

插装的性能影响相当小；结合afl-fuzz实现的其他优化，大多数程序可以比传统工具更快地进行模糊测试。

重新编译目标程序的正确方法可能会因构建过程的具体细节而异，但几乎通用的方法是：

```shell
$ CC=/path/to/afl/afl-gcc ./configure 
$ make clean all
```

对于C ++程序，您还需要设置  `CXX= /path/to/afl/afl-g++` 。

clang包装器（afl-clang和afl-clang++）可以以同样的方式使用；clang用户也可以选择利用更高性能的插装模式，如llvm_mode/README.llvm中所述。

当测试库时，需要找到或编写一个简单的程序，该程序从标准输入或文件中读取数据并将其传递给被测试的库。在这种情况下，重要的是将此可执行文件与插装库的静态版本链接，或者确保正确的.so文件在运行时加载（通常通过设置LD_LIBRARY_PATH来实现）。最简单的选项是进行静态构建，通常可以通过以下方式实现：

```shell
$ CC=/path/to/afl/afl-gcc ./configure --disable-shared
```

在调用“make”时设置 `AFL_HARDEN = 1` 将导致CC包装器自动启用代码硬化选项，这使得检测简单的内存错误更容易。Libdislocator是AFL附带的一个辅助库（请参阅libdislocator/README.dislocator），可以帮助发现堆损坏问题。

PS：建议ASAN用户查看[notes_for_asan.txt](https://github.com/google/AFL/blob/master/docs/notes_for_asan.txt)文件中的重要警告。

## 4）二进制程序的插桩

无源代码时，fuzzer 提供了对黑盒二进制文件的快速、即时的仪器化实验支持。这是通过运行在较少人知道的“用户空间仿真”模式下的 QEMU 版本实现的。

QEMU 是一个与 AFL 分开的项目，但可以通过以下方式方便地构建该功能：

```shell
$ cd qemu_mode 
$ ./build_qemu_support.sh
```

有关附加说明和注意事项，请参见 qemu_mode/README.qemu。

该模式大约比编译时仪器化慢 2-5 倍，不利于并行化，并可能存在其他一些问题。

## 5）选择初始测试用例

为了正确运作，fuzzer 需要一个或多个起始文件，其中包含目标应用程序通常期望的输入数据的良好示例。有两个基本规则：
* 保持文件较小。小于 1 KB 是理想的，尽管不是严格必需的。关于为什么大小很重要的讨论，请参见 [perf_tips.txt](https://github.com/google/AFL/blob/master/docs/perf_tips.txt)。
* 仅在它们在功能上不同的情况下使用多个测试用例。使用五十个不同的度假照片来模糊图像库是毫无意义的。

您可以在附带此工具的 testcases/ 子目录中找到许多良好的起始文件示例。

PS. 如果有大量数据集可用于筛选，则可能需要使用 afl-cmin 实用程序以识别一组功能不同的子集文件，以在目标二进制文件中执行不同的代码路径。

## 6）Fuzzing二进制程序

模糊测试过程本身由 afl-fuzz 实用程序执行。该程序需要一个只读目录，其中包含初始测试用例，一个单独的位置来存储其发现结果，以及要测试的二进制文件的路径。

对于直接从 stdin 接受输入的目标二进制文件，通常的语法是：

```shell
$ ./afl-fuzz -i testcase_dir -o findings_dir /path/to/program [...params...]
```

对于从文件中获取输入的程序，请使用 '@@' 在目标命令行中标记输入文件名应放置的位置。fuzzer 将替换此内容：

```shell
$ ./afl-fuzz -i testcase_dir -o findings_dir /path/to/program @@
```

您还可以使用 -f 选项将突变的数据写入特定文件。如果程序期望特定的文件扩展名，则这很有用。

非插桩的二进制文件可以在 QEMU 模式（在命令行中添加 -Q）或传统的盲目 fuzzer 模式（指定 -n）下进行模糊测试。

您可以使用 -t 和 -m 来覆盖执行进程的默认超时和内存限制；可能需要触及这些设置的极少数目标示例包括编译器和视频解码器。

有关优化模糊测试性能的提示，请参见 [perf_tips.txt](https://github.com/google/AFL/blob/master/docs/perf_tips.txt)。

请注意，afl-fuzz 首先执行一系列确定性模糊步骤，可能需要几天时间，但往往会产生整洁的测试用例。如果您想要快速并脏结果，类似于 zzuf 和其他传统 fuzzer，请在命令行中添加 -d 选项。

## 7）输出

请参阅 [status_screen.txt](https://github.com/google/AFL/blob/master/docs/status_screen.txt) 文件，以了解如何解释显示的统计信息并监视进程的状态。特别是如果任何 UI 元素突出显示为红色，请务必查阅此文件。

模糊测试过程将继续进行，直到按下 Ctrl-C。至少，您要允许 fuzzer 完成一个队列循环，这可能需要几个小时到一周左右的时间。

在输出目录中创建了三个子目录，并实时更新：
* queue/ - 包含每个不同执行路径的测试用例，以及由用户提供的所有起始文件。这是第 2 节中提到的合成语料库。在将此语料库用于其他任何目的之前，您可以使用 afl-cmin 工具将其缩小到更小的大小。该工具将找到提供等效边缘覆盖的更小的子集文件。
* crashes/ - 导致被测试程序接收致命信号（例如 SIGSEGV、SIGILL、SIGABRT）的唯一测试用例。条目按收到的信号分组。
* hangs/ - 导致被测试程序超时的唯一测试用例。在被分类为挂起之前的默认时间限制是 1 秒和 -t 参数值中的较大者。该值可以通过设置 AFL_HANG_TMOUT 进行微调，但这很少是必需的。

如果相关执行路径涉及任何以前记录的故障中未看到的状态转换，则崩溃和挂起被认为是“唯一”的。如果可以通过多种方式达到单个错误，则在过程早期会有一些计数膨胀，但这应该很快消减。

崩溃和挂起的文件名与父项、非失败队列条目相关联。这应有助于调试。

当您无法重现由 afl-fuzz 发现的崩溃时，最可能的原因是您未设置与工具使用的相同的内存限制。请尝试：

```
$ LIMIT_MB=50 
$ ( ulimit -Sv $[LIMIT_MB << 10]; /path/to/tested_binary ... )
```

将 LIMIT_MB 更改为匹配传递给 afl-fuzz 的 -m 参数。在 OpenBSD 上，还要将 -Sv 改为 -Sd。

任何现有的输出目录也可用于恢复已中止的作业；请尝试：

```
$ ./afl-fuzz -i- -o existing_output_dir [...etc...]
```

如果已安装 gnuplot，则可以使用 afl-plot 为任何活动模糊测试任务生成漂亮的图形。有关其外观的示例，请参见 http://lcamtuf.coredump.cx/afl/plot/

## 8）并行化Fuzzing

每个 afl-fuzz 实例大约占用一个核心。这意味着在多核系统上，需要并行化才能充分利用硬件。如何在多个核心或多个网络机器上对常见目标进行模糊测试的提示，请参阅 [parallel_fuzzing.txt](https://github.com/google/AFL/blob/master/docs/parallel_fuzzing.txt)

并行模糊测试模式还提供了一种简单的方式，将 AFL 与其他 fuzzer、符号执行引擎或共性执行引擎等进行交互；同样，请参阅 [parallel_fuzzing.txt](https://github.com/google/AFL/blob/master/docs/parallel_fuzzing.txt) 的最后一节以获取提示

## 9）Fuzzer字典

默认情况下，afl-fuzz 突变引擎针对紧凑的数据格式进行优化，例如图像、多媒体、压缩数据、正则表达式语法或 shell 脚本。它不太适合具有特别冗长和冗余措辞的语言，尤其是包括 HTML、SQL 或 JavaScript 在内。

为避免构建语法感知工具的麻烦，afl-fuzz 提供了一种方式，以选项字典种子方式启动模糊测试进程，其中包含与目标数据类型相关联的语言关键字、魔术头或其他特殊令牌，并在过程中使用它来重建底层语法：[http://lcamtuf.blogspot.com/2015/01/afl-fuzz-making-up-grammar-with.html](http://lcamtuf.blogspot.com/2015/01/afl-fuzz-making-up-grammar-with.html)

要使用此功能，首先需要在 dictionaries/README.dictionaries 中讨论的两种格式之一中创建字典，然后通过命令行中的 -x 选项将 fuzzer 指向它。

(该子目录中已经提供了几个常见的字典。)

没有提供更结构化的底层语法描述的方法，但 fuzzer 可能会基于仅仅基于插桩反馈来找出其中的一些。实际上，这在实践中可行，比如说：[http://lcamtuf.blogspot.com/2015/04/finding-bugs-in-sqlite-easy-way.html](http://lcamtuf.blogspot.com/2015/04/finding-bugs-in-sqlite-easy-way.html)

PS：即使没有给出明确的字典，afl-fuzz 也会尝试通过在确定性字节翻转期间非常密切地观察插桩来提取输入语料库中现有的语法令牌。这适用于某些类型的解析器和语法，但不像 -x 模式那样好。

如果真的很难找到字典，则另一个选项是让 AFL 运行一段时间，然后使用伴随实用程序的 token 捕捉库。有关此内容，请参见 libtokencap/README.tokencap。

## 10）crash分类

基于覆盖率的崩溃分组通常产生一个小数据集，可以快速手动或使用非常简单的 GDB 或 Valgrind 脚本进行分类。每个崩溃也可追溯到其队列中的父项非崩溃测试用例，使诊断故障更容易。

话虽如此，需要承认某些模糊测试中的崩溃可能很难在没有大量调试和代码分析工作的情况下快速评估其可利用性。为了帮助完成这项任务，afl-fuzz 支持一种非常独特的“crash探索”模式，使用 -C 标志启用该模式。

在此模式下，fuzzer 以一个或多个崩溃测试用例作为输入，并使用其反馈驱动的模糊策略，非常快速地枚举可以在程序中到达的所有代码路径，同时保持其处于崩溃状态。

不会导致崩溃的变异被拒绝；同样被拒绝任何不影响执行路径的更改。

输出是一小组文件，可以非常快速地检查攻击者对错误地址的控制程度，或者是否有可能越过初始的越界读取，以及看看底层是什么。

还有就是，为了进行测试用例的最小化，请尝试使用 afl-tmin。该工具可以以非常简单的方式操作：

```shell
$ ./afl-tmin -i test_case -o minimized_result -- /path/to/program [...]
```

该工具适用于崩溃和非崩溃测试用例。在崩溃模式下，它将高兴地接受插装和非插装二进制文件。在非崩溃模式下，最小化器依靠标准 AFL 插桩来使文件更简单，而不会改变执行路径。

最小化器以与 afl-fuzz 兼容的方式接受 -m、-t、-f 和 @@ 语法。

AFL 的另一个最新添加是 afl-analyze 工具。它接受一个输入文件，尝试顺序翻转字节，并观察测试程序的行为。然后，它根据哪些部分似乎是关键的，哪些不是，对输入进行颜色编码。虽然不是绝对可靠，但通常可以快速了解复杂文件格式的情况。关于其操作的更多信息可在 [technical_details.txt](https://github.com/google/AFL/blob/master/docs/technical_details.txt) 的末尾找到。

## 11）crash的其他内容

模糊测试是一种发现非崩溃设计和实现错误的精彩且未充分利用的技术。通过修改目标程序以在以下情况下调用 abort()，可以发现不少有趣的漏洞：
* 当两个大数字库使用相同的 fuzzer 生成的输入时产生不同的输出，
* 当多次请求一个相同的输入图像进行解码时，图像库会产生不同的输出，
* 当迭代地序列化和反序列化 fuzzer 提供的数据时，序列化/反序列化库无法产生稳定的输出，
* 当要求压缩库压缩并解压特定的数据块时，它产生与输入文件不一致的输出。

实现这些或类似的健全性检查通常需要很少的时间；如果您是特定软件包的维护者，则可以使用 `#ifdef FUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION`（该标志也与 libfuzzer 共享）或 `#ifdef __AFL_COMPILER`（仅适用于 AFL）将此代码设置为条件。

## 12）常见风险

请记住，与许多其他计算密集型任务类似，模糊测试可能会对您的硬件和操作系统造成压力。特别是：
* 您的 CPU 将运行的很烫，并需要适当的冷却。在大多数情况下，如果冷却不足或停止正常工作，则 CPU 速度将自动降频。尽管如此，特别是在较不适合的硬件（笔记本电脑、智能手机等）上进行模糊测试时，发生问题并不完全不可能。
* 目标程序可能最终会以异常方式抓取数GB的内存或用垃圾文件填满磁盘空间。AFL 尝试强制实施基本内存限制，但无法防止每一种可能的错误。底线是，在不能接受数据丢失的情况下不应在系统上进行模糊测试。
* 模糊测试涉及对文件系统进行数十亿次的读写。在现代系统上，这通常会被严重缓存，导致相当适度的“物理”I/O - 但有许多因素可能会改变这个方程式。您有责任监视潜在的麻烦；对于非常重的 I/O，许多 HDD 和 SSD 的寿命可能会缩短。

在 Linux 上监视磁盘 I/O 的好方法是使用“iostat”命令：

```shell
$ iostat -d 3 -x -k [...optional disk ID...]
```

## 13）局限性和改进领域

以下是介绍 AFL 的一些重要注意事项：
* AFL 通过检查第一个生成的进程由于信号（SIGSEGV、SIGABRT 等）死机来检测错误。那些安装了这些信号自定义处理程序的程序可能需要注释掉相关代码。同样，模糊目标产生的子进程中的错误可能会逃避检测，除非您手动添加一些代码来捕捉它们。
* 与任何其他暴力工具一样，如果加密、校验和、密码签名或压缩用于完全包装要测试的实际数据格式，模糊器提供的覆盖率有限。
  为了解决这个问题，您可以注释掉相关检查（参见 experimental/libpng_no_checksum/ 获取灵感）；如果不可能，则还可以编写后处理程序，如 experimental/post_library/ 中所述。
* ASAN 和 64 位二进制文件存在某些不利的权衡。这不是 afl-fuzz 的特定错误；请查阅 notes_for_asan.txt 获取建议。
* 没有直接支持模糊测试网络服务、后台守护程序或需要 UI 交互才能正常工作的交互式应用程序的方法。您可能需要进行简单的代码更改，使其以更传统的方式运行。Preeny 也可以提供一个相对简单的选项，请参见：[https://github.com/zardus/preeny](https://github.com/zardus/preeny)
* 有关修改基于网络的服务的一些有用提示也可以在以下网址找到：[https://www.fastly.com/blog/how-to-fuzz-server-american-fuzzy-lop](https://www.fastly.com/blog/how-to-fuzz-server-american-fuzzy-lop)
* AFL 不输出可读性良好的覆盖数据。如果要监视覆盖率，请使用 Michael Rash 的 afl-cov：[https://github.com/mrash/afl-cov](https://github.com/mrash/afl-cov)
* 有时，智能机器会反抗他们的创造者。如果发生这种情况，请咨询 [http://lcamtuf.coredump.cx/prep/。](http://lcamtuf.coredump.cx/prep/%E3%80%82)

此外，请参见 INSTALL 获取特定平台的提示。
