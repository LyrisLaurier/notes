# fuzz

[模糊测试 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/%E6%A8%A1%E7%B3%8A%E6%B5%8B%E8%AF%95)

## 定义

模糊测试 （fuzz testing, fuzzing）是一种软件测试技术。其核心思想是自动或半自动的生成随机数据输入到一个程序中，并监视程序异常，如崩溃，断言(assertion)失败，以发现可能的程序错误，比如内存泄漏
模糊测试工具主要分为两类
* 变异测试（mutation-based），变异测试通过改变已有的数据样本去生成测试数据
* 生成测试（generation-based），生成测试则通过对程序输入的建模来生成新的测试数据

目前主要的三种模糊测试技术
* 黑盒随机模糊：对正确格式的输入数据进行随机变异，然后用这些变异的输入运行程序，看是否能够触发异常
* 基于语法的模糊：是模糊复杂格式输入的替代方法，需要指定输入格式的输入语法，还指定哪些输入部分要进行模糊化以及如何模糊化。基于语法的模糊生成器会生成许多新的输入，每个输入满足语法编码的约束。基于语法的fuzzing通过模糊生成器的用户的创造力和专业知识来指导fuzzing
* 白盒模糊处理：动态地执行测试下的程序，并从执行过程中遇到的条件分支收集输入约束。然后，系统地逐个否定所有这些约束，并使用约束求解器求解，其解被映射到执行不同程序执行路径的新输入。使用系统搜索技术重复这个过程，试图扫描程序的所有可行的执行路径。与黑盒随机模糊相比，白盒模糊通常更精确，可以运行更多的代码，从而发现更多的bug

## 测试用例

随机数据的类型有：超长字符串、格式化字符串、浮点数、超大数、特殊字符、unicode编码……

测试用例构造
* 单项测试用例
* 多项测试用例
* 同类字符不必区分法则：如前所述0-9这类数字，a-z这类字母都是同类，不是很有必要测了一个再去测其他
* 长度不必过细法则：选几个有代表性的就行没必要长度100来一个测试用例，长度101来一个测试用例。

一般测试（不管理普通测试还是渗透测试）是不会强行把软件撕开一个口子去测试的，测试就是就着目标系统提供的接口对接口中的各项值进行修改以此生成测试用例去进行测试。

## 测试过程

初始种子->选择种子->变异数据是否符合要求->符合的注入目标程序->目标程序是否发生异常->输出日志

模糊生成器fuzzer
* fuzzer根据code coverage的结果，调整corpus数据，以达到code coverage最大化
* fuzzer监控一些指令和函数，比如比较指令、switch/case指令、memcmp/strcmp函数等，智能的修改数据
* 支持多线程、多进程
* 不能有随机性，不能受全局状态的影响，否则效率会大幅度降低
* 配合Sanitize，监控程序运行情况：AddressSanitizer和UndefinedBehaviorSanitizer

最简单的模糊测试是通过命令行，网络包或者事件向一个程序输入一段随机比特流。这种技术目前依然是有效的发现程序错误的方法。 另一个常见且易于实现的技术是通过随机反转一些比特或者整体移动一些数据块来变异已有的输入数据。最有效的模糊测试需要能够理解被测试对象的格式或者协议。这可以通过阅读设计规格来实现。基于设计规格的模糊工具包含完整的规格，并通过基于模型的测试生成方法去遍历规格，并在数据内容，结构，消息，序列中引入一些异常。这种“聪明的”模糊测试也被称作健壮性测试，句法测试，语法测试以及错误注入。
文件格式与网络协议是最常见的测试目标，但任何程序输入都可以作为测试对象。常见的输入有环境变量，鼠标和键盘事件以及API调用序列。甚至一些通常不被考虑成输入的对象也可以被测试，比如数据库中的数据或共享内存

# 工具

## American Fuzzy Lop（AFL）

### 安装

