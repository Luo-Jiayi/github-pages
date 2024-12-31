---
title: "pa/pa2"
date: 2024-11-27
---

> ctrl+z : background vim, "fg" : open back

# 理解指令执行的过程
## 一条指令的历程 
`cpu_exec()`最终调用了`isa_exec_once()`.这些核心的操作,包括指令的取指与译码环节,在`isa/$ISA/inst.c`中可以找到.
- 取指:`inst_fetch()`从内存中读取定长
- 译码:`decode_exec()`匹配指令模式,为`rs1, rs2, imm`赋值,调用函数执行指令操作(想想正则表达式!)
  ```c
    INSTPAT_START();
    INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc, U, R(rd) = s->pc + imm);
    // ...
    INSTPAT_END();
  ```
  看起来这么易懂得感谢宏.看看他们原本面目:
    ![](./pa2_fig/INSTPAT.png)
    `INSTPAT`(instrution pattern)调用`pattern_decode()`函数分析当前执行指令的`pattern`;如果分析出的组成部分与预置的指令匹配, 则由`INSTPAT_MATCH()`执行`__VA_ARGS__`所述的指令解析与行为
    ![](./pa2_fig/Decode.png)
    `include/cpu/decode.h`定义的`pattern_decode`将一条32-bit的指令切分成多个'图层'或'蒙版',即`key=值为1, mask=值指定, shift= opcode到最低位的距离`. 这样的三个图层被用于滤除`don't care terms`与`shift`,复现与对比指令格式. 
    回到`inst.c`. 当我们确定当前指令为某指令后,需要 (现在回上去复习一下INSTPAT的用法吧) 
    - 对操作数`R(rs1), R(rs2), imm`进行赋值: 不同类型指令操作数格式不同.值得庆幸的是,riscv操作数的位置还是很固定的
    ![](./pa2_fig/decode_operand.png)
    注意到,读取立即数需要进行一些符号位扩展与拼接的操作(`BITS`和`SEXT`). 
    `SEXT`定义一个匿名结构体,其中包含一个名为 `n` 位宽为 `len` 的位域(`n`实际上是`int64_t`但只能用`len`位存储); 然后实例化结构体于`__x`,将输入`x`赋给`__x.n`并转换为`uint64_t`(`__x.n`的高位被截断与符号扩展)
    ![](./pa2_fig/BITSSEXT.png)
    这里尤其要留心U型,B型和J型,他们的最低位不是0位(U型是组合使用于很大的立即数所以只有高位,J型用于跳转指令显然不能跳个半截); 另外, 它们身上的位数旋转使得六型指令更好地对齐了
    - 执行: 比如`R(rd) = s->pc + imm`. 有时候我们需要调用函数来写内存(`Mw`)或者写可以反复复用的宏来执行复杂的操作

## 似懂非懂
1. snpc, dnpc是什么关系?
pc指向当前指令,snpc指向下一条静态指令, dnpc动态指向
```
(Decode*) s->pc, s->snpc = pc //这里pc是cpu.pc
s->snpc++ //取指时
s->dnpc = s->snpc //解指前
cpu.pc = s->dnpc //解指后
```


> **字节序**
> 小端序=一个多位数的低位放在小地址,高位放在大地址
> 例如,I2C,SPI和网络传输使用大端序,USB和以太网使用小端序
> 具体参考深入理解计算机第二章
> 对于立即数/>1字节的内存访问,我们都应考虑字节序
> 1. 我们在x86的机器上运行riscv32的NEMU,两者都是小端序. 如果不匹配呢?考虑不同ISA的组合
> 例如运行一个大端序的ISA?x86小端序的内存需要翻转以被正确读入;在大端序机器上运行NEMU也是一样,因为两者不匹配
> 2. mips32和riscv32怎么解决一条指令(32位)放不下32位常数的问题?
>  截断?

# 增加指令以运行程序
使用`am-kernel/tests/`中的程序测试NEMU中的指令实现(验证?):
## 第一个客户程序
第一步,运行dummy.c,需要增加:`jal(j), jalr(ret), sw, addi(li, mv)`

