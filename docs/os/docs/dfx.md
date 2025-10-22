# 调试与故障排除

## 一、重点归纳总结

### 17.1 操作系统调试器

操作系统调试器用于分析和修复内核级错误，但由于内核不以进程形式暴露，调试方法不同于用户态应用。重点包括：

- ​**硬件支持**​：断点分为指令断点（如x86的INT3指令）和内存断点（通过修改页表权限实现）。硬件断点使用调试寄存器（如x86的DR0-DR3），避免软件断点的覆盖问题和性能开销，但数量有限。
    
- ​**模拟器调试**​：在QEMU等模拟器中运行操作系统，利用模拟器的gdbserver支持调试。例如，QEMU/KVM通过插入INT3指令触发VMExit，处理断点异常。
    
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/199963f0373faaf825c7bf5a64308bfe-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063445%3B2076423445&q-key-time=1761063445%3B2076423445&q-header-list=host&q-url-param-list=&q-signature=cb40d5c9ad19f455ee13be4fd5b9e80114b8a203)
    
- ​**内核调试器**​：如Linux的kgdb，通过串口连接远程gdb客户端。触发断点时，kgdb向所有核心发送NMI（不可屏蔽中断）以暂停其他核心，确保调试一致性。
    
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/9620655d19d13b651cbe5de5d9c17921-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063446%3B2076423446&q-key-time=1761063446%3B2076423446&q-header-list=host&q-url-param-list=&q-signature=c88626874b03049ae2c0b175dd54cb81c76d38bb)
    

​**关键点**​：调试内核需硬件辅助（如断点寄存器），模拟器调试方便但无法完全模拟真实硬件，内核调试器（如kgdb）能获取操作系统语义信息（如线程状态）。

---

### 17.2 内核追踪机制

追踪机制用于记录内核运行时事件，分为硬件和软件方法，旨在动态获取信息而不修改内核。

- ​**硬件追踪方法**​：
    
    - ​**LBR（Last Branch Record）​**​：使用寄存器栈记录最近跳转的源和目标地址，开销小但记录有限。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/03f7e3432929f7666677b23b445b1859-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063449%3B2076423449&q-key-time=1761063449%3B2076423449&q-header-list=host&q-url-param-list=&q-signature=c552992add68adcc3f552b240893e0278a0fde91)
        
    - ​**BTS（Branch Trace Store）​**​：将跳转信息写入内存，记录量更大但占用TLB和缓存，性能开销较高。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/0d8a415add1213986a9f2f0ad40f6085-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063450%3B2076423450&q-key-time=1761063450%3B2076423450&q-header-list=host&q-url-param-list=&q-signature=2288e274bfe6d2fd81dab2b4524623d67c517139)
        
    - ​**IPT（Intel Processor Tracing）​**​：改进BTS，使用物理地址存储包格式信息，减少开销，支持更丰富事件（如页表切换）。
        
    
- ​**软件追踪方法**​：
    
    - ​**Tracepoint**​：静态插桩，在代码中预定义追踪点（如记录kmalloc参数）。运行时通过分支判断控制开关，开销低。
        
    - ​**Kprobe**​：动态插桩，通过替换指令为断点指令（如INT3）来注入处理函数，支持任意代码地址，但开销较大。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/4a3a528ff79948f4c5a2b5ccf96e0bf3-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063452%3B2076423452&q-key-time=1761063452%3B2076423452&q-header-list=host&q-url-param-list=&q-signature=ad5d2f6080e63bd7c57436b55d940a826b42ddde)
        
    
- ​**交互机制**​：
    
    - ​**ftrace**​：通过debugfs（如/sys/kernel/debug/tracing）提供统一接口，管理多种tracer（如function tracer、tracepoint）。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/ada0f52630c4d3d22b73fae5a6223f73-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063453%3B2076423453&q-key-time=1761063453%3B2076423453&q-header-list=host&q-url-param-list=&q-signature=6beac9f0ff514fef670512bfe7aab5f5b43ae0a2)
        
    - ​**eBPF**​：允许用户态程序注入安全代码到内核，通过BPF map存储数据，支持动态编译（JIT）。工具如bcc和bpftrace简化使用。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/2538ccfa4ef9b63cf7cee97f8639339d-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063454%3B2076423454&q-key-time=1761063454%3B2076423454&q-header-list=host&q-url-param-list=&q-signature=c810df2ceec1a2b80b9f511bda1f5d3d61cbcb99)
        
    
- ​**内核日志管理**​：printk机制将日志异步写入环形缓存，支持级别控制和多路输出（如串口），但需平衡可靠性与性能。
    

