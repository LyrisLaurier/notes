> USENIX分析今年论文，看引用文献，有没有共同之处，然后去看这些经典文献

# FRAMESHIFTER

[FRAMESHIFTER: Security Implications of HTTP/2-to-HTTP/1 Conversion Anomalies | USENIX](https://www.usenix.org/conference/usenixsecurity22/presentation/jabiyev)

github：[bahruzjabiyev/frameshifter: Grammar-based HTTP/2 fuzzer with mutation ability (github.com)](https://github.com/bahruzjabiyev/frameshifter) 

## 前置

思考三个问题
* 帧序列何内容操作是否会导致HTTP/2-to-HTTP/1转换异常
* 导致转换异常的操作以及原因
* 利用转换异常可以进行哪些攻击

>Frame sequence——帧序列：在计算机网络中，数据通常被分割成许多小的单元（称为“帧”），并通过网络传输。这些帧需要按特定的顺序组合起来才能正确地重新构建原始数据。因此，这些帧的顺序和组合方式就形成了一个帧序列。
>Content manipulation——内容操作”或“内容处理：在计算机领域中，内容指代数据或信息。内容操作/处理通常是对这些数据或信息进行一系列改变、转换或优化的过程，以满足特定需求或要求。在HTTP协议中，内容操作/处理可能包括压缩、解压缩、加密、解密、编码和解码等操作。

贡献
* 基于语法的针对HTTP/2的fuzzer
* 全面的、系统的学习HTTP协议转换异常的方法
* 发现有关HTTP/2转换的新的攻击向量，并调查了原因
* 演示攻击并和受影响的供应商协调措施

重点在利用 HTTP/2-to-HTTP/1 请求转换实现HTTP Request Smuggling (HRS)。相关研究

调查范围
* 关注会影响request body的header和frame，如content-length和transfer-encoding的头
* 反向代理可以进行协议转换

定义了正常/非正常的协议转换
* 从一个流生成HTTP/1请求。如果该请求有body，则必须有content-length头且其纸为body长度，或者有transfer-encoding且遵循chunked format。满足这两条就算正常
* 不满足就算异常

## 技术细节

FRAMESHIFTER实现的功能：从grammar中创建输入，变异输入

grammar类似

```
<start> ::= <sequence>  
<sequence> ::= <headers><data> | <headers>  
<headers> ::= <method><path><host>  
<method> ::= :method=GET | :method=POST  
<path> ::= /echo  
<host> ::= echo.com  
<data> ::= hello,world! | bye,world!
```

变异方式有两种
* 帧序列变异：随机位置添加或删除帧
* 帧内容变异：设定了一部分可变字段，才可变区域插入、替换等

## 实验过程

![Pasted image 20230414221300](https://user-images.githubusercontent.com/94295495/233821536-7859a6c6-1a38-4095-96b6-a3b8e12d5ac2.png)

仅帧序列变异的实验：
1. 生成阶段只生成语义有效的片段
2. 每次突变利用随机数（1~4），因为删除容易破坏有效序列，因此插入操作的权重高设为90%
3. 插入帧有10种类型，其中HEADERS（携带HTTP请求或响应的头部信息）、CONTINUATION（延续HEADERS帧中未完成的头部块）、DATA（正文数据）占比大为75

帧序列+帧内容变异的实验：
1. 二者最大变异数量均为2，总的来说就是4，与上一个实验一样
2. 对the content-length和transfer-encoding的header部分的名称和值请求方法这些位置，添加字符（仅限于ASCII）

## 发现

异常情况
* HTTP/1的请求包没有body，长度小于content-length
* 有body，长度小于content-length
* 缺少最后一个chunk
* 缺少最后一个chunk，同时也缺少了终止符号CRLF
* chunk数据缺失
* content-length长度无效
* content-length终止内容由LF而不是CRLF，LF终止很多服务器不支持
* content-length header内容重复
* content-length heade值重复
* 单个流因此请求应该只有一个，但是有多个请求

输入
* 缺少END_STREAM
* 对有缺少END_STREAM标志的有效流，不检查较大content-length和接收的data帧中较小的字节数是否匹配，会自动转发content-length与body长度不一致的请求
* END_HEADERS后的带有HEADERS，会让代理服务器停止流处理，直接转发请求
* 仅第一个data时
* 没有突变过滤器：请求的标头的值或终止符无效
* 没有重复检查，包含多个content-length头
* GET请求，有HEADERS帧

输入与产生的异常对应

![Pasted image 20230415095658](https://user-images.githubusercontent.com/94295495/233821542-147072cc-59db-46f4-b6f1-2471905f1985.png)


异常原因
* 代理模式：缓冲、流式处理。缓冲模式下代理会等待完整的客户端请求然后转发到上层，流式处理模式下，为提高效率会立刻转发不会等待全部请求。部分代理接受END_HEADERS标志后转发不完整请求
* 错误处理：反向代理如何处理错误也可能产生异常，可以将请求转发到上层后短时间内关闭连接，或向客户端发错误响应而不转发请求。前一种处理方式可能产生异常
* 不充分验证：有的代理允许无关字符的出现
* 故障重试：ATS遇到一些特定输入时，会触发一个错误并将请求转发，但是ATS因缺陷没有关闭连接且无法接收请求，ATS会重试来获取请求以返回给客户端，还会在transfer-encoding添加“chunked”

## 攻击测试

DoS：变异帧序列导致反向代理发送的HTTP/1请求包的body不完整，因此服务器会等待（等不到），从而耗尽服务器资源

Request Blackholing：

Query-of-Death attack：

Cache-Poisoned Denial-of-Service attack：

Response Queue Poisoning：

HTTP Request Smuggling：

# BrakTooth

[BrakTooth: Causing Havoc on Bluetooth Link Manager via Directed Fuzzing | USENIX](https://www.usenix.org/conference/usenixsecurity22/presentation/garbelini)

## 整体设计

目标有两类
* Host OS
* BT SoC Firmware

解耦设计（Decoupling Packet Generation from Fuzzing）：传统的fuzzer需要对通信环境进行建模，包括对数据包的生成和处理等，而BrakTooth的涉及使得可以对目标设备和任意第三方堆栈直接实时进行fuzzing。BT涉及动态协议如LCAP，通信过程中只有上下文可用，因此不能用静态seed。解耦设计将数据包生成和fuzzing分开处理，数据生成可以由第三方协议堆栈来实现（其实就是第三方软件）

BrakTooth 针对关于无线协议的数据包，是用 wireshark 处理的

反馈过程中使用粒子群优化（PSO）来细化变异数据字段的概率

状态映射的算法：

![Pasted image 20230430223212](https://user-images.githubusercontent.com/94295495/235461894-2e954d2c-182e-4dba-af62-c0918dbdf48e.png)

* 为数据包P创建一个状态标签，在Mref中创建s→s'的转换
* 3-4行解析P，5-13遍历所有layer看是否能匹配上Mu中名字。确定层后13-15行提取相关字段的值和类型，16行生成状态标签

突变：不同状态下针对不同字段进行突变→设置解码树，设置不同层每个字段的突变概率不一样。这些突变概率的调整用到了PSO，从覆盖率的角度去衡量突变概率的有效性
变异操作：字段变异时用的随机数
复制：复制的数据包会在不适合的时间节点发送，利用乱序发送使得数据包影响更大，注意洪水问题

监控上直接监视蓝牙片上的系统（SoC），而不是空中架设设备监控

测试过程中一旦发现漏洞或不符合要求，可以通过exploitation框架再现该场景。这个过程利用了TX/RX解剖勾子（TX/RX dissection hooks）注入或改变数据包来实现。

> “Hook”指的是解剖勾子（dissection hooks），它们是一种用于拦截、检查和修改网络数据包的工具。这些勾子可以被添加到网络协议分析器（如Wireshark）中，以便在数据包通过时执行自定义操作。在本文中，利用框架使用TX/RX解剖勾子来注入或改变数据包，以实现漏洞利用。

## 实时fuzzing接口

是针对BT协议来设计的

# Stateful Greybox Fuzzing

[Stateful Greybox Fuzzing | USENIX](https://www.usenix.org/conference/usenixsecurity22/presentation/ba)

没有协议规范下发现协议漏洞

# 引用论文

基于grammar的上下文无关的fuzzer CFG：
Andreas Zeller, Rahul Gopinath, Marcel Böhme, Gordon Fraser, and Christian Holler. Fuzzing with Grammars. The Fuzzing Book, 2022. https://www.fuzzingbook.org/html/Grammars.html.

基于grammar的fuzzer
* Christian Holler, Kim Herzig, and Andreas Zeller. Fuzzing with Code Fragments. In USENIX Security Symposium), 2012
* Xuejun Yang, Yang Chen, Eric Eide, and John Regehr.  Finding and Understanding Bugs in C Compilers. In ACM SIGPLAN Conference on Programming Language Design and Implementation, 2011.
* Ivan Fratric. Domato. GitHub Repository, 2021. https://github.com/googleprojectzero/domato.

与FRAMESHIFTER相似的fuzzer：
* Cornelius Aschermann, Tommaso Frassetto, Thorsten Holz, Patrick Jauernig, Ahmad-Reza Sadeghi, and Daniel Teuchert. NAUTILUS: Fishing for Deep Bugs with Grammars. In The Network and Distributed System Security Symposium, 2019
* Bahruz Jabiyev, Steven Sprecher, Kaan Onarlioglu, and Engin Kirda. T-Reqs: HTTP Request Smuggling with Differential Fuzzing. In ACM Conference on Computer and Communications Security, 2021