> 遇到的问题体现RTFM的重要性:
> ret反汇编的机器码是`00008067`,但是reader里给的是按照jalr即`?010_????_?110_0111`,两者不符
> 看了riscv-spec,给的就是`func3=000`
> 结论:旧版/翻译误人!不能老是偷懒看中文的

> 然鹅还是不对.是`s->pc`溢出了吧
> ![](./pa2_fig/overflowAt8000_0028.png)
> 果然,`immJ()`位数没数明白

现在是这样了:指令倒也不错,可是确实是HIT BAD TRAP
![](./pa2_fig/JALfixedNoROT1.png)
我们发现`ebreak`指令将`nemu_state`设为`END`,而此时BAD TRAP说明退出检查`nemu_state.halt_ret==0`没有被满足.通过ctags跳转与gdb我们可以更好地追踪,并且找到相关的变量与函数:
`utils.h`定义
```c
struct NEMU_STATE {
	int state, vaddr_t halt_pc, uint32_t halt_ret
}
```
这些正是程序结束log输出的`nemu:(HIT BAD TRAP) at pc = (halt_pc)`!
`src/engine/interpreter/hostcall.c`写入了这两个变量,文件里有两个函数:
```c
set_nemu_state(int state, vaddr_t pc, int halt_ret)
invalid_inst(vaddr_t thispc) //不能识别opcode则state=ABORT
```
现在思考:为什么`ebreak`断点要调用函数,设`state=END, halt_ret=$a0`?因为需要检查返回值是否为0来判断有无异常!
所以为什么有异常?猜测与跳转指令相关,检查指令实现将`s->pc = ?`改为`s->dnpc = ?`
完工!
## 继续增加指令
bne指令遇到的问题:由于immB()错误地对每一段扩展符号位,导致某个跳转错误形成死循环;计算pc+gdb发现问题


# 运行时环境与abstract-machine
## 程序运行在不同的架构与平台上

> - Runtime environment  提供执行代码环境的软件平台 	
> - Engine 通过编译或解释来执行代码的运行时环境的组件	
> - Interpreter 逐行读取并执行代码，无需事先编译整个程序的引擎类型


- 运行时环境是什么? 提供执行代码环境的软件平台,解决了内存管理、程序访问变量、过程之间传递参数的机制、接口等问题
- 这里, 运行时环境以库的形式存在. 运行程序所需要的公共要素(比如`halt()`)被抽象成API, 不同的架构各自实现API, 需要的时候只要调用这些API, 就能使用运行时环境提供的相应功能.
  流程: `(在NEMU中)实现硬件功能 -> (在AM中)提供运行时环境 -> (在APP层)运行程序`


## AM的Makefile写的很好...
1. Basic Setup and Checks
  ```makefile
  ### Default to create a bare-metal kernel image
  ifeq ($(MAKECMDGOALS),)
    MAKECMDGOALS  = image
    .DEFAULT_GOAL = image
  endif
  ### 检查:查找am.h文件以判断环境变量`$AM_HOME`是否正确
  ifeq ($(wildcard $(AM_HOME)/am/include/am.h),)
    $(error $$AM_HOME must be an AbstractMachine repo)
  endif
  ### 检查:查找*.mk文件,提取出文件名(notdir去除路径,basename只保留文件名去除后缀), 从而比较输入的$(ARCH)是否匹配*.mk文件给定的ISA-平台组合列表
  ARCHS = $(basename $(notdir $(shell ls $(AM_HOME)/scripts/*.mk)))
  ifeq ($(filter $(ARCHS), $(ARCH)), )
    $(error Expected $$ARCH in {$(ARCHS)}, Got "$(ARCH)")
  endif
  ### 替换-为空格, 并拆分出ISA与PLATFORM变量; 例如: `ARCH=x86_64-qemu -> ISA=x86_64; PLATFORM=qemu`
  ARCH_SPLIT = $(subst -, ,$(ARCH))
  ISA        = $(word 1,$(ARCH_SPLIT))
  PLATFORM   = $(word 2,$(ARCH_SPLIT))
  ### 检查: 查询$SRCS的flavor特性,是否等于undefined(flavor是变量赋值与使用值的方式,包括undefined,recursive,simple)
  ### 如果$SRCS没有赋值(nothing to build),报错
  ifeq ($(flavor SRCS), undefined)
    $(error Nothing to build)
  endif
  ```
