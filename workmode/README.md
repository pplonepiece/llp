1. 工作模式
- 实模式（16位）
- 保护模式（32位）
- 虚拟模式（在保护模式下的实模式）
- 系统管理模式（在睡眠状态及电源管理时提供的模式）
- 长模式（64位）

2. 实模式
- 最古老的模式，没有虚拟内存的概念，操作系统直接在物理内存上操作，寄存器的长度也只有16位
- 实模式下使用的寄存器
	- ip, flags
	- ax, bx, cx, dx, sp, bp, si, di
	- 段寄存器：cs, ds, ss, es (gs, fs)
		- 寄存器只有65535字节的寻址能力，这些内存区域被称之为段
- 为了访问更大的地址空间，需要借助段寄存器
	- 物理内存地址是20位
	- 逻辑地址由两部分组成：段寄存器 * 16 + 偏移量
		- 段寄存器表示段的起始位置
		- 偏移量表示在段内的偏移位置 
	- 段寄存器 
		- cs: 存储代码段起始地址
		- ds: 存储数据段起始地址
		- ss: 存储栈段的起始地址
	- 程序被加载时，会自动设置ip, cs, ss, sp，这样cs:ip就指向了程序的入口，而ss:ip则指向了栈的顶部
- CPU总是从实模式开始运行，然后主加载器执行代码切换至保护模式，最后再切到长模式
- 实模式的缺点
	- 多任务非常困难：因为所有程序共享内存地址，所以不同程序需要被加载到不同内存地址，但是他们的相对排列需要在编译器指定
	- 不安全：因为共享内存地址，程序员可以更改系统代码
	- 还是不安全：程序员可以执行任意指令

3. 保护模式
- 首次出现在80386的32位保护模式处理器上
- 重大更新
	- 32位寄存器：eax, ebx, ..., esi, edi
	- protection ring
	- 虚拟内存
	- 改善分段
		- 当应用程序异常结束时，不会对操作系统有任何影响，更没有方法破坏处理器的内存
		- 获取段的起始地址的方式发生了变化，不再是段寄存器*16，而是根据一个特别的表条目计算得到
			- 线性地址 = 段基址 \(从系统表中读取\) + 偏移量
		- 段寄存器有cs, ds, ss, es, gs, fs，其中存储的内容叫做段选择器，是一个指向特殊描述符表的索引及一些附加信息
		- 描述符表有LDT和GDT(Global Descriptor Tabel)，LDT基本没有使用
			- GDTR是一个寄存器，存储了GDT的地址和大小
				- 15 ........... 3 2 1 0
				- |     Index    | T RPL
					- Index表示描述符在GDT中的位置
					- T表示LDT还是GDT（都是0，即GDT）
					- RPL特权级别：通过段选择器来访问一个段时，会进行请求特权级别和描述符特权级别的检查，如果特权级别不够，则会安生错误
				- GDT的详细内容参见：https://cch123.gitbooks.io/duplicate/content/part1/legacy/protected-mode.html
- 处理器总是从实模式启动，为了进入保护模式，需要创建GDT并设置好GDTR。在cr0设置好一个特殊的标识，并执行far jump
	- far jump：表示段（或段选择器）已经显示给出
		- 如： jump 0x08:addr
- shadow 寄存器
	- 为了防止内存的每次事物都去读取GDT，每个段寄存器都有了一个shadow寄存器的东西，该寄存器中的内容就是GDT的缓存，但是注意不能直接访问
	- 当一个段寄存器被修改了，对应的shadow寄存器就载入GDT中对应的描述符，之后所有需要这个段信息就可以直接将shadow寄存器当做数据源
- 分段式个麻烦事
	- 对程序员来说，不分段更为简单
	- 编程语言都没有分段的概念
	- 分段会造成内存碎片
	- 描述符表最多只有8192个，太少啦
- 引入长模式之后，除了在protection ring，分段已经消失了，也就是说程序员不需要在了解这些了。

4. 长模式
- 长模式下，还是会使用分段技术，系统提供给我们的是一维线性虚拟地址，然后再将虚拟地址转换为物理地址
- 长模式下的区别在于：通过主要的段寄存器(cs, ds, ss, es)来寻址都不在考虑GDT中的base和offset了；不论descriptor内容是什么，段的起始地址都是从0x0开始，段的大小没有限制；而descriptor的其他字段不会被忽略
- 因此，长模式下的GDT中至少还需要三个描述符：null descriptor、代码段，以及数据段

5. 寄存器访问
- 对于64位寄存器，写8位或者16位可以让集群器的其余部分保持不变；但是在写32位时，系统会把64位寄存器的高位全部填上标识符
	- 示例：
		- mov rax, 0x1122334455667788	; rax = 0x11 22 33 44 55 66 77 88
		- mov eax, 0x42					; rax = 0x00 00 00 00 00 00 00 42
		- mov rax, 0x1122334455667788
		- mov ax, 0x9999				; rax = 0x11 22 33 44 55 66 99 99
- 这个问题的成因在于系统指令集的优化
	- 处理器一般有两个指令集
		- CISC (Complete Instruction Set Computer)
		- RISC (Reduced Instruction Set Computer)
		- 对比：CISC指令一般运行比较慢，有时候CISC指令比组合许多RISC指令来实现一个复杂功能更为简便；但是高级语言基本都是依赖编译器，而实现CISC指令集的编译器非常的困难
	- 对于指令的解码，CPU的指令解码器会将复杂的CISC指令翻译成RISC指令；再结合流水线技术，通知执行多条指令，能够极大的提升系统处理速度
	- 为了实现上面这个目的，寄存器成了一个虚拟的概念，当微指令执行的时候，需要从寄存机集合中选择一个可用的寄存器；而如果微指令的之间数据相互依赖，就没办法同时执行
	- 为了消除上面的问题，对于64位操作系统来说，如果访问eax寄存器，通过忽略rax的高32位就可以消除这种依赖，从而实现微指令的流水线执行
	- 为什么8位和16位的操作就没有这个问题呢？这是为了兼容老系统应用程序！因为这种寄存器的行为是在64位操作系统上才引入的