​**关键点**​：硬件追踪（如IPT）适合底层流分析，软件追踪（如tracepoint）语义丰富，ftrace和eBPF提供用户态交互接口。

---

### 17.3 操作系统性能监控

性能监控通过硬件计数器和采样机制评估系统行为，perf工具作为前端简化使用。

- ​**性能监控计数器**​：如x86的IA32_PERFEVTSELx和IA32_PMCx寄存器，统计事件次数（如缓存未命中）。PEBS（Processor Event Based Sampling）在计数器溢出时记录详细状态（如寄存器值），减少中断延迟。
    
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/814c43d6f9b9a0f942536618dccd57de-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063458%3B2076423458&q-key-time=1761063458%3B2076423458&q-header-list=host&q-url-param-list=&q-signature=ad4a490f6f14639381c23921cef2049bae080d43)
    
- ​**perf events**​：将事件抽象为计数统计或profile（采样）。用户通过perf_event_open系统调用配置，共享内存传递数据以减少开销。perf工具支持硬件事件、软件事件（如page-faults）、tracepoint等。
    

​**关键点**​：性能监控依赖硬件寄存器，perf提供统一抽象，共享内存优化数据传输效率。

---

### 17.4 操作系统测试方法

测试确保操作系统正确性、性能和兼容性，涵盖功能测试、压力测试、回归测试等。

- ​**测试流程**​：
    
    - ​**功能测试**​：从小规模单元测试（如Linux的kunit）到集成测试（如LTP），最后系统测试。单元测试在UML模式下运行，隔离硬件影响。
        
    - ​**压力测试**​：榨取资源（如CPU、内存）至极限，验证高并发和边界条件（如LTP的growfile测试文件系统）。
        
    - ​**性能测试**​：通过Phoronix Test Suite等框架定量分析指标（如吞吐量），需控制环境变量和预热。
        
    - ​**兼容性测试**​：验证硬件（如kernelci自动化测试多平台）和软件（如POSIX标准符合性）兼容性。
        
    - ​**持续集成**​：自动化编译、部署和回归测试，确保代码变更后系统稳定。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/634d2900242f8ccffbcb558faa614d1d-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063462%3B2076423462&q-key-time=1761063462%3B2076423462&q-header-list=host&q-url-param-list=&q-signature=18571649a3a15b9c45e7185408ad9ef34c88c6fd)
        
    
- ​**辅助手段**​：
    
    - ​**静态分析**​：如sparse工具检测代码错误（如未释放锁）。
        
    - ​**内存检查**​：KASAN（Kernel Address Sanitizer）通过shadow memory检测内存错误（如越界访问），但增加开销。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/319d82fd304bdd4007ce9dc09e1a96c6-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063464%3B2076423464&q-key-time=1761063464%3B2076423464&q-header-list=host&q-url-param-list=&q-signature=a9a194a89704f813cd315d8534317ed63e9f97dd)
        
    - ​**代码覆盖检测**​：gcov统计基本块执行次数，kcov优化模糊测试中的覆盖收集。
        
    
- ​**其他方法**​：
    
    - ​**模糊测试**​：如syzkaller自动生成系统调用输入，通过语料库变异提高覆盖。
        
        ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/e35ba16583b5c4ceac579496c531308c-image.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1761063465%3B2076423465&q-key-time=1761063465%3B2076423465&q-header-list=host&q-url-param-list=&q-signature=251c2a2c063848a39b1aa10b5a569f088344af31)
        
    - ​**安全测试**​：审计配置（如OpenSCAP）和渗透测试（如Metasploit）验证安全性。
        
    

​**关键点**​：测试需覆盖多维度，辅助工具（如KASAN）提升错误检测能力，持续集成保障迭代质量。

---

### 17.5 案例分析：ChCore内核测试流程

ChCore采用持续集成，代码提交触发自动化测试：

- ​**轻量级测试**​：静态检查（fbinfer）、单元测试（模块隔离）、集成测试（启动验证）。
    
- ​**合并前测试**​：完整内核环境下的单元测试、集成测试（用户态应用）、性能测试（IPC延迟）、兼容性测试（x86/AARCH64）。
    
- ​**新功能测试**​：及时添加对应测试，确保回归覆盖。
    

​**关键点**​：测试驱动开发，自动化流程降低人工负担。

---

## 二、思考题详细解答

以下针对文档17.6节的8个思考题，提供详细答案基于文档内容推理。

### 1. perf采样生成函数调用图时，默认采取backtrace方式，但优化程序性能时可能消除栈指针操作（如gcc的-fomit-frame-pointer），此时无法通过backtrace生成调用关系。你认为本章节所述哪些硬件机制能够帮助你重构函数调用关系？