2. General Compilation Targets
  ```makefile
  ### Create the destination directory (`build/$ARCH`)
  WORK_DIR  = $(shell pwd)
  DST_DIR   = $(WORK_DIR)/build/$(ARCH)
  $(shell mkdir -p $(DST_DIR))

  ### Compilation targets (a binary image or archive)
  ### 可执行文件?与静态库的生成目标地址
  IMAGE_REL = build/$(NAME)-$(ARCH)
  IMAGE     = $(abspath $(IMAGE_REL))
  ARCHIVE   = $(WORK_DIR)/build/$(NAME)-$(ARCH).a

  ### Collect the files to be linked: object files (`.o`) and libraries (`.a`)
  ### 将am-$ISA-nemu.a,应用程序源文件编译的目标文件,程序依赖的运行库(如abstract-machine/klib/)收集打包成归档文件
  OBJS      = $(addprefix $(DST_DIR)/, $(addsuffix .o, $(basename $(SRCS))))
  LIBS     := $(sort $(LIBS) am klib) # lazy evaluation ("=") causes infinite recursions
  LINKAGE   = $(OBJS) \
    $(addsuffix -$(ARCH).a, $(join \
      $(addsuffix /build/, $(addprefix $(AM_HOME)/, $(LIBS))), \
      $(LIBS) ))
  ```
3. General Compilation Flags
  ```makefile
  ### 交叉编译 e.g., mips-linux-gnu-g++
  AS        = $(CROSS_COMPILE)gcc
  CC        = $(CROSS_COMPILE)gcc
  CXX       = $(CROSS_COMPILE)g++
  LD        = $(CROSS_COMPILE)ld
  AR        = $(CROSS_COMPILE)ar
  OBJDUMP   = $(CROSS_COMPILE)objdump
  OBJCOPY   = $(CROSS_COMPILE)objcopy
  READELF   = $(CROSS_COMPILE)readelf

  ### Compilation flags
  INC_PATH += $(WORK_DIR)/include $(addsuffix /include/, $(addprefix $(AM_HOME)/, $(LIBS)))
  INCFLAGS += $(addprefix -I, $(INC_PATH))

  ARCH_H := arch/$(ARCH).h
  CFLAGS   += -O2 -MMD -Wall -Werror $(INCFLAGS) \
              -D__ISA__=\"$(ISA)\" -D__ISA_$(shell echo $(ISA) | tr a-z A-Z)__ \
              -D__ARCH__=$(ARCH) -D__ARCH_$(shell echo $(ARCH) | tr a-z A-Z | tr - _) \
              -D__PLATFORM__=$(PLATFORM) -D__PLATFORM_$(shell echo $(PLATFORM) | tr a-z A-Z | tr - _) \
              -DARCH_H=\"$(ARCH_H)\" \
              -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
              -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden
  CXXFLAGS +=  $(CFLAGS) -ffreestanding -fno-rtti -fno-exceptions
  ASFLAGS  += -MMD $(INCFLAGS)
  LDFLAGS  += -z noexecstack
  ```
4. Arch-Specific Configurations
  ```makefile
  ### Paste in arch-specific configurations (e.g., from `scripts/x86_64-qemu.mk`)
  -include $(AM_HOME)/scripts/$(ARCH).mk

  ### Fall back to native gcc/binutils if there is no cross compiler
  ifeq ($(wildcard $(shell which $(CC))),)
    $(info #  $(CC) not found; fall back to default gcc and binutils)
    CROSS_COMPILE :=
  endif
  ```