（我用的ubuntu18）
github源码：[google/AFL: american fuzzy lop - a security-oriented fuzzer (github.com)](https://github.com/google/AFL) 
或者官网[american fuzzy lop (coredump.cx)](https://lcamtuf.coredump.cx/afl/)下载最新版本后，放到ubuntu里解压，windows不能用，然后make

```shell
tar zxvf afl-latest.tgz
make
sudo install make
```

### 测试-有源码程序

[(1条消息) AFL (American fuzzy lop) 二进制程序模糊测试工具学习___lifanxin的博客-CSDN博客](https://blog.csdn.net/A951860555/article/details/119666269) 

```c
#include <stdio.h>
int main() {
    char buf[100] = {0};
    gets(buf);   // stack overflow
    if (buf[0] == 'A')
        printf("hello\n");
    else
        printf("NO A\n");
    return 0;
}
```

编译。编译c程序使用afl-gcc，编译c++程序使用afl-g++

```shell
afl-gcc test.c -o test # 编译
mkdir fuzz_in # 创建输入的文件夹
echo "hello" > fuzz_in/testcase # 给一个输入样例
afl-fuzz -i fuzz_in -o fuzz_out ./test # 开始测试
```

使用afl-fuzz进行测试，-i指明测试用例的目录，-o指明测试结果的存放目录。对于直接从终端获取输入的程序来说，我们需要在testcase_dir目录下新建一个文件，文件的内容就是程序的输入，文件命名不唯一。fuzz_out会自动生成。如果afl-fuzz出现报错时，修改一下即可

```shell
sudo su
echo core >/proc/sys/kernel/core_pattern
```

执行afl-fuzz后会出现下面的界面，正在进行模糊测试。这是跑了五分钟的结果，可以看到已经有两个crash了。不想跑了就ctrl+c退出

<img width="360" alt="Pasted image 20221120095000" src="https://user-images.githubusercontent.com/94295495/236790275-02a68902-73ec-47b5-8dad-16edd59e218e.png">


### afl界面参数说明

process timing：fuzzing测试的时间消耗

overall results：汇总了fuzzing测试的执行结果
* cycles done：fuzzing的轮数，品红色表示处于the first pass。如果在the first pass后有新发现，进入子过程，会变黄色。所有子过程完成后会变成蓝色，最后变成绿色表面长时间无新动作，可以ctrl+c关闭了

cycle progress：展示当前队列中fuzzer的位置，以及放弃了的超时输入

map coverage：程序的覆盖率

stage progress：进一步展示fuzzer的执行过程细节
   * now trying指明当前所用的变异输入方法
      * calibration：在fuzzing测试前的阶段，主要检查执行路径检测异常，建立基线执行速度
      * trim L/S：fuzzing测试前的阶段，修建测试用例使其更短，但保证裁剪后仍能达到相同的执行路径。L表示length长度，S表示stepover步距，其值与文件大小是相关的
      * bitflip L/S：确定性的比特位翻转。以S为增量，L长度的bit数被翻转。有几种变型模式：1/1, 2/1, 4/1, 8/8, 16/8, 32/8
      * arith L/8：确定性的算术运算。AFL会尝试去减去或者加上一些整数使其为8bit/16bit/32bit的值，步距永远是8bits
      * interest L/8：确定性的值覆盖。AFL自身保留了一些Interesting的8bit/16bit/32bit的值，用这些值去覆盖原有的测试用例，步距永远是8bits
      * extras：确定性的字典注入。AFL自身有一个字典，也可以用-x选项来指明使用用户提供的字典
      * havoc：固定长度的堆叠随机扭曲。该阶段会尝试位翻转，用随机数或者Interesting的整数去覆盖，块删除，块复制，以及字典的相关操作
      * splice：最后一种策略。在上述策略都执行完后将会执行该策略，它和havoc差不多，不过它会首先将队列中的两个随机输入先拼接在一起
      * sync – 这个是并行执行的策略选项，通过-M或者-S选项进行指定。该策略并不会涉及到真正的fuzzing，会导入从另一个fuzzer得到的输出和测试用例
   * exec speed：测试用例的执行速度，正常不低于500 exec/sec，长时间低于100的话可以查看perf_tips.txt来寻求优化

findings in depth：一些路径、crash信息

fuzzing strategy yields：进一步展示AFL所作的工作，采用的策略情况

path geometry：路径测试的相关信息
* levels：测试等级
* pending：还没有经过fuzzing的输入数量
* pend fav：fuzzer感兴趣的输入数量
* own finds：在fuzzing过程中新找到的，或者是并行测试从另一个实列导入的
* imported：n/a表示不可用，即没有导入
* stability：相同输入是否产生相同的行为，一般是100%。如果低于100%且变红，需要查官方文档寻找解决方法

### 输出文件

分析一下fuzz_out里的文件，先看下结构

<pre><font color="#8AE234"><b>ubuntu@ubuntu</b></font>:<font color="#729FCF"><b>~/Desktop/pwn/fuzz</b></font>$ tree fuzz_out
<font color="#729FCF"><b>fuzz_out</b></font>
├── <font color="#729FCF"><b>crashes</b></font>
│   ├── id:000000,sig:06,src:000000,op:havoc,rep:128
│   ├── id:000001,sig:06,src:000001,op:havoc,rep:128
│   ├── id:000002,sig:11,src:000000,op:havoc,rep:64
│   └── README.txt
├── fuzz_bitmap
├── fuzzer_stats
├── <font color="#729FCF"><b>hangs</b></font>
├── plot_data
└── <font color="#729FCF"><b>queue</b></font>
    ├── id:000000,orig:testcase
    └── id:000001,src:000000,op:havoc,rep:32,+cov
</pre>

queue：存放fuzzer生成的所有不同执行路径的测试用例+自己一开始构造的测试用例
crashes：存放造成程序崩溃的测试用例，根据产生的信号不同进行分类
hangs：存放造成程序超时的测试用例
剩下的文件记录fuzzer工作的一些信息

### 测试-无源码程序

AFL依赖QEMU实现了这个功能，qemu是一个仿真器。下载AFL的源码后可以很方便的通过其自带脚本完成安装，如下命令所示，执行该命令后会去下载指定版本的qemu然后安装

```shell
cd qemu_mode
./build_qemu_support.sh
```

操作和前面一样，不过编译用gcc，afl-fuzz加上参数-Q

```shell
mkdir fuzz_in
echo "hello" > fuzz_in/testcase
gcc test.c -o test
afl-fuzz -i fuzz_in -o fuzz_out -Q ./test
```

# 基本框架结构

[Fuzzbook系列（3）：Fuzz的基本框架结构 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/security-management/260600.html) 

## Runner类

使用给定的输入来执行某些特定的程序（要接受测试的某些程序或函数）

```python
import subprocess  

class Runner(object):  
    # Test outcomes  
    PASS = "PASS"  
    FAIL = "FAIL"  
    UNRESOLVED = "UNRESOLVED"  
    def __init__(self):  
        """Initialize"""  
        pass  
    def run(self, inp):  
        """Run the runner with the given input"""  
        return (inp, Runner.UNRESOLVED)  

class PrintRunner(Runner):  
    def run(self, inp):  
        """Print the given input"""  
        print(inp)  
        return (inp, Runner.UNRESOLVED)

class ProgramRunner(Runner):  
    def __init__(self, program):  
        """Initialize.  `program` is a program spec as passed to `subprocess.run()`"""  
        self.program = program    
    def run_process(self, inp=""):  
        """Run the program with `inp` as input.  Return result of `subprocess.run()`."""  
        return subprocess.run(self.program,  
                              input=inp,  
                              stdout=subprocess.PIPE,  
                              stderr=subprocess.PIPE,  
                              universal_newlines=True)  
    def run(self, inp=""):  
        """Run the program with `inp` as input.  Return test outcome based on result of `subprocess.run()`."""  
        result = self.run_process(inp)  
        if result.returncode == 0:  
            outcome = self.PASS  
        elif result.returncode < 0:  
            outcome = self.FAIL  
        else:  
            outcome = self.UNRESOLVED  
        return (result, outcome)

class BinaryProgramRunner(ProgramRunner):  
    def run_process(self, inp=""):  
        """Run the program with `inp` as input.  Return result of `subprocess.run()`."""  
        return subprocess.run(self.program,  
                              input=inp.encode(),  
                              stdout=subprocess.PIPE,  
                              stderr=subprocess.PIPE)
```

```python
cat = ProgramRunner(program="cat")  
print (cat.run("hello"))
```

输出

<pre><font color="#8AE234"><b>ubuntu@ubuntu</b></font>:<font color="#729FCF"><b>~/Desktop/pwn/fuzz</b></font>$ python3 test1.py
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;hello&apos;, stderr=&apos;&apos;), &apos;PASS&apos;</pre>

## Fuzzer类

fuzzer生成数据并送至runner

```python
class Fuzzer(object):  
    def __init__(self):  
        pass  
    def fuzz(self):  
        """Return fuzz input"""  
        return ""  
    def run(self, runner=Runner()):  
        """Run `runner` with fuzz input"""  
        return runner.run(self.fuzz())  
    def runs(self, runner=PrintRunner(), trials=10):  
        """Run `runner` with fuzz input, `trials` times"""  
        # Note: the list comprehension below does not invoke self.run() for subclasses  
        # return [self.run(runner) for i in range(trials)]        outcomes = []  
        for i in range(trials):  
            outcomes.append(self.run(runner))  
        return outcomes  
  
class RandomFuzzer(Fuzzer):  
    def __init__(self, min_length=10, max_length=100,  
                 char_start=32, char_range=32):  
        """Produce strings of `min_length` to `max_length` characters  
           in the range [`char_start`, `char_start` + `char_range`]"""        self.min_length = min_length  
        self.max_length = max_length  
        self.char_start = char_start  
        self.char_range = char_range  
    def fuzz(self):  
        string_length = random.randrange(self.min_length, self.max_length + 1)  
        out = ""  
        for i in range(0, string_length):  
            out += chr(random.randrange(self.char_start,  
                                        self.char_start + self.char_range))  
        return out
```

创建一个模糊器

```python
random_fuzzer = RandomFuzzer(min_length=20, max_length=20)  
for i in range(10):  
    print(random_fuzzer.fuzz())
```

<pre><font color="#8AE234"><b>ubuntu@ubuntu</b></font>:<font color="#729FCF"><b>~/Desktop/pwn/fuzz</b></font>$ python3 test2.py
#+;&quot;09*81.))9&amp;-+ #3&lt;
*?2!*&gt;$5,379 511&apos; %?
(6)=&quot;4 -0*/.,&lt;8)&apos;#(&amp;
-))+7,/;43&gt;?#);70/&amp;.
#05? 42%!=*9#844455)
$* 0&amp;2?5(%#6 8:)$4-1
&lt;4== (*9(2&lt;=(+#:-,)$
&apos;&gt;97(,&apos;.3;-,/%27+!1&apos;
1&apos;04&gt;1/&gt;9*.?/0&gt;+%4(%
*5*;$:6-9?8$61+9$4/!
</pre>

以cat为例，生成输入发送的cat中

```python
random_fuzzer = RandomFuzzer(min_length=20, max_length=20)
for i in range(10):
    inp = random_fuzzer.fuzz()
    result, outcome = cat.run(inp)
    assert result.stdout == inp
    assert outcome == Runner.PASS
print(random_fuzzer.run(cat))
```

<pre><font color="#8AE234"><b>ubuntu@ubuntu</b></font>:<font color="#729FCF"><b>~/Desktop/pwn/fuzz</b></font>$ python3 test2.py
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;(92: ;;!80;6+3.8?;&lt;;&apos;, stderr=&apos;&apos;), &apos;PASS&apos;)
</pre>

使用runs可以重复执行模糊测试多次

```python
print(random_fuzzer.runs(cat,10))
```

<pre><font color="#8AE234"><b>ubuntu@ubuntu</b></font>:<font color="#729FCF"><b>~/Desktop/pwn/fuzz</b></font>$ python3 test2.py
[(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;5;?9+&quot;=,0-6&gt;\&apos;6 ?,#1:&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&quot;%(0;7-+*07=,934=3 &apos;9&quot;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;$&gt;&gt;7&amp;-&lt;6-$+(&gt; &lt;0=+;&quot;&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;-(,;9!$!:&amp;,3=03 &gt;85)&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;*#:/&quot;7(/+?00:!1;$&quot;?#&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;8-/*987/#5%0.#1&lt;&quot;5:5&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&quot;47;!)&apos;!447;&lt;29&apos;&gt;&gt; 6!&quot;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;1(:$/7)10*;/%-$ :$99&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;&amp;?8);&quot;!;!4&amp;.762/)$0-&apos;, stderr=&apos;&apos;), &apos;PASS&apos;), 
(CompletedProcess(args=&apos;cat&apos;, returncode=0, stdout=&apos;\&apos;9%&quot;=.162\&apos;)313)1(8+-&apos;, stderr=&apos;&apos;), &apos;PASS&apos;)]
</pre>

