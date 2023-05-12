> 本章笔记只做源码整体分析，逐行分析是通过在源码中写注释的方式

# 编译

## afl-gcc

### main()

afl-gcc.c

```c
int main(int argc, char** argv) {
  ...
  find_as(argv[0]); //负责查找 as 并用 afl-as 替换
  edit_params(argc, argv); //根据argv[0]选择编译器，并为编译器添加参数
  execvp(cc_params[0], (char**)cc_params);
  FATAL("Oops, failed to execute '%s' - check your PATH", cc_params[0]);
  return 0;
}
```

## afl-as

> 这里留个点，apple系统相关代码没有看，暂时不想考虑特殊问题

### main()

afl-as.c

```c
/* Examine and modify parameters to pass to 'as'. Note that the file name is always the last parameter passed by GCC, so we exploit this property to keep the code simple. */
static void edit_params(int argc, char** argv){...} //修改参数

/* Process input file, generate modified_file. Insert instrumentation in all the appropriate places. */
static void add_instrumentation(void){...} //插桩

int main(int argc, char** argv) {
  //根据当前时间设置随机种子
  gettimeofday(&tv, &tz);
  rand_seed = tv.tv_sec ^ tv.tv_usec ^ getpid();
  srandom(rand_seed);

  edit_params(argc, argv);

  if (inst_ratio_str) { //是插桩数量
    if (sscanf(inst_ratio_str, "%u", &inst_ratio) != 1 || inst_ratio > 100) 
      FATAL("Bad value of AFL_INST_RATIO (must be between 0 and 100)");

  }

  setenv(AS_LOOP_ENV_VAR, "1", 1);

  /* When compiling with ASAN, we don't have a particularly elegant way to skip
     ASAN-specific branches. But we can probabilistically compensate for
     that... */
  if (getenv("AFL_USE_ASAN") || getenv("AFL_USE_MSAN")) {
    sanitizer = 1;
    inst_ratio /= 3;
  }

  if (!just_version) add_instrumentation();

  if (!(pid = fork())) {
    execvp(as_params[0], (char**)as_params);
    FATAL("Oops, failed to execute '%s' - check your PATH", as_params[0]);
  }
}
```

mian 函数中涉及到三个 afl 定义的环境变量，有对这三个环境变量的判断
* AFL_INST_RATIO：用于控制插桩代码数量
* AS_LOOP_ENV_VAR：用于检测是否出现无线循环，1为出现，通常是因为as被重定向到当前目录或或父目录导致
* AFL_KEEP_ASSEMBLY：是否保留汇编代码

如果使用的是 ASAN 或 MSAN （两个都是内存错误检测工具），会减少插桩来弥补 ASAN 特定的分支

整体逻辑上：在插桩前需要设置好 seed、`edit_params()` 配置参数、插桩数量、其他小配置，然后调用 `add_instrumentation()` 插桩，然后利用 `fork()` 生成子进程，调用原生的 as 编译

### edit_params()

关于 `edit_params()` 

```c
static void edit_params(int argc, char** argv) {
  u8 *tmp_dir = getenv("TMPDIR"), *afl_as = getenv("AFL_AS");

  /* Although this is not documented, GCC also uses TEMP and TMP when TMPDIR
     is not set. We need to check these non-standard variables to properly
     handle the pass_thru logic later on. */

  //即判断TMPDIR，没有就获取TEMP、TMP
  if (!tmp_dir) tmp_dir = getenv("TEMP");
  if (!tmp_dir) tmp_dir = getenv("TMP");
  if (!tmp_dir) tmp_dir = "/tmp";

  //开始分析参数，as_params存放的是as的参数列表
  as_params = ck_alloc((argc + 32) * sizeof(u8*));
  as_params[0] = afl_as ? afl_as : (u8*)"as";
  as_params[argc] = 0;

  for (i = 1; i < argc - 1; i++) {...} //根据as的参数开始配置

  input_file = argv[argc - 1]; //获取文件名，后面有if判断是对获取过程中的处理

  //拼接参数，分配内存
  modified_file = alloc_printf("%s/.afl-%u-%u.s", tmp_dir, getpid(),(u32)time(NULL)); 
}
```

* `use_clang_as`/`afl_as` 选择是哪个as
* 会判断是否为苹果系统，因为 MacOS 上有些不同

### 插桩

关于 `add_instrumentation()`