5. Compilation Rules
  ```makefile
  ### Rule (compile): a single `.c` -> `.o` (gcc)

  ### &&(shell AND): 如果前者失败,不会运行后者; $@: 所有参数, $<: 第一个参数?
  $(DST_DIR)/%.o: %.c
  	@mkdir -p $(dir $@) && echo + CC $< 
  	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)

  ### Rule (compile): a single `.cc` -> `.o` (g++)
  $(DST_DIR)/%.o: %.cc
  	@mkdir -p $(dir $@) && echo + CXX $<
  	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

  ### Rule (compile): a single `.cpp` -> `.o` (g++)
  $(DST_DIR)/%.o: %.cpp
  	@mkdir -p $(dir $@) && echo + CXX $<
  	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

  ### Rule (compile): a single `.S` -> `.o` (gcc, which preprocesses and calls as)
  $(DST_DIR)/%.o: %.S
  	@mkdir -p $(dir $@) && echo + AS $<
  	@$(AS) $(ASFLAGS) -c -o $@ $(realpath $<)

  ### Rule (recursive make): build a dependent library (am, klib, ...)
  $(LIBS): %:
  	@$(MAKE) -s -C $(AM_HOME)/$* archive

  ### Rule (link): objects (`*.o`) and libraries (`*.a`) -> `IMAGE.elf`, the final ELF binary to be packed into image (ld)
  $(IMAGE).elf: $(OBJS) $(LIBS)
  	@echo + LD "->" $(IMAGE_REL).elf
  	@$(LD) $(LDFLAGS) -o $(IMAGE).elf --start-group $(LINKAGE) --end-group

  ### Rule (archive): objects (`*.o`) -> `ARCHIVE.a` (ar)
  $(ARCHIVE): $(OBJS)
  	@echo + AR "->" $(shell realpath $@ --relative-to .)
  	@$(AR) rcs $(ARCHIVE) $(OBJS)

  ### Rule (`#include` dependencies): paste in `.d` files generated by gcc on `-MMD`
  -include $(addprefix $(DST_DIR)/, $(addsuffix .d, $(basename $(SRCS))))
  ```
  
## 将AM的默认模式设置成batch mode
在sdb的代码中注意到: `set_batch_mode()` 会直接`cmd_c(NULL)`从sdb返回, 不经历调试
那么nemu的Makefile是怎么选择batch mode和'sdb mode'的?
在`nemu/scripts/native.mk`中有 `ARGS ?= --log, ARGS += --diff`, 将默认的模式选择写入`ARGS`中 (例如将日志输出到某个文件); 运行时, 我们可以通过`make run ARGS=...`加上新的要求.
因此, 同样的道理, 我们可以在AM的`scripts/platform/nemu.mk`中加上类似的设置: 加上`NEMUFLAGS := --batch`实现批处理
 
# 基础设施(2)
## iringbuf
`iringbuf`在cpu文件夹中新建的文件内实现, 并与 `CONFIG_ITRACE_COND` 编译选项相关联

in `src/cpu/iringbuf.h`
```c
#ifndef __CPU_IRINGBUF_H__
#define __CPU_IRINGBUF_H__

#include <common.h>
typedef struct {
	int len;	// char * 16
	int width;	// char [128]
	char **buf;
	int position;
} Ringbuf;

void init_iringbuf();

#define RINGBUFFER_PRINT(R) int i;\
	for(i = 0; i < R->len; i ++) {\
		if(i == R->position) printf("--->");\
		printf("%s\n", R->buf[i]);\
	}

void iringbuf_update(char *logbuf);
void iringbuf_free();

#endif
```
in `src/cpu/iringbuf.c`
```C
#include <cpu/iringbuf.h>
#define RB_LEN 16
#define RB_WIDTH 128
#ifdef CONFIG_ITRACE_COND
static Ringbuf *iringbuf;
void init_iringbuf()
{
    iringbuf = calloc(1, sizeof(Ringbuf));
    iringbuf->len = RB_LEN;
    iringbuf->width = RB_WIDTH;
    iringbuf->position = 0;
    iringbuf->buf = (char**)calloc(iringbuf->len, sizeof(char*));
	assert(iringbuf->buf != NULL);
	for(int i = 0; i < RB_LEN; i ++) {
		iringbuf->buf[i] = (char*)calloc(iringbuf->width, sizeof(char));
		assert(iringbuf->buf[i] != NULL);
	}
}