​**答案**​：

当backtrace失效时，可利用硬件追踪机制重构调用关系：

- ​**LBR（Last Branch Record）​**​：记录最近跳转的源和目标地址，能直接反映函数调用序列。但LBR深度有限（通常几十条记录），仅适合短路径重构。
    
- ​**BTS（Branch Trace Store）​**​：将跳转信息持续写入内存，覆盖更长执行流，可解析出完整调用图。但BTS有性能开销，可能影响实时性。
    
- ​**IPT（Intel Processor Tracing）​**​：记录更丰富的包信息（如调用/返回事件），结合符号表可精确重构调用关系。IPT绕过缓存，开销低于BTS。
    
- ​**perf的call-graph功能**​：perf工具支持使用LBR或IPT作为回调机制（如`perf record --call-graph lbr`），自动生成调用图。
    

综上，IPT是最佳选择，因它平衡了开销和信息完整性。

---

### 2. 在操作系统中使用断点调试时，有时会遇到动态装载的代码，即在设置断点地址后代码才被写入特定位置。你认为这种情况应该使用硬件断点还是软件断点，为什么？

​**答案**​：

应优先使用**硬件断点**，原因如下：

- ​**软件断点问题**​：通过修改指令为断点指令（如INT3）实现，但动态装载的代码可能覆盖已设置的软件断点，导致断点失效或误触发。
    
- ​**硬件断点优势**​：使用调试寄存器（如DR0-DR3）监控地址，无需修改代码内存，因此不受代码动态装载影响。硬件断点在代码写入后仍能正常触发。
    
- ​**局限性**​：硬件断点数量有限（x86仅4个），需谨慎分配。若断点数量超限，可结合软件断点作为补充，但需确保代码装载完成后再设置。
    

结论：在动态代码场景下，硬件断点更可靠，但需注意寄存器资源限制。

---

### 3. perf events为何使用共享内存传递采样信息给用户态应用？相比系统调用的接口有何优势？

​**答案**​：

perf events使用共享内存的主要优势是**降低开销和提高实时性**​：

- ​**系统调用开销**​：若每次采样都通过系统调用（如read）传递数据，会导致频繁内核态-用户态切换，增加CPU负载和延迟。
    
- ​**共享内存优势**​：
    
    - ​**零拷贝**​：内核直接将采样数据写入环形缓存，用户态应用通过mmap映射访问，避免数据复制。
        
    - ​**异步处理**​：用户态可批量读取数据，减少上下文切换次数，适合高频率采样（如每秒数千次事件）。
        
    - ​**实时性**​：共享内存允许用户态实时监控数据，而系统调用有调度延迟。
        
    
- ​**用例**​：在perf profile中，采样数据量较大，共享内存能有效支撑高性能监控。
    

相比系统调用，共享内存更适合大量、频繁的数据传递场景。

---

### 4. 除了本章所述的通过系统调用、共享内存等方式获取内核信息外，Linux中还有通过插入内核模块的方式直接获取内核信息的方法（如SystemTap、sysdig）。该方法相比与前者的优点和缺点分别是什么？

​**答案**​：

​**内核模块方式（如SystemTap）​**​：

- ​**优点**​：
    
    - ​**灵活性高**​：可直接访问内核数据结构，获取深层信息（如内部队列状态），不受ftrace或eBPF限制。
        
    - ​**性能可控**​：模块可定制数据收集逻辑，避免通用框架的开销。
        
    - ​**功能强大**​：支持动态插桩和复杂过滤，适合定制化监控。
        
    
- ​**缺点**​：
    
    - ​**安全风险**​：模块运行在内核态，错误可能导致系统崩溃或安全漏洞。
        
    - ​**开发复杂**​：需编写、编译和加载模块，维护成本高，且需处理内核版本兼容性。
        
    - ​**调试困难**​：模块错误不易追踪，可能干扰系统稳定性。
        
    

​**对比ftrace/eBPF**​：

- ftrace/eBPF通过验证机制保障安全，但功能受限；内核模块更强大但风险高。实际中，优先使用eBPF等安全框架，仅复杂场景用模块。
    

---

### 5. 如果为了验证I/O功能正确性，需要对其进行压力测试时，有必要增加其它资源（如处理器、内存）的负载吗？

​**答案**​：

​**有必要**，原因基于测试的真实性和全面性：

- ​**模拟真实场景**​：实际应用中，I/O操作常与CPU密集型任务（如数据压缩）或内存分配（如缓存）并发进行。单独测试I/O可能无法暴露资源竞争引发的错误（如死锁或性能抖动）。
    
