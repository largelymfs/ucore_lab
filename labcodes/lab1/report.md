###操作系统第一次实验报告

####第一题

* #####1.1:操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中 每一条相关命令和命令参数的含义,以及说明命令导致的结果)?

	首先我们对于题目生成`ucore.img`的过程进行翻译，生成`ucore.img`的语句是：
	
	```
	$(UCOREIMG): $(kernel) $(bootblock)
   	$(V)dd if=/dev/zero of=$@ count=10000
   	$(V)dd if=$(bootblock) of=$@ conv=notrunc
   	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
	```
	
	实际执行的语句是
	
	```
	sdd if=/dev/zero of=bin/ucore.img count=10000	dd if=bin/bootblock of=bin/ucore.img conv=notrunc	dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
	```
	
	因此我们发现我们需要进行`kernel`和`bootblock`两个部分的编译。编译之后执行下面几个步骤：
	1. 写入10000个512字节的0
	2. 将`bootblock`写入一个块
	3. 将`kernel`相邻下写入内存中，完成`ucore.img`的生成
	
	
	关于`kernel`的生成过程对应的代码是：
	
	```
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
	```
	
	实际执行的代码是
	
	```
	ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
	```
	这句话对所有需要的`.o`进行链接。
	
	这句话使用的参数是：
	* -m \<emulation\>  模拟为i386上的连接器
	* -nostdlib  不使用标准库
	* -T \<scriptfile\>  让连接器使用指定的脚本
	* -o <output_filename> 输出结果
	
	这句话所需要的前提是`kernel.ld init.o readline.o stdio.o kdebug.o
kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
trapentry.o vectors.o pmm.o  printfmt.o string.o`，其中`kernel.ld`是已知的。
	
	生成`*.o`的语句是（以`init.o`为例）
	
	```
	$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
	```
	实际执行的语句是
	
	```
gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
-gstabs -nostdinc  -fno-stack-protector \
-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
-o obj/kern/init/init.o
	```
	这句话对`.c`文件进行了编译，生成`.o`文件。
	
	其中使用的编译参数是：
	* -I\<dir\> 添加搜索头文件的路径
	* -fno-builtin 除非用__builtin_前缀，否则不进行builtin函数的优化
	* -Wall 显示警告信息
	* -ggdb 生成可供gdb使用的调试信息。为了方便进行gdb调试。
	* -m32 生成适用于32位环境的代码。
	* -gstabs 生成stabs格式的调试信息，也就是生成调试信息。
	* -nostdinc 不使用标准库 
	* -fno-stack-protector 不生成用于检测缓冲区溢出的代码
	* -c \<filename\> 		源文件
	* -o \<output_filename\> 输出文件
	
	关于`bootblock`的生成代码：
	
	```
	$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	```
	具体的执行流程是：
	
	1. `ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o`对`bootasm.o`和`bootmain.o`进行链接，使用的参数是
		* `-m <emulation>`连接模拟器
		* `-N` 代码段和数据段都可以写
		* `-e` 指定入口
		* `-TAddr`制定程序的入口
	
	2.`objcopy -S -O binary obj/bootblock.o obj/bootblock.out`将`.o`拷贝到`.out`之后，使用的参数是
		* `-S`   移除所有符号和重定位信息
		* `-O <bfdname>`  指定输出格式
		
	3.`bin/sign obj/bootblock.out bin/bootblock` 使用sign工具处理bootblock.out，生成bootblock
	
	这句话所需要的是编译`bootasm.o`、`bootmain.o`、`sign`。
	
	编译`bootasm.o`的过程，所需要的代码是
	
	```
	bootfiles = $(call listf_cc,boot) 
	$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
	$(CFLAGS) -Os -nostdinc))
	```
	对应的实际指令时：
	
	```
	gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
	-c boot/bootasm.S -o obj/boot/bootasm.o
	```
	编译生成`bootasm.o`，同理生成了`bootmain.o`。
	
	编译生成`sign`的过程， 使用的内容是：
	
	```
	$(call add_files_host,tools/sign.c,sign,sign)
	$(call create_target_host,sign,sign)
	```
	对应的实际指令是
	
	```
	gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
	gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
	```
	这样编译出`sign`工具。
	
	其中用到的新参数是:
	* `-g` 输出调试信息
	
	以上就是编译生成`ucore.img`的全过程。

* #####1.2:一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
我们观察`sign.c`我们观察`buf`数组。对应的代码是：

```
buf[510] = 0x55;
buf[511] = 0xAA;
```

发现合法的硬盘主引导扇区的特征是:

* 只有512个字节（0~511）
* 第510号字节是`0x55`
* 第511号字节是`0xAA`

####第二题：
* #####2.1:从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。