void iringbuf_update(char *logbuf) {
	iringbuf->position = (iringbuf->position + 1) % iringbuf->len;
	strncpy(iringbuf->buf[iringbuf->position], logbuf, iringbuf->width - 1); // write log to buf
	iringbuf->buf[iringbuf->position][iringbuf->width-1] = '\0';
}

void iringbuf_free() {
	RINGBUFFER_PRINT(iringbuf);
	for(int i = 0; i < RB_LEN; i ++)
		free(iringbuf->buf[i]);
	free(iringbuf->buf);
	free(iringbuf);
}
#endif
```
## mtrace
很简单, 在vaddr.c中添加这样的代码打印log即可; 并与 `CONFIG_MTRACE_COND` 编译选项相关联
```C
#ifdef CONFIG_MTRACE_COND	
	printf("memory write:\t to %08x\t length = %d;\t write:%08x\n", addr, len, data);
#endif
```
## ftrace
讲义介绍了ftrace的流程:
1. 从.strtab开始查找符号表(.symtab似乎比.strtab多了很多空??)
2. 符号的value即地址
3. 对于jal或jalr指令中一个给定的地址, 如果地址在[value, value + size)范围内, 则返回函数名(符号名); 否则, 返回???

发现am的Makefile生成$(IMAGE).elf, 也就是我们需要读入的.elf文件
我们首先在sdb/monitor代码中init, 也就是在初始化ftrace时从ELF文件中读出符号表和字符串表
### ELF 是什么
ELF文件将节头位置与数量存在起始处的ELF头, 节头以数组的形式连续存放
- symbol table: 存放'静态 全局'的符号
- string table: 节头(例如符号表)的name项为offset量, 指向一个新的节, 也就是存放字符串的区域
![](./pa2_fig/typical_elf.jpg)
### 解析ELF
1. 读取文件: 如果要读取很多数据, 可以通过file mapping `fopen->fseek->ftell->mmap->fclose` 将文件内容映射到虚拟内存, 就可以关闭文件读写流进行后续操作; 我们只需要读取符号表`.symtab`和字符串表`.strtab`, 可以通过`fseek->fread`迅速跳转到对应位置
2. 读取ELFN_hdr: 校验ELF文件格式->读取`e_shoff`->跳转
3. 读取ELFN_shdr: 对于从`e_shoff`开始, `e_shnum`个结构体的数组, 我们逐一读取, 找到`shdr.sh_type == SHT_SYMTAB`的结构体读取`sh_offset, sh_size`, 找到`shdr.sh_type == SHT_STRTAB`的结构体的`sh_offset`来录入字符串表 (也有可能读取`SHT_DYNSYM`, 但我们的程序不输出这个)
4. 读取Sym: 对`sym.st_info == STT_FUNC`的条目, 记录下`st_value`(函数地址), `st_size`(函数尺寸)和`strtab_offset + st_name`对应的字符串(函数名), 作为ftrace的基准查找表
### 检查指令跳转
完成初始化后, 每遇到跳转, 需要根据对照表判断
1. 怎么知道跳转? 可以从`Decode *s->logbuf = "$pc: inst3 inst2 inst1 inst0 jal ra, imm"`中读, 因为itrace的过程也读取了这个变量; 但似乎有点麻烦

> 如何识别函数调用与返回?
> 调用一般是: `jal ra imm`
> 或者与`lw`等连用, `jalr ra offset(rs1)`, 将pc+4存入ra, pc=offset(rs1), 这样可以跳转更远
> 返回(ret)则是: `jalr zero 0(ra)`, 将pc+4存入zero(没有效果), pc=ra
> 注意到虽然是同一指令, 操作数是不同的