```c
static void add_instrumentation(void) {
  static u8 line[MAX_LINE]; //存储每行汇编
  inf = fopen(input_file, "r"); //输入文件
  outfd = open(modified_file, O_WRONLY | O_EXCL | O_CREAT, 0600); //输出文件，权限为仅读写
  
  //开始循环读取每一行代码
  while (fgets(line, MAX_LINE, inf)) {
    //后面几个判断，根据条件调整变量数值，这些变量决定最后一个if是否进行插桩
    //符合插桩条件
    if (!pass_thru && !skip_intel && !skip_app && !skip_csect && instr_ok &&
	    instrument_next && line[0] == '\t' && isalpha(line[1])) {
      //插桩
      fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32, R(MAP_SIZE));
      instrument_next = 0;
      ins_lines++;
    }
    //还有很多if去判断
    
    //上面判断过条件了，根据上面的条件结果决定是否跳过这行汇编
    if (skip_intel || skip_app || skip_csect || !instr_ok ||
        line[0] == '#' || line[0] == ' ') continue;
    //后面还有一些条件
    
    //如果插桩过，在末尾添加payload，用于覆盖率信息收集
    if (ins_lines) fputs(use_64bit ? main_payload_64 : main_payload_32, outf);
}
```

关于是否插桩的判断
* 位于 .text 段内会插入
* 是条件分支指令如 jcc 会插入
* 遇到符合特定格式的标签如函数名、分支标签时插入。即是 .L 或 .LBB，
* 不是注释指令

里面涉及的一些变量
* pass_thru：文件部分设置的，为0才行

```c
if (strncmp(input_file, tmp_dir, strlen(tmp_dir)) && strncmp(input_file, "/var/tmp/", 9) && strncmp(input_file, "/tmp/", 5)) pass_thru = 1;
```

* instr_ok 当前行是否跳过。为1才行
* skip_intel:.intel_syntax 即是 intel 就为1，.att_syntax 即 AMD 就为0。为0才行
* skip_app：注释信息是 `#APP` 则为1，为 `#NO_APP` 为0。为0才行
* skip_next_label：在判断 text 段和后面特定格式标签/函数名时会改变这个变量

```c
if (!clang_mode && instr_ok && !strncmp(line + 2, "p2align ", 8) && isdigit(line[10]) && line[11] == '\n') skip_next_label = 1;
```

* instrument_next 是上一行生成的，不符合 .L0: 或 LBB0_0: 样式的跳转目的，或者 skip_next_label 为0，instrument_next都会被设置为1。为1才行

其中，下面这行代码是插桩的关键代码

```c
fprintf(outf, use_64bit ? trampoline_fmt_64 : trampoline_fmt_32, R(MAP_SIZE));
// 有写 define R(x) (random() % (x))，取随机数
```

* use_64bit 选择是32还是64位的模板
* trampoline_fmt_32 和 trampoline_fmt_64 变量是用于存储覆盖率分析代码模板的字符串
* 与 MAP_SIZE 参数组合起来生成最终的覆盖率分析代码
* R(MAP_SIZE) 是一个宏定义，其作用是返回一个介于 0 到 MAP_SIZE-1 之间的随机整数。这个值将作为覆盖率分析代码中的跳转偏移量，从而使得每个插桩指令都有一个唯一的标识符

### afl-as.h

其中 trampoline_fmt_64 和 trampoline_fmt_32 即插桩的内容定义在 afl-as.h 中

* trampoline_fmt_64 如下

```text

/* --- AFL TRAMPOLINE (64-BIT) --- */

.align 4

leaq -(128+24)(%%rsp), %%rsp
movq %%rdx,  0(%%rsp)
movq %%rcx,  8(%%rsp)
movq %%rax, 16(%%rsp)
movq $0x%08x, %%rcx
call __afl_maybe_log
movq 16(%%rsp), %%rax
movq  8(%%rsp), %%rcx
movq  0(%%rsp), %%rdx
leaq (128+24)(%%rsp), %%rsp

/* --- END --- */

```

* trampoline_fmt_32 如下

```text

/* --- AFL MAIN PAYLOAD (32-BIT) --- */

.text
.att_syntax
.code32
.align 8

__afl_maybe_log:

  lahf
  seto %al

/* Check if SHM region is already mapped. */

  movl  __afl_area_ptr, %edx
  testl %edx, %edx
  je    __afl_setup

__afl_store:

  /* Calculate and store hit for the code location specified in ecx. There
     is a double-XOR way of doing this without tainting another register,
     and we use it on 64-bit systems; but it's slower for 32-bit ones. */

```

## afl-fuzz

afl-fuzz.c

