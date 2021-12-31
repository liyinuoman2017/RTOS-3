# RTOS-3
嵌入式实时操作系统3——任务切换
**任务切换原理**
假设有一程序，程序内有一个无限循环，在循环内部有5个表达式，代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5a49ebed29404da6972823feea0f6253.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

程序在循环中，会依次执行表达式1-》表达式2-》表达式3-》表达式4-》表达式5-》表达式1无限循环。假设没有使用静态变量，没有使用堆空间，没有中断程序。程序每执行一个表达式后的处理器状态如下（**每一种颜色代表一种状态，颜色变化说明状态变化**）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e28f73dd56ec48efb6cf505bda773302.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

程序循环周期执行，处理器的状态也循环周期变化。
我们将**指令存储器，数据存储器，寄存器堆的所有数据称为：处理器总值**。上图中处理器总值为P1,P2,P3,P4,P5 。

假设现在有一种“神奇的力量”将处理总值改变成P3，处理器将如何运行？
处理器将会按照P3,P4,P5，P1,P2,P3的顺序运行，运行图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e1fe971e88b14a44b9928533d6d6d67f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

对应的程序运行状态：表达式3-》表达式4-》表达式5-》表达式1-》表达式2-》表达式3无限循环。
我们可以得出一个结论：**在不考虑外设，中断等因素，给处理器一个合理总值Pn,处理器的下一个总值必然为Pn+1 。**

利用这个原理，**给处理器任意一个合理总值Pn，使得程序从任意一个合理位置开始运行。**（关于总值切换详细说明请参考《RTOS系列2——任务调度子系统1》）

我们设计一种模式：每个任务有独立的代码区，有独立的栈区。多任务系统处理器状态如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/8404631129af40399e790ba505b02aec.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

每个任务的代码区（代码区可多任务共享），独立任务栈区，静态区（可以共享）。将每个任务的**代码区，独立任务栈区，静态区，独立寄存器堆所有数据称为相对总值**。通过保存和载入相对总值实现任务切换。单个任务处理器状态如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/0ee6ddc2bebe47e791c086189384a48e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

由于代码区数据不会改变，独立任务栈区不会被其他任务操作，任务栈区相对任务不会变，寄存器堆会变化，因此在这种情况下**保存相对总值，等效为保存寄存器堆数据**。

如何让任务有独立任务栈区，在静态区创建一个静态数组task1_stack[SIZE],任务使用这一个静态数组作为**独立任务栈区**，栈指针指向这个静态数组，栈的大小为SIZE。**栈指针保存在静态区，作为一个全局变量。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/7f5c4b60164e47be9373acb642f0962e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

如何保存任务的寄存器堆数据，当任务需要暂停并切换其他任务时，**将寄存器堆的值保存到任务独有栈区**，当任务需要恢复运行时，从任务独有的堆栈区恢复CPU寄存器的值。这样就实现了任务“独享”CPU寄存器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/398df9ef3a914c1ab68c97d8b713ebad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

代码区不会变化，并且可以被多个任务共享。同一个任务代码可以创建多个任务。静态区内的数据可以共享，也可以私有。

在这种设计模式下，**通过保存寄存器堆数据值和装载寄存器堆数据实现任务切换**。保存寄存器堆数据称为**保存现场** ，载入寄存器堆数据称为**恢复现场**。保存现场和恢复现场称为**上下文切换**。

**任务切换源码分析**
上文我们了解了任务切换的原理，接下来我们通过分析一下任务切换的代码（参数CM3平台下FREERTOS源码）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/43688e53394446c6bf9f3e0dc1e632cd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)
**代码分析：**