- ​**暴露隐藏问题**​：增加CPU/内存负载可触发边界条件，例如：
    
    - 高CPU负载可能导致I/O调度器行为变化。
        
    - 内存压力可能影响I/O缓存效率，引发缺页异常。
        
    
- ​**测试原则**​：压力测试应覆盖多资源交互，如LTP测试中组合文件系统操作与内存分配。但需根据目标调整负载比例，避免过度泛化。
    

结论：综合负载测试能更全面验证I/O功能正确性。

---

### 6. 你认为在正确性测试中保持KASAN开启是否能让正确性测试更加可靠？

​**答案**​：

​**是，但需权衡开销**​：

- ​**KASAN的作用**​：检测内存错误（如越界访问、使用释放内存），这些错误在常规测试中可能隐藏（如越界访问未立即崩溃）。开启KASAN能提前暴露问题，提升测试覆盖率。
    
- ​**可靠性提升**​：在测试阶段（尤其是开发期），KASAN能发现静态分析遗漏的动态错误，增强正确性保证。
    
- ​**开销问题**​：KASAN增加内存和性能开销（约2-3倍慢down），可能影响测试效率或掩盖性能问题。因此：
    
    - ​**正确性测试中**​：建议开启KASAN，以最大化错误检测。
        
    - ​**性能测试中**​：应关闭KASAN，避免开销干扰结果。
        
    
- ​**最佳实践**​：如ChCore流程，在持续集成中开启KASAN进行单元测试，但部署时关闭。
    

结论：KASAN提升正确性测试可靠性，但需区分测试类型。

---

### 7. 在fuzzing测试中，syzkaller通过构造系统调用向操作系统产生不同的输入，你认为这样的输入方式合理吗？有没有其它为操作系统产生输入的方法？

​**答案**​：

​**系统调用输入合理，但可扩展其他方法**​：

- ​**合理性**​：
    
    - 系统调用是用户态与内核交互的主要接口，覆盖大部分内核功能（如文件、网络、进程管理）。
        
    - syzkaller通过语料库变异系统调用参数，能有效触发深层代码路径，实践已发现大量内核漏洞。
        
    
- ​**局限性**​：仅通过系统调用无法测试所有场景，如：
    
    - 硬件中断或异常处理。
        
    - 内核内部函数（不暴露为系统调用）。
        
    - 并发竞争条件（需更精细的时序控制）。
        
    
- ​**其他输入方法**​：
    
    - ​**网络包注入**​：模拟网络栈输入，测试协议处理。
        
    - ​**设备I/O模拟**​：通过虚拟设备驱动触发中断（如QEMU的故障注入）。
        
    - ​**内存直接操作**​：修改内核数据结构（如通过/proc或sysfs），但需谨慎避免崩溃。
        
    - ​**混合输入**​：结合系统调用、定时器和外部事件，提高覆盖度。
        
    

结论：系统调用是核心方法，但补充其他输入能增强fuzzing效果。

---

### 8. 本章哪些调测手段可用于在Linux中追踪系统调用？

​**答案**​：

多种手段可追踪系统调用，按开销和功能排序：

1. ​**tracepoint**​：静态插桩点（如`sys_enter`/`sys_exit`），通过ftrace或perf使用（如`perf trace`），开销低，适合生产环境。
    
2. ​**kprobe**​：动态追踪任意系统调用函数（如`__x64_sys_open`），功能灵活但开销较大。
    
3. ​**eBPF**​：编写BPF程序挂载到系统调用tracepoint，安全高效（如bcc的`trace`工具）。
    
4. ​**perf events**​：配置系统调用事件（如`syscalls:sys_enter_open`），进行计数或采样。
    
5. ​**strace**​：基于ptrace的用户态工具，简单但开销高，不适合高频追踪。
    
6. ​**内核日志**​：printk打印系统调用信息，但需修改代码，主要用于调试。
    

​**推荐**​：日常使用优先选择tracepoint或eBPF，平衡性能和功能。

---

## 三、总结

本文档系统介绍了操作系统调测的全流程，从调试器、追踪机制到性能监控和测试方法，强调了硬件支持（如断点寄存器、性能计数器）和软件工具（如perf、ftrace）的结合。思考题解答基于文档原理，突出了实践中的权衡（如硬件/软件断点选择）和最佳实践（如KASAN在测试中的使用）。调测手段需根据场景灵活选择，以平衡性能、可靠性和开发效率。

---

以上内容基于文档《第十七章 操作系统调测》整理，如有细节需求可进一步参考原文。