![在这里插入图片描述](https://img-blog.csdnimg.cn/c5e2bf327d24449991e6c6072cdb3090.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

将寄存器堆的值保存到任务的栈空间，并将任务的栈指针保存到任务的TCB结构中，任务的TCB结构通常为一个全局结构变量，TCB的第一个变量通常用于存放栈指针。pxCurrentTCB为一个全局TCB指针变量。

![在这里插入图片描述](https://img-blog.csdnimg.cn/691a317b70f44b6c9ae67a2fbf832ed2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

将找出最高优先级任务，并将pxCurrentTCB指向最高优先级任务TCB。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7269c89f73404145bc3e001ab6fca59e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

取出最高优先级的栈指针，利用栈指针恢复寄存器堆数据实现任务切换。

> **任务切换5个步骤：
1、保存现场 ，将寄存器堆的值保存到任务的栈空间。
2、保存栈指针，将栈指针保存到当前任务的TCB结构中。
3、找出最高优先级任务，并修改当前任务对象。
4、读取栈指针，当前任务的TCB结构中读取栈指针。
5、恢复现场，从任务的栈空间恢复寄存器堆的值。**



**任务切换实现**
代码如何实现任务切换，分析代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/69de9a2290884fa38791567ce3db91ad.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)

示例中使用“语句”代替实现该功能的代码。

示例代码的意图是在任务完成函数处理后，直接使用语句实现保存现场 ，保存栈指针，找出最高优先级任务，取出栈指针，恢复现场。“企图”使用这种方式实现任务切换，这种方式可行吗？

假设我们的设计目的是task1运行，task1执行完task1_handle函数后将自己设置为等待，将task2设置为就绪，然后切换到task2运行，task2执行完task2_handle函数后将自己设置为等待，将task1设置为就绪，然后切换到task1运行，循环运行。根据这种设计逻辑我们来分析一下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e644a3fcfd584f368778b4b42294a9f0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

虽然这种方式无法实现任务切换 ，但是我们得出如下结论：


> **1、调用任务切换语句是单一语句。
2、保存现场保存的PC值是调用任务切换语句的下一条语句。
3、任务切换功能需要在另外一处实现。**

根据以上结论，可以通过两种方式实现切换功能：函数调用，软中断。
这两种方式的特点是：

> **1、执行函数调用或者中断函数是单一语句。
2、执行函数调用或者中断函数前，处理器会自动保存下一条用户指令的PC值。
3、切换代码可以在被调用的函数或中断函数中实现
4、函数返回或者中断返回，会载入切换点的下一条语句。**

**通过函数调用实现切换示例。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/9acaf7e1ce894e2583b3e46fa603a40d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)

保存点如上图所示，当任务被恢复时，从被保存点开始执行。**切换代码在被调用的函数中实现。**

**通过软中断实现切换示例。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/f764a2cc1a734d3c994f6f3798e1a651.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)

保存点如上图所示，当任务被恢复时，从被保存点开始执行。**切换代码在中断函数中实现。**

前文分析的__asm void xPortPendSVHandler( void ) 函数是通过软中断机制实现。通常情况下首选软中断方式，当处理器硬件不支持软中断时才使用函数调用的方式。软中断方式会触发一个中断，硬件会自动入栈保存部分寄存器，上下文切换效率较高。

**任务切换等级**
任务切换通常分为两个等级：**任务级切换**和**中断级切换**。

任务级切换指的是任务再运行过程中调用了操作系统切换任务接口函数实现任务切换。任务级切换需要实现保存现场 ，保存栈指针，找出最高优先级任务，取出栈指针，恢复现场。

中断级切换指的是任务运行时发生了一个中断，中断函数中调用了一些操作系统接口函数，导致高优先级任务就绪，在中断返回时需要进行任务切换。中断级切换只用实现找出最高优先级任务，取出栈指针，恢复现场。因为在进入中断是已经完成了保存现场 ，保存栈指针的工作。

**任务切换点**
前文中说明了任务切换原理，现在还有一个问题：任务切换发生在哪里？

假设现在有两个任务：高优先级任务task1和低优先级任务task2 。task1执行时需要等待信号S，等待信号时task1休眠，task2运行并发送信号S,此时task1被唤醒并执行。运行图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/0903aca591e846b585bd98de1fc62ee6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

分析：在执行等待信号函数后task1任务休眠，此时发生一次任务切换；在发送信号函数后task1运行，此时又发生一次任务切换。正是因为这种机制使得条件准备就绪后（发送信号）重要的任务可以立即执行（task1被唤醒并执行），实现了实时操作系统的核心价值——实时性。

任务切换不仅仅存在于信号发生和等待，任务切换发生在如下情况：

> **1、等待信号，发生信号
2、创建任务，删除任务
3、挂起任务，恢复任务
4、调用延时，取消延时
5、中断程序返回
6、任务主动调用调度函数**



**TCB设计**
上文中提到需要把任务的栈指针保存到静态区，对于一个任务有许多信息需要保存，因此我们需要在静态区创建一个任务的信息对象，这个对象为 task control block 简写为 TCB。 我们开始为任务任务定义一个简单的TCB结构：

![在这里插入图片描述](https://img-blog.csdnimg.cn/4b6b0056e5fb4aca9f90e5257a60f1f5.png)

定义的TCB中包含了栈指针，优先级。本系列会根据提到的知识点逐步完善任务的TCB结构。TCB结构于寄存器堆，任务栈的关系图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/f30b37ecde324d73be0967527779edb0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_14,color_FFFFFF,t_70,g_se,x_16)

静态区定义的任务TCB对象中的栈指针指向任务栈区的栈顶，任务栈区保存着寄存器堆的数据，寄存器堆中的PC值指向任务的代码区。


> <font color=red>**未完待续…
实时操作系统系列将持续更新
创作不易希望朋友们点赞，转发，评论，关注。
您的点赞，转发，评论，关注将是我持续更新的动力
作者：李巍
Github：liyinuoman2017**
