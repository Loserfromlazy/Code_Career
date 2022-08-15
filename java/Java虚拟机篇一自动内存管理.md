# Java虚拟机篇一自动内存管理

> 参考资料：深入理解Java虚拟机周志明第三版以及互联网资源具体引用会在文中标注

# 一、Java内存区域

对于Java程序员来说，在虚拟机自动内存管理机制下，不再需要为每一个new操作配对delete/free代码，但是正由于Java将控制内存的权力交给虚拟机，一旦出现内存泄漏或溢出等问题，如果不了解虚拟机对内存的使用就会使排查问题变得艰难。

## 1.1 运行时数据区域

Java虚拟机在执行Java程序过程中会把它所管理的内存划分为不同的数据区域这些区域有各自的用途、建立和销毁的事件，有的区域会随着虚拟机的进程而一直存在，有的区域则是随着用户线程的启动和结束而建立和销毁，根据Java虚拟机规范，Java虚拟机内存管理包含以下几个运行时数据区域。

![image-20220803095308621](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803095308621.png)

### 程序计数器

程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器，在Java虚拟机的概念模型中，字节码解释器工作时就是通过改变这个计数器来选取下一条需要执行的字节码指令，它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成。

由于Java虚拟机的多线程是通过线程轮流切换、分配处理器执行时间实现的，在任何一个时刻，一个处理器（多核的话是一个内核）都会只执行一条线程中的指令。因此，为了线程切换后能恢复到正常的执行位置，每条线程都有一个独立的程序计数器，各线程之间互不影响独立存储，这类内存区域称为线程私有内存。

如果线程正在执行一个Java方法，这个计数器记录的是正在执行的虚拟机字节码的地址，如果正在执行本地方法，这个计数器的值为空。**此内存区域是唯一一个没有规定OOM的区域。**

### Java虚拟机栈

Java虚拟机栈也是线程私有，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的线程内存模型，每个方法执行的时候，虚拟机栈会同步创建一个栈帧，用于存储局部变量表、动态链接、方法出口等信息，每一个方法都有执行完毕的过程，对应着一个栈帧在虚拟机栈中从入站到出站的过程。

![image-20220810141054675](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810141054675.png)

> 总有人将Java内存笼统的分为堆内存和栈内存，栈通常指的就是虚拟机栈或者更多情况下指的是虚拟机栈中的局部变量表。

虚拟机栈中的局部变量表存放了编译期可知的各种Java虚拟机基本数据类型、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始 地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress 类型（指向了一条字节码指令的地址）。这些数据类型在局部变量表中的存储空间以局部变量槽（Slot）来表示，其中64位长度的long和 double类型的数据会占用两个变量槽，其余的数据类型只占用一个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。请读者注意，这里说的“大小”是指变量槽的数量， 虚拟机真正使用多大的内存空间来实现一 个变量槽，这是完全由具体的虚拟机实现自行决定的事情。

在《Java虚拟机规范》中，对这个内存区域规定了两类异常状况：**如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。**

如果我们想设置虚拟机栈的大小，可以通过-Xss进行设置，比如`-Xss1m`，`-Xss1024k`

### 本地方法栈

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别只是虚拟机 栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native） 方法服务。

Java虚拟机规范对本地方法栈中方法使用的语言、使用方式与数据结构并没有任何强制规定，因此具体的虚拟机可以根据需要自由实现它，甚至有的Java虚拟机（譬如Hot-Spot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。**与虚拟机栈一样，本地方法栈也会在栈深度溢出或者栈扩展失败时分别抛出StackOverflowError和OutOfMemoryError异常。**

### Java堆

对于Java应用程序来说，Java堆Java Heap是虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，Java世界里所有的对象实例都在这里分配内存。Java堆是垃圾收集器管理的主要内存区域，因此一些资料中它也被称作“GC堆”。从内存回收的角度看，大部分收集器都采用分代收集算法，所以Java堆还分为新生代和老年代，新生代包括：Eden区、SurvivorFrom、SurvivorTo。

如果从分配内存的角度看，所有线程共享的Java堆中可以划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB），以提升对象分配时的效率。不过无论从什么角度，无论如何划分，都不会改变Java堆中存储内容的共性，无论是哪个区域，存储的都只能是对象的实例，将Java堆细分的目的只是为了更好地回收内存，或者更快地分配内存。

Java堆可以处于物理上不连续的内存空间中，但在逻辑上它应该 被视为连续的，这点就像我们用磁盘空间去存储文件一样，并不要求每个文件都连续存放。但对于大对象（典型的如数组对象），多数虚拟机实现出于实现简单、存储高效的考虑，很可能会要求连续的内存空间。Java堆既可以被实现成固定大小的，也可以是可扩展的，不过当前主流的Java虚拟机都是按照可扩展来实现的（通过参数-Xmx和-Xms设定）。**如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，Java虚拟机将会抛出OutOfMemoryError异常。**

因为现代垃圾收集器大部分采用分代理论（这部分可以在下一章查看），因此Java堆存储也可以分为两个部分青年代和老年代（在jdk7中Java堆内存还多了一个永久代，因为Java8以后方法区不分配在Java堆上了，而是存储在本地存储元空间中，所以永久代不存在了）：

![image-20220810145042764](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810145042764.png)

从上图我们也能看到Java堆被分为两部分，年轻代主要存放新创建的对象，内存大小相对较小，垃圾回收很频繁，年轻代分为Eden Space和2个Survivor Space，而年老代主要存放JVM认为生命周期较长的对象，内存大小相对比较大，垃圾回收也相对没有那么频繁。

默认情况下新生代占整个堆的1/3，老年代占2/3参数为`-XX:NewRatio=2`。我们可以通过这个参数进行修改，比如设置了`-XX:NewRatio=4`,那么老年代就占整个堆的4/5。

Eden区和两个Survivor区的占比是8:1:1,可以通过`-XX:SurvivorRatio`进行调整计算方式为：

Eden = (R*Y)/(R+1+1)
From = Y/(R+1+1)
To   = Y/(R+1+1)

其中Y为新生代的大小，R为`-XX:SurvivorRatio`的值。JVM每次只会使用一块Survivor和Eden区，因此总会有一块S区是空闲的。

> Survivor区主要是减少被送往老年代的对象，设置两块是为了使用复制算法时，解决了碎片化内存空间。

下图是对象分配的过程

![image-20220810152053231](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810152053231.png)

触发Minor GC时首先将Eden区剩余对象移入S0区域，如果再次垃圾回收就移入S1区域，Survivor的两个区域交替使用，如果累计次数达到15次，就会将此对象移入老年代。

### 方法区

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。虽然Java虚拟机规范中把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫作“非堆”（Non-Heap），目的是与Java堆区分开来。

> 说到方法区，不得不提一下“永久代”这个概念，尤其是在JDK 8以前，许多Java程序员都习惯在HotSpot虚拟机上开发、部署程序，很多人都更愿意把方法区称呼为“永久代”），或将两者混为一谈。本质上这两者并不是等价的，因为仅仅是当时的HotSpot虚拟机设计团队选择把收集器的分代设计扩展至方法区，或者说使用永久代来实现方法区而已，这样使得HotSpot的垃圾收集器能够像管理Java堆一样管理这部分内存，省去专门为方法区编写内存管理代码的工作。但是对于其他虚拟机实现，譬如BEA JRockit、IBM J9等来说，是不存在永久代的概念的。
>
> 考虑到HotSpot未来的发展，在JDK 6的时候HotSpot开发团队就有放弃永久代，逐步改为采用本地内存（Native Memory）来实现方法区的计划了，到了JDK 7的HotSpot，已经把原本放在永久代的字符串常量池、静态变量等移出，而到了JDK 8，终于完全废弃了永久代的概念，改用与JRockit、J9一样在本地内存中实现的元空间（Meta-space）来代替，把JDK 7中永久代还剩余的内容（主要是类型信息）全部移到元空间中。
>
> **元空间、永久代是方法区的具体的落地实现，方法区看作是一块独立于Java堆的内存空间，它主要用来存储所加载的类信息的。**

根据Java虚拟机规范的规定，如果方法区无法满足新的内存分配需求时，将抛出 OutOfMemoryError异常。

### 运行时常量池

运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。Java虚拟机对于Class文件每一部分（自然也包括常量池）的格式都有严格规定，如每一个字节用于存储哪种数据都必须符合规范上的要求才会被虚拟机认可、加载和执行，但对于运行时常量池，Java虚拟机规范并没有做任何细节的要求，不同提供商实现的虚拟机可以按照自己的需要来实现这个内存区域，不过一般来说，除了保存Class文件中描述的符号引用外，还会把由符号引用翻译出来的直接引用也存储在运行时常量池中。

**既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。**

### 直接内存

直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中 定义的内存区域。但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常出现。

在JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道Channel与缓冲区Buffer的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，则肯定还是会受到本机总内存（包括物理内存、SWAP分区或者分页文件）大小以及处理器寻址空间的限制，一般服务器管理员配置虚拟机参数时，会根据实际内存去设置-Xmx等参数信息，但经常忽略掉直接内存，使得**各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），从而导致动态扩展时出现OutOfMemoryError异常**。

## 1.2 HotSpot虚拟机对象

这里以最常用的HotSpot虚拟机和最常用的Java堆来研究一下对象分配、布局和访问的全过程。

### 1.2.1 对象的创建

当Java虚拟机遇到一条字节码new指令时，首先去检查这个指令的参数能否在常量池中定位到一个类的符号引用，并且检查这个类的符号引用代表的类是否被加载、解析初始化过，如果没有需要执行相应的类加载过程。

在类加载检查通过后，接下来虚拟机将为新对象分配内存，对象所需的内存大小在类加载完成后就可以确定，为对象分配空间相当于把一块确定的内存从Java堆中划分出来，然后进行分配，一共有两种方式选择哪种方式是由Java对是否规整决定：

1. 如果Java堆规整，使用过的内存在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那么分配内存只需要将指针挪动一段与对象大小相等的距离，这种分配方式称为指针碰撞
2. 如果不规整，使用的内存和未使用的交错在一起，那么虚拟机就必须维护一个列表，记录哪些内存块可用，然后找到一块足够大的空间划分给对象实例，并更新表的记录，这种被称为空闲列表

Java堆是否规整是由GC是否带有空间压缩整理能力决定的。当使用Serial、ParNew等带压缩整理的GC时，系统的分配算法就是指针碰撞，如果使用CMS基于清除算法的GC时，理论上就采用空闲列表来分配内存。

> 除了划分空间，分配内存还要考虑创建对象是否频烦，在并发情况下能否线程安全。解决此问题有两种方案：一是对分配内存空间的动作进行同步处理（通过CAS加失败重试保证更新操作的原子性），另一种就是把内存分配的动作按照线程划分在不同空间中，每个线程在Java队中预先分配一小块内存称为本地线程分配缓冲（TLAB），然后哪个线程要分配就在对应的本地缓冲中分配，只有本地缓冲区用完了需要分配新的缓冲区时才需要同步锁定。**虚拟机是否使用TLAB可通过`-XX: +/-UseTLAB`参数设定。**

分配完内存后虚拟机将分配的内存空间初始化为零值，如果使用TLAB也可以在分配TLAB时进行。这步操作保证了Java代码中不赋初值也能直接使用。

接下来虚拟机，需要对对象进行必要的设置，比如这个对象是哪个类的实例、如何找到类的元数据信息、对象的GC分代年龄等信息。这些信息存放在对象头之中。

上面工作全部完成后，虚拟机视角下对象已经创建完成，从Java程序角度来说，对象才刚开始创建，构造函数还未开始执行，所有字段都是默认的零值。一般来说，new指令之后会接着执行`<init>()`方法，按照程序的意愿进行初始化，这样一个真正可用的对象才算被构造出来。

下图是是HotSpot虚拟机字节码解释器（bytecodeInterpreter.cpp）中的代码片段，可以了解HotSpot的对象创建过程。

> 深入理解Java虚拟机第三版中强调，这个解释器实现很少有机会实际使用，大部分平台上都使用模板解释器。不过这段代码用于了解HotSpot的运作过程是没有什么问题的。（代码注释主要来自于深入理解Java虚拟机和源码自带的注释，以及个人的补充）

~~~c++
CASE(_new): {
    u2 index = Bytes::get_Java_u2(pc+1);
    ConstantPool* constants = istate->method()->constants();
    //确保常量池中是已解释的类
    if (!constants->tag_at(index).is_unresolved_klass()) {
        // Make sure klass is initialized and doesn't have a finalizer
        //确保是klassOop和InstanceKlassOop
        Klass* entry = constants->resolved_klass_at(index);
        InstanceKlass* ik = InstanceKlass::cast(entry);
        //确保对象所属类型经过了初始化阶段
        if (ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
            size_t obj_size = ik->size_helper();
            oop result = NULL;
            // If the TLAB isn't pre-zeroed then we'll have to do it
            //如果TLAB过程中没有预置零，这里就需要进行初始化预置零值
            bool need_zero = !ZeroTLAB;
            if (UseTLAB) {
                result = (oop) THREAD->tlab().allocate(obj_size);
            }
            // Disable non-TLAB-based fast-path, because profiling requires that all
            // allocations go through InterpreterRuntime::_new() if THREAD->tlab().allocate
            // returns NULL.
            #ifndef CC_INTERP_PROFILE
            if (result == NULL) {
                need_zero = true;
                // Try allocate in shared eden
                //在eden中分配对象，下面是retry标记，用于分配失败跳转重试
                retry:
                HeapWord* compare_to = *Universe::heap()->top_addr();
                HeapWord* new_top = compare_to + obj_size;
                //cmpxchg是x86中的cas指令，这里是通过cas分配空间，并发失败则转到retry中重试直至分配成功
                if (new_top <= *Universe::heap()->end_addr()) {
                    if (Atomic::cmpxchg(Universe::heap()->top_addr(), compare_to, new_top) != compare_to) {
                        goto retry;
                    }
                    result = (oop) compare_to;
                }
            }
            #endif
            if (result != NULL) {
                // Initialize object (if nonzero size and need) and then the header
                //如果需要，初始化对象和头文件
                if (need_zero ) {
                    HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
                    obj_size -= sizeof(oopDesc) / oopSize;
                    if (obj_size > 0 ) {
                        memset(to_zero, 0, obj_size * HeapWordSize);
                    }
                }
                //根据是否使用偏向锁，设置对象的头信息
                if (UseBiasedLocking) {
                    result->set_mark(ik->prototype_header());
                } else {
                    result->set_mark(markWord::prototype());
                }
                result->set_klass_gap(0);
                result->set_klass(ik);
                // Must prevent reordering of stores for object initialization
                // with stores that publish the new object.
                OrderAccess::storestore();
                SET_STACK_OBJECT(result, 0);
                UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
            }
        }
    }
    // Slow case allocation
    CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
            handle_exception);
    // Must prevent reordering of stores for object initialization
    // with stores that publish the new object.
    OrderAccess::storestore();
    //将对象的引用入站，继续执行下一条指令
    SET_STACK_OBJECT(THREAD->vm_result(), 0);
    THREAD->set_vm_result(NULL);
    UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
}
~~~

### 1.2.2 对象的内存布局

在HotSpot虚拟机中，对象在堆内存中可以分为三个部分：对象头、实例数据和对齐补充。

**HotSpot虚拟机对象的对象头信息**分为三类信息：第一类用于存储自身运行时数据（比如哈希码，GC年龄分代，锁状态标志、线程持有的锁、偏向锁ID等等），这部分数据长度在32位和64位虚拟机中分别为32个比特和64个比特。官方称为**Mark Word**。对象需要存储的数据很多，其实已经超出了32或64比特的限度，因此考虑到虚拟机空间效率Mark Word被设计成一个有动态定义的数据结构，以便在极小的空间内存储尽量多的数据。

> 以下内容来自我的高并发编程学习笔记第二部分
>
> Mark Word字段的结构与Java内置锁的状态强相关。为了让Mark Word字段存储更多的信息，JVM将Mark Word最低两个位设置为Java内置锁状态位，不同锁状态下的32位Mark Word结构如下：
>
> <table>
>     <thead>
>         <tr>
> 	    <th rowspan="3">内置锁状态</th>
> 	    <th colspan="2" >25bit</th>
> 	    <th rowspan="3">4bit</th> 
>         <th>1bit</th>
>         <th>2bit</th>
> 	</tr>
> 	<tr>
> 	    <th rowspan="2">23bit</th>
>         <th rowspan="2">2bit</th>
>         <th rowspan="2">biased偏向标志位</th>
>         <th rowspan="2">lock锁状态</th>
> 	</tr>
>     </thead>
> 	<tbody>
>     	<tr>
> 	    <td>无锁</td>
>         <td colspan="2" >对象的hashCode(25bit)</td>
>         <td >分代年龄</td>
>         <td >0</td>
>         <td >01</td>
> 	</tr>
>     <tr>
> 	    <td>偏向锁</td>
>         <td>线程ID(23bit)</td>
>         <td>epoch(2bit)</td>
>         <td >分代年龄</td>
>         <td >1</td>
>         <td >01</td>
> 	</tr>
>     <tr>
> 	    <td>轻量级锁</td>
>         <td colspan="4" >ptr_to_lock_record方法栈帧中的锁记录指针(30bit)</td>
>         <td >00</td>
> 	</tr>
>     <tr>
> 	    <td>重量级锁</td>
>         <td colspan="4" >ptr_to_heavyweight_monitor指向重量级锁监视器的指针(30bit)</td>
>         <td >10</td>
> 	</tr>
>     <tr>
> 	    <td>GC标志</td>
>         <td colspan="4" >空(30bit)</td>
>         <td >11</td>
> 	</tr>
>     </tbody>
> </table>
>64位Mark Word结构如下：
> 
><table>
>  <thead>
>         <tr>
>    	    <th rowspan="2">内置锁状态</th>
> 	    <th rowspan="2" colspan="3" >57bit</th>
> 	    <th rowspan="2">4bit</th> 
>      <th>1bit</th>
>         <th>2bit</th>
>    	</tr>
> 	<tr>
>      <th>biased</th>
>         <th>lock</th>
>    	</tr>
>  </thead>
>    	<tbody>
>  	<tr>
>    	    <td>无锁</td>
>      <td >unused(25bit)</td>
>         <td >对象的hashCode(31bit)</td>
>         <td >unused(1bit)</td>
>         <td >分代年龄</td>
>         <td >0</td>
>         <td >01</td>
>    	</tr>
>  <tr>
>    	    <td>偏向锁</td>
>      <td>线程ID(54bit)</td>
>         <td>epoch(2bit)</td>
>         <td >unused(1bit)</td>
>         <td >分代年龄</td>
>         <td >1</td>
>         <td >01</td>
>    	</tr>
>  <tr>
>    	    <td>轻量级锁</td>
>      <td colspan="5" >ptr_to_lock_record方法栈帧中的锁记录指针(62bit)</td>
>         <td >00</td>
>    	</tr>
>  <tr>
>    	    <td>重量级锁</td>
>      <td colspan="5" >ptr_to_heavyweight_monitor指向重量级锁监视器的指针(62bit)</td>
>         <td >10</td>
>    	</tr>
>  <tr>
>    	    <td>GC标志</td>
>      <td colspan="5" >空(62bit)</td>
>         <td >11</td>
>    	</tr>
>  </tbody>
>    </table>
> 
>
>在64位的Mark Word中
> 
>- lock：锁状态标记位，占两个二进制位，由于希望用尽可能少的二进制位表示尽可能多的信息，因此设置了lock标记。该标记的值不同，整个Mark Word表示的含义就不同。
> - biased：表示当前对象无锁还是偏向锁
> - age：分代年龄，在GC中，对象在Survivor区复制一次，年龄就增加1，当对象达到设定的阈值就会晋升老年代，默认并行GC的年龄阈值为15，并发GC的年龄阈值是6.因为只有4位所以最大是15.
> - identity_hashcode:对象的hashCode，31位对象标识哈希码采用延迟加载，只有调用`Object.hashCode()`方法或者`System.identityHashCode()`方法计算对象的HashCode后，其结果将被写到该对象头中。当对象被锁定时，该值会移动到Monitor（监视器）中.
> - epoch:偏向时间戳
> - thread：线程ID，54位是持有偏向锁的线程ID
> - ptr_to_lock_record，轻量级锁状态下指向栈帧中锁记录的指针。
> - ptr_to_heavyweight_monitor，重量级锁状态下指向对象监视器的指针。

另一类就是类型指针，即对象指向它的类型元素据的指针，Java虚拟机通过这个指针来确定这个对象是哪个类的实例。并不是所有的虚拟机都必须在对象数据上保留指针类型。

第三类就是如果对象是一个Java数组，那么对象头中还会有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是如果数组的长度不确定将无法通过元数据信息推断出数组的大小。

**对象在堆内存中的第二个部分就是实例信息**了，这部分是对象真正存储的有效信息，是我们在代码中定义的各种类型的字段内容，无论是父类继承的还是在子类中定义的都必须记录下来，这部分的存储顺序会受到虚拟机分配策略参数（`-XX: FieldAllocationStyle`）和字段在Java源码中的定义顺序的影响。

HotSpot虚拟机默认的分配顺序为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers，OOPs），从以上默认的分配策略中可以看到，相同宽度的字段总是被分配到一起存放，在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。如果HotSpot虚拟机的`+XX：CompactFields`参数值为true（默认就为true），那子类之中较窄的变量也允许插入父类变量的空隙之中，以节省出一点点空间。

**对象在堆内存中的第三部分是对齐补充**，这部分不是必然的，由于HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说就是任何对象的大小都必须是8字节的整数倍。对象头部分已经被精心设计成8字节的整数倍，因此如果实例数据没有对齐，需要这部分来补充。

### 1.2.3 对象的访问和定位

创建对象后自然是为了使用它，Java程序会通过栈上的reference数据来操作堆上的具体对象，由于reference类型在Java虚拟机规范中只规定了是一个指向对象的引用，并没有规定这个引用该如何定位访问堆中对象，因此对象访问方式也是由虚拟机实现而定的，主流的访问方式有两种，句柄和直接指针。

句柄访问，Java堆中会划分出一块内存作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的地址信息，结构图如下：

![image-20220803135059364](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803135059364.png)

直接指针，Java堆中对象的内存布局必须考虑如何放置访问类型数据的相关信息，reference中存储的直接就是对象地址，如果只是访问对象本身就不需要多一次间接访问的开销。

![image-20220803135303315](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803135303315.png)

这两种方式各有优势，使用句柄的最大好处就是reference中存储的是稳定句柄地址，在对象被移动时（比如GC时）只会改变句柄中的实例数据指针，而reference本身不需要被修改。

使用直接指针来访问最大的好处就是速度更快，它节省了一次指针定位的时间开销，由于对象访问在Java中非常频繁，因此这类开销积少成多也是一项极为可观的执行成本，就本书讨论的主要虚拟机HotSpot而言，它主要使用第二种方式进行对象访问。

## 1.3 OOM异常示例

本小节所有的示例在注释中都有对应的VM参数，请运行程序时自行配置，IDEA添加VM参数方法如下：

![image-20220803140356785](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803140356785.png)

### 1.3.1 Java堆溢出

Java堆主要作用就是存放对象的实例数据，因此我们调低堆内存，然后一直创建对象，只要总容量触及最大堆的容量限制就会发生内存溢出异常。代码如下：

```java
/**
 * <p>
 * -Xms20m  堆的最小值
 * -Xmx20m  堆的最大值
 * -XX:+HeapDumpOnOutOfMemoryError  让虚拟机出现内存溢出时Dump出当前的内存堆转储快照，以便事后分析
 * </p>
 */
public class HeapOOM {
    static class OOMObject{}
    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
        }
    }
}
```

运行结果：

![image-20220803140717704](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803140717704.png)

Java堆内存的OutOfMemoryError异常是实际应用中最常见的内存溢出异常情况。出现Java堆内存溢出时，异常堆栈信息“java.lang.OutOfMemoryError”会跟随进一步提示“Java heap space”。要解决这个内存区域的异常，常规的处理方法是首先通过内存映像分析工具（如Eclipse Memory Analyzer）对Dump出来的堆转储快照进行分析。第一步首先应确认内存中导致OOM的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）。

> 内存泄漏和内存溢出区别：
>
> 内存泄漏*memory leak*：是指程序在申请内存后，无法释放已申请的内存空间，导致系统无法及时回收内存并且分配给其他进程使用。通常少次数的内存无法及时回收并不会到程序造成什么影响，但是如果在内存本身就比较少获取多次导致内存无法正常回收时，就会导致内存不够用，最终导致内存溢出。
> 内存溢出*out of memory*：:指程序申请内存时，没有足够的内存供申请者使用，导致数据无法正常存储到内存中。也就是说给你个int类型的存储数据大小的空间，但是却存储一个long类型的数据，这样就会导致内存溢出。

如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链（可达性分析算法，用于分析对象是否需要被回收），找到泄漏对象是通过怎样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收它们，根据泄漏对象的类型信息以及它到GC Roots引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置。

如果不是内存泄漏，换句话说就是内存中的对象确实都是必须存活的，那就应当检查Java虚拟机的堆参数(-Xmx与-Xms)设置，与机器的内存对比，看看是否还有向上调整的空间。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗。

### 1.3.2 虚拟机栈和本地方法栈溢出

HotSpot虚拟机并不区分虚拟机栈和本地方法栈，对于HotSpot来说-Xoss参数（用于设置本地方法栈大小）虽然存在但是实际上没有任何效果，栈容量只能由-Xss参数来设定。关于虚拟机和本地方法栈，在Java虚拟机规范中描述了两种异常：一是如果线程请求的栈深度大于虚拟机允许的最大深度，将抛出StackOverflowError异常。二是如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出。

虽然Java虚拟机规范允许是否支持栈的动态扩展，但是HotSpot虚拟机不支持扩展，除非在创建线程申请内存时无法获取足够内存而出现OOM，否则在线程运行时是不会因为扩展而导致内存溢出的，只会因为栈容量无法容纳新的栈帧而导致StackOverflowError。

我们下面做两个示例，首先通过-Xss参数减少栈内存容量：

```java
/**
 * -Xss128k
 */
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak(){
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        }catch (Throwable e){
            System.out.println("stack length"+oom.stackLength);
            throw e;
        }
    }
}
```

运行结果：

![image-20220803161125759](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803161125759.png)

然后我们通过大量的本地变量增大方法帧中本地变量表的长度：

```java
public class JavaVMStackSOF {
    private static int stackLength = 1;
    public static void test(){
        long    test1,test2,test3,test4,test5,test6,test7,test8,test9,test10,
                test11,test12,test13,test14,test15,test16,test17,test18,test19,test20,
                test21,test22,test23,test24,test25,test26,test27,test28,test29,test30,
                test31,test32,test33,test34,test35,test36,test37,test38,test39,test40,
                test41,test42,test43,test44,test45,test46,test47,test48,test49,test50,
                test51,test52,test53,test54,test55,test56,test57,test58,test59,test60,
                test61,test62,test63,test64,test65,test66,test67,test68,test69,test70,
                test71,test72,test73,test74,test75,test76,test77,test78,test79,test80,
                test81,test82,test83,test84,test85,test86,test87,test88,test89,test90,
                test91,test92,test93,test94,test95,test96,test97,test98,test99,test100;
        stackLength++;
        test();
        test1=test2=test3=test4=test5=test6=test7=test8=test9=test10=
                test11=test12=test13=test14=test15=test16=test17=test18=test19=test20=
                test21=test22=test23=test24=test25=test26=test27=test28=test29=test30=
                test31=test32=test33=test34=test35=test36=test37=test38=test39=test40=
                test41=test42=test43=test44=test45=test46=test47=test48=test49=test50=
                test51=test52=test53=test54=test55=test56=test57=test58=test59=test60=
                test61=test62=test63=test64=test65=test66=test67=test68=test69=test70=
                test71=test72=test73=test74=test75=test76=test77=test78=test79=test80=
                test81=test82=test83=test84=test85=test86=test87=test88=test89=test90=
                test91=test92=test93=test94=test95=test96=test97=test98=test99=test100=0;
    }

    public static void main(String[] args) {
        try {
            test();
        }catch (Throwable e){
            System.out.println("stack length"+stackLength);
            throw e;
        }
    }
}
```

![image-20220803161551482](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803161551482.png)

无论是由于栈帧太大还是虚拟机栈容量太小，当新的栈帧内存无法分配的时候，HotSpot虚拟机抛出的都是StackOverflowError异常。可是如果在允许动态扩展栈容量大小的虚拟机上，相同代码则会导致不一样的情况。

下图来自深入理解Java虚拟机第三版（这个实验我没做，jdk1.0.2确实远古）：

![image-20220803161709737](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803161709737.png)

原因其实是这样的，因为操作系统分配给每个进程的内存是有限的，比如32位windows单进程最大2GB，Hotspot提供了可以控制Java堆和方法区这两部分的内存的最大值，剩余的内存就是最大内存2GB减去最大堆容量和最大方法区容量，如果在把直接内存和虚拟机进程本身的内存去掉的话，**剩下的内存就由虚拟机栈和本地方法栈来分配了，因此每个线程分配到的栈内存越大，可建立的线程数量就越少，建立线程就很容易把剩下的内存耗尽**。

下图来自深入理解Java虚拟机第三版（这个实验我也没做，32位windows比较麻烦。doge）：

![image-20220803162837872](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803162837872.png)

出现StackOverflowError异常时，会有明确错误堆栈可供分析，相对而言比较容易定位到问题所在。如果使用HotSpot虚拟机默认参数，栈深度在大多数情况下到达1000~2000是完全没有问题，对于正常的方法调用（包括不能做尾递归优化的递归调用），这个深度应该完全够用了。但是，如果是建立过多线程导致的内存溢出，在不能减少线程数量或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少栈容量来换取更多的线程。这种通过“减少内存”的手段来解决内存溢出的方式，一般比较难以想到，这一点读者需要在开发32位系统的多线程应用时注意。也是由于这种问题较为隐蔽，从JDK 7起会给出possibly out of memory or process/resource limits reached的提示。

### 1.3.3 方法区和运行时常量池溢出

由于运行时常量池也是方法区的一部分，所以两者测试可以放到一起。

这里由于深入理解Java虚拟机中使用了jdk6、7、8等jdk进行测试，这里安装比较麻烦因此暂时不进行实验。这里暂时引用周志明大佬在书中的测试过程：

![image-20220810135348260](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810135348260.png)

![image-20220810135413430](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810135413430.png)

![image-20220810135518201](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810135518201.png)

![image-20220810135652522](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220810135652522.png)

### 1.3.4 本机直接内存溢出

直接内存的大小可以通过-XX:MaxDirectMemorySize=10M来指定，如果不指定则与Java堆的最大值一致。

直接内存溢出例子如下：

```java
/**
 * -Xmx20M指定Java 最大堆
 * -XX:MaxDirectMemorySize=10M 指定直接内存大小
 */
public class DirectOOM {
    private static final int _1MB = 1024*1024;

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        final Field declaredField = Unsafe.class.getDeclaredFields()[0];
        declaredField.setAccessible(true);
        Unsafe unsafe = (Unsafe) declaredField.get(null);
        while (true){
            unsafe.allocateMemory(_1MB);
        }
    }
}
```

运行结果：

![image-20220804093047300](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220804093047300.png)

我在实验深入理解Java虚拟机一书中的这个例子时加上了-XX:+HeapDumpOnOutOfMemoryError来打印dump日志，但是却发现并没有打印出日志来（这里猜测是将操作系统内存打满，无法在打印dump日志），于是我加了计数器发现计数器刷了很多轮如下图（一直在刷，这是截取的一部分）：

![image-20220804095744268](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220804095744268.png)

理论上我设置了10M最大堆外内存应该很快就跑完了，于是通过万能的互联网发现（主要是通过这篇博客进行了了解[[JVM]了断局: 堆外内存无法 [ -XX:MaxDirectMemorySize ] 限制](https://blog.csdn.net/zhanglong_4444/article/details/116701143)）**反射获取的unsafe实例不受JVM最大堆内存参数控制而是受操作系统的内存限制**。只有Java自带的`ByteBuffer.allocateDirect`或者`new DirectByteBuffer`这种方式获取的直接内存才受参数-XX:MaxDirectMemorySize限制。我们来测试一下，代码如下：

```java
/**
 * -Xmx20M指定Java 最大堆
 * -XX:MaxDirectMemorySize=10M 指定直接内存大小
 * -XX:+HeapDumpOnOutOfMemoryError 转储快照
 */
public class DirectOOM {
    private static final int _1MB = 1024*1024;
    private static int num=0;

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
//        final Field declaredField = Unsafe.class.getDeclaredFields()[0];
//        declaredField.setAccessible(true);
//        Unsafe unsafe = (Unsafe) declaredField.get(null);
        List<ByteBuffer> list = new ArrayList<>();
        while (true){
            num++;
            System.out.println(num);
            //unsafe.allocateMemory(_1MB);
            ByteBuffer buffer = ByteBuffer.allocateDirect(_1MB);
            list.add(buffer);
        }
    }
}
```

运行结果：

![image-20220804100547514](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220804100547514.png)

# 二、垃圾收集器与内存分配策略

Java运行时内存区域的各个部分，其中程序计数器、虚拟机栈、本地方法栈这三个区域随线程生，随线程死，栈中的栈帧随着方法的进入和推测，进行出栈和入栈操作，每一个栈帧分配多少内存基本上是在类结构确定下来时就已知的，所以这几个区域不用过多考虑如何回收内存，因为当方法结束或线程结束时，内存基本上就跟着回收了。

而Java堆和方法区有着不确定性，一个接口的多个实现类的内存可能不一样，一个方法所执行的不同条件分支所需要的内存也不一样，只有处于运行期间才知道程序究竟会创建多少对象，这些内存的分配是动态的。垃圾回收器所关注的正是这部分内存该如何管理。

## 2.1 判断对象是否存活

在堆里面存放着Java中几乎所有的对象实例，垃圾回收器回收前首先要做的就是确定对象是否存活

### 2.1.1 引用计数算法

在对象中添加一个引用计数器，每当有一个地方引用它时，计数器加一；当引用失效时，计数器减一；任何时刻计数器为0的对象就是不可在被使用的。虽然引用计数算法占用了一些额外的内存空间来计数，但是原理简单判定效率高。但是这个看似简单的算法需要很多例外情况要考虑，必须配合额外大量处理才能正确工作，譬如单纯的应用计数就很难解决对象之间的互相循环引用问题。在主流的Java虚拟机中都没有选择引用计数法管理内存。

我们看下面的例子：

```java
/**
 * -verbose:gc 在控制台输出GC情况
 * -XX:+PrintGCDetails 打印输出详细的GC收集日志的信息.
 */
public class ReferenceCount {
    public Object instance = null;
    private static final int _1MB = 1024*1024;
    private byte[] size = new byte[2*_1MB];

    public static void testGC(){
        ReferenceCount a = new ReferenceCount();
        ReferenceCount b = new ReferenceCount();
        a.instance = b.instance;
        b.instance = a.instance;
        a = null;
        b = null;
        System.gc();
    }

    public static void main(String[] args) {
        testGC();
    }
}
```

> 为什么说单纯的引用计数难以解决循环引用呢？比如上面的例子ab对象创建后计数为1，然后他们互相引用后计数为2，最后在置为null，这时理论上可以被垃圾回收，但这时引用只是变回1，并不满足回收的条件，如果ab间的引用一直存在，就永远无法被收回了。

运行结果如下：

![image-20220804162250135](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220804162250135.png)

从运行结果可以看出Java虚拟机进行了垃圾回收，意味着没有因为互相引用而放弃回收，也侧面说明了Java虚拟机使用的不是计数算法。

### 2.1.2 可达性分析算法

当前主流的商用语言（Java、C#等）都是通过可达性分析来判定对象是否存活的。这个算法的基本思路就是通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始向下搜索，搜索过的路径称为引用链，如果某个对象与GC Roots没有引用链，或者是从GC Roots到该对象不可达，那么此对象不可能在被使用了。如下图Obj5678这四个对象由于不可达GC Roots因此可被回收

![image-20220804165140526](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220804165140526.png)

在Java技术体系中以下几种可以固定作为GC Roots：

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，比如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。
- 在方法区中类静态属性引用的对象，比如Java类的引用类型静态变量。
- 在方法区中常量引用的对象，比如字符串常量池中的引用。
- 在本地方法栈中的JNI引用的对象
- Java虚拟机栈内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（OutOfMemoryError等），还有系统类加载器
- 所有被同步锁（synchronized）持有的对象。
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调，本地代码缓存等

除了这些固定的GC Roots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象临时性的加入，共同构建完整的GC Roots集合。比如分代垃圾收集和局部回收，如果只针对Java堆中的某一块内存区域发器垃圾收集时，必须考虑到内存区域是虚拟机自己的实现，不是独立封闭的，**所以某个内存区域的对象完全有可能被堆内的其他区域对象所引用，这时就需要将关联区域的对象也加入到GC Roots中，才能保证可达性分析的正确**。

### 2.1.3 Java中的引用

在jdk1.2之前，Java中的引用是很传统的定义，只要reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称该reference数据是代表某块内存、某个对象的引用。在jdk1.2,Java对引用的概念进行了扩充，将引用分为强引用、软引用、弱引用和虚引用。强度一次递减。

- 强引用是最传统的引用的定义，是指在程序代码中普遍存在的引用赋值，即Object obj=new Object()这种引用关系，无论在什么情况下，只要强引用关系还存在，垃圾收集器就永远不会收掉被引用的对象。

- 软引用是用来描述一些还有用，但非必须的对象，只要被软引用关联着的对象，在系统即将要发生内存溢出前，会把这些对象列为回收范围进行第二次回收，如果这次回收还没有足够的内存才会抛出内存溢出异常。在JDK1.2后提供了SoftReference类来实现软引用。

- 弱引用。这也是用来描述那些非必需对象的，但它比软引用还要弱一些，只要到下一次垃圾收集，无论内存是否足够，都会回收掉只被弱引用关联的对象。在jdk1.2后提供了WeakReference来实现弱引用。

  在ThreadLocalMap中的key就是通过弱引用进行包裹的。具体可见我的[Java高并发笔记一多线程](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%B8%80%E5%A4%9A%E7%BA%BF%E7%A8%8B.md#173-threadlocal%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)

- 虚引用，又称为幽灵或幻影引用，它是最弱的一种引用，一个对象是否有虚引用存在，完全不对其生存空间有影响，也无法通过虚引用来取得一个对象实例，为一个对象设置虚引用的唯一目的是为了在这个对象被收集器回收前收到一个系统通知。在jdk1.2后提供PhantomReference来实现虚引用。

### 2.1.4 可达性分析的缓刑

在可达性分析算法标定为不可达的对象，不会立即被释放，而是处于“缓刑阶段”，**一个对象“死亡”，需要被标记两次**即如果对象在进行可达性分析后发现没 有与GC Roots相连接的引用链，那它将会被第一次标记，随后进行一次筛选，筛选的条件是此对象是 否有必要执行finalize()方法。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用 过，那么虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为确有必要执行finalize()方法，那么该对象将会被放置在一个名为F-Queue的 队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize() 方法。这里所说的“执行”是指虚拟机会触发这个方法开始运行，但并不承诺一定会等待它运行结束。这样做的原因是，如果某个对象的finalize()方法执行缓慢，或者更极端地发生了死循环，将很可能导致F-Queue队列中的其他对象永久处于等待，甚至导致整个内存回收子系统的崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会，稍后收集器将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己this关键字赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出“即将回收”的集 合；如果对象这时候还没有逃脱，那基本上它就真的要被回收了。

![image-20220805095427272](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805095427272.png)

```java
public class FinalizeGC {

    public static FinalizeGC SAVE_HOOK = null;

    public void isAlive(){
        System.out.println("i am alive");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize is executed");
        FinalizeGC.SAVE_HOOK = this;
    }

    public static void main(String[] args) throws InterruptedException {
        SAVE_HOOK = new FinalizeGC();
        SAVE_HOOK = null;
        System.gc();
        //finalize优先级低，暂停0.5秒进行等待
        Thread.sleep(500);
        if (SAVE_HOOK != null){
            SAVE_HOOK.isAlive();
        }else {
            System.out.println("i am dead");
        }

        SAVE_HOOK = null;
        System.gc();
        Thread.sleep(500);
        if (SAVE_HOOK != null){
            SAVE_HOOK.isAlive();
        }else {
            System.out.println("i am dead");
        }
    }
}
```

运行结果：

~~~
finalize is executed
i am alive
i am dead
~~~

这是因为任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，它的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了。

> finalize() 如今已被官方明确声明为不推荐使用的语法。finalize()能做的所有工作，使用try-finally或者其他方式都可以做得更好、更及时，所以建议完全可以忘掉Java语言里面的这个方法。

### 2.1.5 回收方法区

在Java虚拟机规范中可以不对方法区进行垃圾收集，因为方法区垃圾收集的性价比较低，在堆中尤其是新生代，一次回收可回收70%-90%的空间，而方法区却收效甚微。

方法区主要回收两部分：废弃的常量和不再使用的类型。回收废弃常量与Java堆中的对象类似，比如一个字符串进入常量池中，但是没有任何字符串对象引用，且虚拟机中也没有其他地方引用，那么这时这个对象可以被清理出常量池。常量池中的其他类、方法也类似。

判断一个常量是否废弃很简单，但是判定一个类型是否属于不在被使用的类，这个条件就很苛刻，需要满足三个条件：

- 该类的所有的实例都已经被回收，也就是堆中不存在该类及任何派生子类的实例
- 加载该类的类加载器已经被回收（这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的）
- 该队对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

> Java虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是“被允许”，而并不是和对象一样，没有引用了就必然会回收。关于是否要对类型进行回收，HotSpot虚拟机提供了`-Xnoclassgc`参数进行控制，还可以使用`-verbose:class`以及`-XX: +TraceClass-Loading`、`-XX: +TraceClassUnLoading`查看类加载和卸载信息，其中`-verbose:class`和`-XX:+TraceClassLoading`可以在Product版的虚拟机中使用，`-XX:+TraceClassUnLoading`参数需要FastDebug版的虚拟机支持。 
>
> 在大量使用反射、动态代理、CGLib等字节码框架，动态生成JSP以及OSGi这类频繁自定义类加载器的场景中，通常都需要Java虚拟机具备类型卸载的能力，以保证不会对方法区造成过大的内存压力。

## 2.2 垃圾收集算法

### 2.2.1 分代收集算法

分代收集的思路非常简单，根据对象的生存周期将内存划分，然后进行分区管理，当前商业虚拟机的垃圾收集器，大多数都遵循了分代收集理论，分代收集实质上是一套符合大多数程序运行实际情况的经验法则，它建立在两个分代假说上：

- 弱分代假说：绝大多数对象都是朝生夕灭的。
- 强分代假说：熬过越多次垃圾收集过程的对象就越难以消亡

这两个假说奠定了多款垃圾收集器的一致原则：收集器应该将Java堆划分出不同的区域，然后将回收对象依据年龄（年龄就是熬过垃圾收集的次数）分配到不同的区域之中存储。 如果一个区域中大多数对象都是朝生夕灭， 难以熬过垃圾收集过程的话， 那么把它们集中放在一起， 每次回收时只关注如何保留少量存活而不是去标记那些大量将要被回收的对象， 就能以较低代价回收到大量的空间；如果剩下的都是难以消亡的对象， 那把它们集中放在一块， 虚拟机便可以使用较低的频率来回收这个区域，这就同时兼顾了垃圾收集的时间开销和内存的空间有效利用。

在Java堆划分出不同的区域之后，垃圾收集器才可以每次只回收其中某一个或者某些部分的区域，因而才有了“Minor GC”“Major GC”“Full GC”这样的回收类型的划分；**也才能够针对不同的区域安排与里面存储对象存亡特征相匹配的垃圾收集算法**——因而发展出了“标记-复制算法”“标记-清除算 法”“标记-整理算法”等针对性的垃圾收集算法。

但是加入现在进行一次只局限一次新生代区域内的收集，但新生代的对象完全有可能被老年代引用，为了找出该区域的存活对象，不得不在固定的GC Roots之外，额外遍历整个老年代中所有对象来确保可达性分析的正确性，反过来也是一样。但这样会给内存回收带来较大的性能负担。为了解决这个问题，我们在上面两条假说上新增一条：

- 跨代引用假说：跨代引用相对于同代引用来说仅占极少数

这其实是可根据前两条假说逻辑推理得出的隐含推论：存在互相引用关系的两个对象，是应该倾向于同时生存或者同时消亡的。举个例子，如果某个新生代对象存在跨代引用，由于老年代对象难以消亡，该引用会使得新生代对象在收集时同样得以存活，进而在年龄增长之后晋升到老年代中，这时跨代引用也随即被消除了。 

依据这条假说，我们就不应再为了少量的跨代引用去扫描整个老年代，也不必浪费空间专门记录每一个对象是否存在及存在哪些跨代引用，只需在新生代上建立一个全局的数据结构（该结构被称为“记忆集”，Remembered Set），这个结构把老年代划分成若干小块，标识出老年代的哪一块内存会存在跨代引用。此后当发生Minor GC时，只有包含了跨代引用的小块内存里的对象才会被加入到GC Roots进行扫描。虽然这种方法需要在对象改变引用关系（如将自己或者某个属性赋值）时维护记录数据的正确性，会增加一些运行时的开销，但比起收集时扫描整个老年代来说仍然是划算的。

> Minor GC/Young GC：指目标只是新生代的垃圾收集
>
> Major GC/Old GC：指目标只是老年代的垃圾收集，目前只有CMS收集器才会有单独收集老年代的行为
>
> Mixed GC：指目标是收集整个新生代和部分老年代的垃圾收集，目前只有G1会有这种行为
>
> Full GC：收集整个Java堆和方法区的垃圾收集。

### 2.2.2 标记清除算法

最早出现的也是最基础的垃圾收集算法就是标记-清除算法，算法分为标记和清除两个阶段：首先标记处所有的需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回收所有未标记的对象。如下图（图片来自深入理解Java虚拟机第三版）

![image-20220808091254228](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220808091254228.png)

标记清除算法有两个不足：

1. 执行效率不稳定，如果Java堆包含大量对象，其中大部分是需要被回收的，这是必须要进行大量标记清除的动作，导致执行效率随着对象数量的增长而降低。
2. 内存空间存在碎片化的问题，标记清楚后会产生大量不连续的内存碎片，空间碎片过多可能会导致分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

### 2.2.3 标记复制算法

标记复制算法是为了解决面向大量对象时执行效率低的问题，它将可用内存按容量划分为大小相等的两块，每次只使用一块，当其中一块的内存用完了，就将还活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉，如果内存中的多数对象都是存活的，那么这种算法会产生大量的内存复制的开销，但对于多数对象都是可回收的时候，算法仅需要复制少数的存活对象即可。而且每次都是对整个半区进行内存回收，分配时也不需要考虑空间碎片的复杂情况，只要移动栈顶指针，按顺序分配即可。但这种算法的缺点也很明显就是内存空间被减少了一半。如下图（图片来自深入理解Java虚拟机第三版）

![image-20220808130643008](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220808130643008.png)

现在的商用虚拟机大多都优先采用这种算法，IBM曾经对新生代“朝生夕灭”的特点做了更量化的诠释——新生代中的对象有98%熬不过第一轮，因此不需要按照1:1的比例分配新生区的内存空间。在1989年，Andrew Appel针对具备“朝生夕灭”特点的对象，提出了一种更优化的半区复制分代策略，现在称为“Appel式回收”。HotSpot虚拟机的Serial、ParNew等新生代收集器均采用了这种策略来设计新生代的内存布局。Appel式回收的具体做法是把新生代分为一块较大的Eden空间和两块较小的Survivor空间，每次分配内存只使用Eden和其中一块Survivor。发生垃圾搜集时，将Eden和Survivor中仍然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。**HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1，也即每次新生代中可用内存空间为整个新生代容量的90%（Eden的80%加上一个Survivor的10%），只有一个Survivor空间，即10%的新生代是会被“浪费”的**。当然，98%的对象可被回收仅仅是“普通场景”下测得的数据，任何人都没有办法百分百保证每次回收都只有不多于10%的对象存活，因此Appel式回收还有一个充当罕见情况的“逃生门”的安全设计，当Survivor空间不足以容纳一次Minor GC之后存活的对象时，就需要依赖其他内存区域（实际上大多就是老年代）进行分配担保（Handle Promotion）。

### 2.2.4 标记整理算法

标记复制算法在对象存活率较高时就要进行较多的复制操作，效率将会降低。并且如果不想浪费50%的空间就要有额外的空间进行担保，所以老年代一般不能使用标记复制算法。

针对老年代对象的存亡特征，出现了标记整理算法，其标记过程与标记清除算法一样，但后续不是直接对可回收对象进行清理，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存，如下图（图片来自深入理解Java虚拟机第三版）：

![image-20220808132115039](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220808132115039.png)

如果要移动存活对象，尤其是老年代这种大量对象存活区域，移动存活对象并更新所有引用这些对象的地方是一种负重极大的操作，且必须全程暂停用户应用程序才能进行，这就需要使用者权衡利弊，像这样的停顿最初被叫做Stop The World。

但如果跟标记-清除算法那样完全不考虑移动和整理存活对象的话，那么堆中的存活对象导致的空间碎片化问题就只能依赖更为复杂的内存分配器和内存访问器来解决。譬如通过“分区空闲分配链表”来解决内存分配问题（计算机硬盘存储大文件就不要求物理连续的磁盘空间，能够在碎片化的硬盘上存储和访问就是通过硬盘分区表实现的）。内存的访问是用户程序最频繁的操作，甚至都没有之 一，假如在这个环节上增加了额外的负担，势必会直接影响应用程序的吞吐量。

基于以上两点，是否移动对象都存在弊端，移动则内存回收时会更复杂，不移动则内存分配时会更复杂。从垃圾收集的停顿时间来看，不移动对象停顿时间会更短，甚至可以不需要停顿，但是从整个程序的吞吐量来看，移动对象会更划算。即使不移动对象会使得收集器的效率提升一些，但因内存分配和访问相比垃圾收集频率要高得多，这部分的耗时增加，总吞吐量仍然是下降的。HotSpot虚拟机里面关注吞吐量的Parallel Scavenge收集器是基于标记-整理算法的，而关注延迟的CMS收集器则是基于标记-清除算法的，这也从侧面印证这点。另外，还有一种中和的方式，做法是让虚拟机平时多数时间都采用标记-清除算法，暂时容忍内存碎片的存在，直到内存空间的碎片化程度已经大到影响对象分配时，再采用标记-整理算法收集一次，以获得规整的内存空间。基于标记-清除算法的CMS收集器面临空间碎片过多时采用的就是这种处理办法。

## 2.3 HotSpot的算法细节实现

### 2.3.1 根节点枚举

固定可作为GC Roots的节点主要在全局引用（例如常量和静态变量）与执行上下文（例如栈帧中的本地变量表）中，尽管目标明确，但保证查找过程高效并非十分容易，Java应用越做越大，光是方法区的大小就常有数百上千兆，若要逐个检查肯定消耗非常多的时间。迄今为止，所有收集器在根节点枚举这一步都是必须暂停用户线程的，因此毫无疑问**根节点枚举与标记整理算法都会面对Stop The World的困扰**。现在可达性分析算法耗时最长的查找引用链的过程已经可以做到和用户线程一起并发，但根节点枚举始终还是必须在一个能保障一致性的快照中才得以进行——这里“一致性”的意思是整个枚举期间执行子系统看起来就像被冻结在某个时间点上，不会出现分析过程中，根节点集合的对象引用关系还在不断变化的情况，若这点不能满足的话，分析结果准确性也就无法保证。这是导致垃圾收集过程必须停顿所有用户线程的其中一个重要原因，即使是号称停顿时间可控，或者（几乎）不会发生停顿的CMS、G1、ZGC等收集器，枚举根节点时也是必须要停顿的。

目前主流的Java虚拟机使用的都是准确式垃圾收集（准确式内存管理是指虚拟机可以知道内存中某个位 置的数据具体是什么类型。譬如内存中有一个32bit的整数123456，虚拟机将有能力分辨出它到底是一 个指向了123456的内存地址的引用类型还是一个数值为123456的整数），所以当用户线程停下来后，并不需要一个不漏的，检查完所有上下文和全局引用位置，虚拟机应当是有办法直接得到那些地方存放着对象引用的，**在HotSpot中是使用一个OopMap的数据结构来实现的**。一旦类加载完成，HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译的过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用，这样收集器在扫描时就可以直接得知这些信息了，并不需要一个不漏的从方法区等GC Roots开始查找。

### 2.3.2 安全点

在OopMap的帮助下，HotSpot可以快速完成GC Roots的枚举，但如果每一条指令都生成对应的OopMap，那么需要大量的额外内存空间，这样垃圾收集伴随而来的空间成本就会变得特别高昂。**实际上HotSpot并没有为每条指令都生成OopMap，只有特定的位置记录了这些信息，这样的位置被称为安全点。**有了安全点的设定，也就决定了用户执行时并非在代码指令流的任意位置都能停下来收集垃圾，而是强制要求必须到安全点才能暂停，因此安全点的选定既不能太少让垃圾收集器等待时间过长，也不能太多让垃圾收集器频烦清理增大内存负荷。

安全点的选择基本上是以是否具有*让线程长时间执行的特征*而选定的，“长时间执行”的最明显特征就是指令序列的复用，例如方法调用、循环跳转、异常跳转等都属于指令序列复用，所以只有具有这些指令才会产生安全点。

有了安全点，另外一个需要考虑的问题是如何在垃圾收集发生时让所有的线程都跑到最近的安全点，然后停顿下来这里有两种方式：

- 抢先式中断：不需要线程的执行代码主动配合，系统首先把所有的用户线程全部中断，如果发现有用户线程中断的地方不是安全点，就恢复这条线程执行，让它过一会在重新中断，直到跑到安全点。
- 主动式中断：当垃圾收集需要中断线程的时候，不直接对线程操作，仅仅简单的设置一个标志位，各个线程在执行时会不断主动轮询这个标志，一旦发现中断标志为真，就自己在最近的安全点主动中断挂起。

> 各个线程在执行时会不断主动轮询这个标志，那么轮询如何保证高效。下图来自深入理解Java虚拟机第三版
>
> ![image-20220808154713721](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220808154713721.png)
>
> 上面的原理可能有些绕，需要了解关于unix的自陷的知识（这里参考了UNIX技术内幕）
>
> 自陷和中断非常相似，它也是通过在总线上发起一个信号来打断当前运行的程序而获得CPU的，只不过该信号是由CPU自身产生的，比如非法指令。自陷有三大类：系统出错自陷、系统调用自陷和调试自陷，而在深入理解Java虚拟机书中所讲的是属于系统出错自陷中的内存管理违例（Memory Management Violation）。
>
> 只在内存管理单元打开，也就是使用虚拟内存时，才可能产生该违例。产生该违例的原因有：
>
> （1）**所访问的（虚拟）内存页面没有权限。比如试图向一个具有“只读”权限的页面写入数据。**
>
> （2）访问的地址超出页面范围
>
> 其向量位于内核地址168处。它也是程序中经常容易出现的错误，又称为“段违例”，UNIX下常见的“segmentation fault”就是指它。

### 2.3.3 安全区域

使用安全点的设计似乎已经完美解决如何停顿用户线程，让虚拟机进入垃圾回收状态的问题了。但是实际情况却不一定，安全点机制保证了程序执行时，在不太长的时间内就会遇到可进入垃圾收集过程的安全点。但是程序不执行时（没有分配处理器时间，比如用户线程处于Sleep或Blocked状态）线程无法响应虚拟机的中断请求，不能再走到安全的地方去中断挂起自己，虚拟机也显然不可能持续等待线程重新被激活分配处理器时间，对于这种情况就需要引入安全区域来解决。

安全区域是指能够确保某一段代码片段中，引用关系不会发生变化，因此在这个区域任意地方开始收集垃圾都是安全的。我们也可以把安全区域看作是被扩展拉伸了的安全点。

当用户线程执行到安全区域里面的代码时，首先会标识自己已经进入了安全区域，那样当这段时间里虚拟机要发起垃圾收集时就不必去管这些已声明自己在安全区域内的线程了。当线程要离开安全区域时，它要检查虚拟机是否已经完成了根节点枚举（或者垃圾收集过程中其他需要暂停用户线程的阶段），如果完成了，那线程就当作没事发生过，继续执行；否则它就必须一直等待，直到收到可以离开安全区域的信号为止。

### 2.3.4 记忆集和卡表

在分代收集算法（2.2.1）中，为了解决对象跨代引用带来的问题，垃圾收集器在新生代建立了名为记忆集（Remember Set）的数据结构，实际上不只是新生代和老年代才有跨代引用问题，所有涉及到部分区域收集的行为的垃圾收集器比如G1，ZGC和Shenandoah收集器都会面临相同的问题。

记忆集是一种用于记录**非收集区指向收集区的指针集合的抽象数据结构**，如果不考虑成本的话，可以用非搜集区域所有含跨代引用的对象数组来实现此结构。但是这种方式的空间和维护成本很高，实际上收集器只需要通过记忆集判断出某一块非搜集区域是否存在有指向搜集区域的指针就可以了。所以设计记忆集时可以用更粗犷的粒度来节省记忆集的存储和维护成本。比如：

- 字长精度：每个记录精确到一个机器字长（就是处理器的寻址位数，如常见的32位或64位，这个精度决定了机器访问物理内存地址的指针长度），该字包含跨代指针。
- 对象精度：每个记录精确到一个对象，该对象里有字段含有跨代指针
- 卡精度：每个记录精确到一块内存区域，该区域内有对象含有跨代指针

第三种方式用的是“卡表”（Card Table）的方式实现的记忆集，这也是最常用的一种记忆集形式。卡表是记忆集的一种具体实现，它记录了记忆集的记录精度，与堆内存的映射关系等。

卡表最简单的形式就是一个字节数组，HotSpot虚拟机中也是这么做的。

~~~c++
//卡表默认标记逻辑
//一个卡页的内存中通常包含不止一个对象，只要卡页内有一个（或更多对象的字段存在着跨代指针，那就将对应卡表的数组元素的值标识为1，称为这个元素变脏（Dirty），没有则标识为0。在垃 圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它 们加入GC Roots中一并扫描
CARD_TABLE [this address >>9]=0;
//将地址右移9位，相当于用地址除以512（2的9次方）。假设卡表卡页的起始地址为0，那么卡表项0、1、2对应的卡页起始地址分别为0、512、1024（卡表项索引号乘以卡页512字节）
~~~

字节数组CARD_TABLE的每一个元素都对应着其标识的内存区域中一块大小特定的内存块，这个内存页被称为卡页。一般来说，卡页大小是2的N次方幂的字节数，通过上面代码可以看出HotSpot的卡页是2的9次幂，即512字节。如果卡表标识内存区域的起始地址是0x0000的话，数组CARD_TABLE的第0、1、2号元素，分别对应了地址范围为0x0000～0x01FF、0x0200～0x03FF、0x0400～0x05FF的卡页内存块。如下图（图片来自深入理解Java虚拟机第三版）：

![image-20220809093219349](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809093219349.png)

### 2.3.5 写屏障

我们可以用记忆集来维护缩减GC Roots扫描范围的问题但是还没有解决卡表元素如何维护的问题，比如何时将其变脏，谁来把它变脏。

当其他分代区域引用了本区域对象时，其对应的卡表元素就应该变脏，变脏的时间点原则上应该发生在引用类型字段赋值的那一刻。如果是解释执行的字节码，虚拟机负责每条字节码指令的执行，有充分的介入空间，但如果是即时编译的机器指令流，那么就需要找到一个在机器码层面的手段，把维护卡表的动作放到每一个赋值操作中。

在HotSpot虚拟机中是通过写屏障（**这里的写屏障并不是内存屏障**）技术维护卡表的。写屏障可以看作是在虚拟机层面对引用类型字段赋值这个动作的AOP切面。在引用对象赋值时会产生一个环形通知，供程序执行额外的动作，赋值前叫写前屏障，赋值后叫写后屏障。直到G1出现前其他收集器都只用了写后屏障。

~~~c++
//更新卡表的简化逻辑
void oop field_store(oop* field,oop new _value){\
	//引用字段赋值操作
	*field = new value;
	//写后屏障，完成卡表状态更新
	post_write_barrier(field , new_value);
}
~~~

应用写屏障后，虚拟机就会为所有赋值操作生成相应的指令，一旦收集器在写屏障中增加了更新卡表操作，无论更新的是不是老年代对新生代对象的引用，每次只要对引用进行更新，就会产生额外的开销，不过这个开销与Minor GC时扫描整个老年代的代价相比还是低得多的。

除了写屏障的开销外，卡表在高并发场景下还面临着“伪共享”问题。伪共享是处理并发底层细节时一种经常需要考虑的问题，现代中央处理器的缓存系统中是以缓存行（Cache Line）为单位存储的，当多线程修改互相独立的变量时，如果这些变量恰好共享同一个缓存行，就会彼此影响（写回、无效化或者同步）而导致性能降低，这就是伪共享问题。

假设处理器的缓存行大小为64字节，由于一个卡表元素占1个字节，64个卡表元素将共享同一个缓存行。这64个卡表元素对应的卡页总的内存为32KB（64×512字节），也就是说如果不同线程更新的对象正好处于这32KB的内存区域内，就会导致更新卡表时正好写入同一个缓存行而影响性能。为了避免伪共享问题，一种简单的解决方案是不采用无条件的写屏障，而是先检查卡表标记，只有当该卡表元素未被标记过时才将其标记为变脏，即将卡表更新的逻辑变为以下代码所示：

~~~
if(CARD_TABLE [this address >> 9] !=0)
	CARD_TABLE [this address >> 9] ==0
~~~

在JDK 7之后，HotSpot虚拟机增加了一个新的参数-XX：+UseCondCardMark，用来决定是否开启卡表更新的条件判断。开启会增加一次额外判断的开销，但能够避免伪共享问题，两者各有性能损耗，是否打开要根据应用实际运行情况来进行权衡。

### 2.3.6 并发的可达性分析

可达性分析算法理论上要求全过程都基于一个能保障一致性快照中才能分析，这意味着必须要全程冻结用户线程的运行。在根节点枚举这一步上由于GC ROOT在Java堆中算少数，且在优化（OopMap）下这一停顿非常短暂，但是在继续往下遍历对象时停顿时间就与Java堆容量成正比了。

向解决或降低用户线程的停顿，就必须明白为什么必须在一个能保障一致性的快照上进行对象图的遍历。我们以三色标记法进行辅助推导：

- 白色：表示对象尚未被垃圾收集器访问过。在分析刚开始所有的对象都是白色的，在分析结束时还是白色的表示不可达
- 黑色：表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过。黑色代表安全存活的，如果有其他引用指向了黑色对象无需重新扫描一遍。黑色对象不能直接指向白色
- 灰色表示对象已经被垃圾收集器访问国，但这个对象至少存在一个引用还没有被扫描过。

> 在分析的过程中如果用户线程是冻结的，那么不会有任何问题。如果是并发运行的，收集器在对象图中标色，同时用户线程在修改引用关系，也就是对象图的结构，这样的话会出现两种情况：
>
> 一是把原本死亡的对象标为存活。这样可以容忍，因为这会产生一点逃过清理的垃圾，下一次清理掉就行。
>
> 第二种是把该存活的对象标为死亡，这样就十分致命，因为会发生错误。

下面我们来推导（推导过程与原书一致，仅图片不同，图片是我自己按照原书步骤画的）把原本死亡的对象标为存活这种致命错误出现的原因：

1. 初始状态，只有GC ROOTS是黑色的，注意箭头引用是有向的，只有被黑色对象引用才会存活，没有黑色对象引用，再如何引用其他对象最后也会被清理。

   ![image-20220809133816631](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809133816631.png)

2. 扫描过程中，以灰色为波峰的波浪从黑向白进行推进，灰色对象是黑白对象的分界线。

   ![image-20220809133950835](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809133950835.png)

3. 扫描完成后，黑色是存活的，白色是即将回收的对象。

   ![image-20220809134049131](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809134049131.png)

   **上面是正常流程，下面是if线**

4. 如果用户线程在标记进行时并发的修改了引用关系，比如正在扫描的灰色对象的一个引用被切断了，同时又与扫描过的黑色对象建立了引用关系。

   ![image-20220809134622433](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809134622433.png)

   又比如切断后重新被黑色对象引用的是原有引用链的一部分，这就导致被黑色对象引用的对象依旧是白色，这个对象就会被清理（因为黑色对象不会重新扫描）。

   ![image-20220809134704970](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220809134704970.png)

Wilson在1994年证明了当且仅当以下两个条件满足时，会产生对象消失问题（本是黑色被误标成白色）：

- 赋值器插入了一条或多条黑色到白色对象的新引用
- 赋值器删除了全部从灰色到白色的直接或间接引用

因此，如果想解决并发扫描的对象消失问题，只需要破坏这两个条件的任一个即可。因此产生了两种解决方案：

- 增量更新：增量更新破坏的是第一个条件，当黑色对象插入新的指向白色对象的引用关系时，就将这个新插入的引用记录下来，等并发扫描结束后，再将这些记录过的引用关系的黑色对象为根重新扫描一遍。简而言之就是黑色对象一旦插入，它就变成灰色对象了。
- 原始快照：原始快照破坏的是第二个条件，当灰色对象要删除指向白色对象的引用关系时，就将要删除的引用记录下来，在并发扫描结束后，再将这些记录过的引用关系中的灰色对象为根重新扫描一遍。简单来说就是无论关系删除与否，都会按照刚刚开始扫描那一刻的对象图快照来进行搜索。

无论是对引用关系记录的插入还是删除，虚拟机的记录操作都是通过写屏障实现的。在HotSpot虚拟机中，增量更新和原始快照这两种解决方案都有实际应用，譬如，CMS是基于增量更新来做并发标记的，G1、Shenandoah则是用原始快照来实现。

## 2.4 经典垃圾收集器

> 本小节2.4节的图片均来自深入理解Java虚拟机第三版

### 2.4.1 Serial收集器

Serial收集器是最基础历史最悠久的收集器，在JDK1.3.1之前是HotSpot新生代收集器的唯一选择。这个收集器在工作时只使用一个处理器或一个线程完成垃圾收集工作，且在进行垃圾收集时必须暂停其他所有的工作线程，直到收集结束。由于Stop the World的恶劣体验，HotSpot虚拟机开发团队一直在为消除戳降低用户线程的停顿进行着努力。Serial/Serial Old收集器运行如下图：

![image-20220811084803410](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811084803410.png)

虽然Serialy有很多缺点，但是其也有优点，那就是简单高效，在客户端模式下的虚拟机是一个很好的选择。

使用方式：`-XX:+UseSerialGC `

### 2.4.2 ParNew收集器

ParNew收集器实质上是Serial收集器的多线程并行版本，除了同时使用多条线程进行垃圾收集之外，其余的行为包括Serial收集器可用的所有控制参数 、 收集算法、 Stop The World、 对象分配规则、 回收策略等都与Serial收集器完全一致，在实现上这两种收集器也共用了相当多的代码。工作过程如下：

![image-20220811085350191](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811085350191.png)

ParNew收集器除了支持多线程并行收集之外，其他与Serial收集器相比并没有太多创新之处，但它却是不少运行在服务端模式下的HotSpot虚拟机，尤其是JDK 7之前的遗留系统中首选的新生代收集器，其中有一个与功能、性能无关但其实很重要的原因是：除了Serial收集器外，目前只有它能与CMS收集器配合工作。也可以理解为，ParNew合并入CMS，成为它专门处理新生代的组成部分。ParNew收集器在单CPU服务器上的垃圾收集效率绝对不会比Serial收集器高；但是在多CPU服务器上，效果会明显比Serial好。

> 并行与并发：并行（Parallel）指多条垃圾收集线程之间的关系，说明同一时间有多条这样的线程在协同工作，通常默认此时用户线程处于等待状态。并发（Concurrent）是指同一时间垃圾收集线程与用户线程都在运行，但是程序的吞吐量会受到一定影响。

使用方式：`-XX:+UseParNewGC`

设置线程数: `XX:ParllGCThreads`

### 2.4.3 Parallel Scavenge收集器

这也是一个新生代收集器，也是基于标记复制算法实现的收集器，同时也是多线程并行收集器。它的特点是它的关注点与其他收集器不同，CMS等收集器是尽可能短的收集用户线程的停顿时间，而Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量。吞吐量就是用户代码运行时间比上处理器总消耗时间（用户代码运行时间+垃圾运行收集时间）的比值。

Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量：

- 控制最大垃圾收集停顿时间的-XX:MaxGCPauseMillis参数。此参数尽力保证内存回收时间不超过用户设定值。当然垃圾收集停顿时间以牺牲新生代大小和吞吐量的代价换取的。
- 直接设置吞吐量大小的-XX:GCTimeRatio参数。 此参数在(0-100)，开区间且为整数，如果设置为9%，那么最大垃圾收集时间就占总时间的10%（1/(1+9)），默认值为99。

还提供了一个垃圾收集的自适应的调节策略数-XX: +UseAdaptiveSizePolicy这是一个开关参数，当这个参数被激活之后，就不需要人工指定新生代的大小、Eden与Survivor区的比例、晋升老年代对象大小等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。

> Parallel Scavenge收集器架构中本身有PS MarkSweep收集器来进行老年代收集，并非直接调用Serial Old收集器，但是这个PS MarkSweep收集器与Serial Old的实现几乎是一样的

### 2.4.4 Serial Old收集器

Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。这个收集器的主要意义也是供客户端模式下的HotSpot虚拟机使用。如果在服务端模式下，它也可能有两种用途：一种是在JDK 5以及之前的版本中与Parallel Scavenge收集器搭配使用，另外一种就是作为CMS收集器发生失败时的后备预案，在并发收集发生Concurrent Mode Failure时使用。Serial Old收集器的工作过程如图所示：

![image-20220811084803410](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811084803410.png)

### 2.4.5 Parallel Old收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。这个收集器是直到JDK 6时才开始提供的，在此之前，新生代的Parallel Scavenge收集器一直处于相当尴尬的状态，原因是如果新生代选择了Parallel Scavenge收集器，老年代除了Serial Old（PS MarkSweep）收集器以外别无选择，其他表现良好的老年代收集器，如CMS无法与它配合工作。同样，由于单线程的老年代收集中无法充分利用服务器多处理器的并行处理能力，在老年代内存空间很大而且硬件规格比较高级的运行环境中，这种组合的总吞吐量甚至不一定比ParNew加CMS的组合来得优秀。直到Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的搭配组合，在注重吞吐量或者处理器资源较为稀缺的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器这个组合。Parallel Old收集器的工作过程如图所示：

![image-20220811093118832](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811093118832.png)

### 2.4.6 CMS收集器

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的Java应用集中在互联网网站或者基于浏览器的B/S系统的服务端上，这类应用通常都会较为关注服务的响应速度，希望系统停顿时间尽可能短，以给用户带来良好的交互体验。

这个垃圾收集器是基于标记清除算法实现的。它的运作过程分为四个步骤：

1. 初始标记
2. 并发标记
3. 重新标记
4. 并发清除

其中初始标记和重新标记仍然需要Stop The World，初始标记仅仅只是标记一下GC Roots能直接关联的对象，速度很快；并发标记是从GC Roots开始遍历整个对象图，这个过程耗时较长但不需要停顿用户线程；重新标记阶段就是为了修正并发标记期间变动的记录（通过增量更新的方式），这个阶段停顿时间比初始标记长一些，但远比并发标记阶段耗时短；最后就是并发清除阶段，清理掉已经死亡的对象，由于不需要移动存活对象所以此阶段也可以和用户线程并发。各阶段的运行过程如下：

![image-20220812091554240](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220812091554240.png)

看到这可能会有疑问，标记清除算法会产生空间碎片，那CMS该如何处理呢？为了解决这个问题CMS提供了一个`-XX: +UseCMS-CompactAtFullCollection`开关参数（默认是开启的，此参数在JDK9废弃）用于在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程，由于此过程必须移动存活对象，所以无法并发。虽然这样解决了空间碎片问题但是停顿时间又变长了，所以虚拟机设计者又提供了另外的参数`-XX: CMSFullGCBefore-Compaction`(此参数在JDK9废弃)，这个参数（默认为0，表示每次进入Full GC时都进行碎片整理）作用是要求CMS收集器在执行若干次不整理空间的Full GC后，下一次进入Full GC前会先进行碎片整理。

当然除了此问题，CMS还有一些缺点，比如它对处理器资源非常敏感，因为它要与用户线程并发，虽然不会停顿用户线程但是会降低吞吐量。CMS默认启动的回收线程数是`(处理器核心数量+3)/4`,也就是说处理器核心数在4个以上，并发回收垃圾时占用不超过25%的处理器运算资源，且会随着核心数增加而下降，但如果核心数小于四个就会导致用户程序执行速度忽然大幅下降。然后CMS收集器无法处理浮动垃圾可能会出现并发失败导致出现Stop the World的FullGC（在CMS的并发标记和并发清理阶 段，用户线程是还在继续运行的，程序在运行自然就还会伴随有新的垃圾对象不断产生，但这一部分垃圾对象是出现在标记过程结束以后，CMS无法在当次收集中处理掉它们，这一部分垃圾就称为“浮动垃圾”）同时因为要支持并发所以需要给用户线程留一部分空间，在JDK5默认68%就会被激活，到了JDK6CMS收集器的启动阈值就默认提升到92%。这个参数可以通过`-XX: CMSInitiatingOccupancyFraction`进行设置。此参数如果设置的太高，CMS运行时可能就没有足够的内存分配新对象，就会出现并发失败，这时虚拟机会冻结用户线程，临时启用Serial Old进行老年代垃圾回收。

### 2.4.7 Garbage First收集器

G1收集器是垃圾收集器发展历史上的里程碑式的成果，开创了面向局部收集的设计思路和基于Region的内存布局形式。G1是一款主要面向服务端引用的垃圾收集器。HotSpot开发团队最初赋予它的期望是未来可以替换掉JDK5中发布的CMS收集器，JDK9发布之后，G1宣告取代Parallel Scavenge+Parallel Old组合，成为服务端模式下的默认垃圾收集器如果用户在JDK9及以上版本的HotSpot虚拟机使用参数`-XX: +UseConcMarkSweepGC`来开启CMS收集器的话用户会收到一个警告信息，提示CMS未来将被废弃。

在G1出现之前，所有的收集器包括CMS在内，垃圾收集的目标范围要么是新生代(Minor GC)，要么是老年代(Major GC)，要么就是整个Java堆(Full GC)。而G1可以面向堆内存任何部分来组成回收集进行回收，这就是G1的Mixed GC模式。G1开创的基于Region的堆内存布局虽然也遵循分代收集理论，但是其堆内存的布局与其他收集器有非常明显的差异，即G1不在坚持固定大小的分代区域划分，而是把连续的Java堆划分为多个大小相等的独立区域，每一个Region都可以根据需要扮演新生代Eden、Survivor或者老年代空间。收集器能够对不同角色的Region采用不同的策略去处理。Region中还有一类特殊的Humongous区域，专门用来存储大对象，G1认为只要大小超过了一个Region容量的一半的对象即可判定为大对象。每个Region的大小可以通过`-XX G1HelpRegionSize`进行设置，取值范围为1MB-32MB，且应该为2的n次幂，而超过了整个Region容量的超级大对象将会被存放N个连续的Humongous Region中，G1大多数行为都将Humongous Region作为老年代的一部分来进行看待。如下图：

![image-20220812133435941](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220812133435941.png)

而且虽然G1保留着新生代和老年代的概念，但是它们均不固定，G1将Region作为单元回收的最小单元，每次回收Region大小的整数倍，这样避免了整个Java堆的全收集。具体来说就是让G1跟踪每个Region的垃圾的价值，然后在后台维护一个优先级列表，每次根据用户设置的收集停顿时间(-XX: MaxGCPauseMillis指定，默认200ms)优先处理回收价值大的Region。

> G1至少有以下问题需要解决：
>
> 1. 跨Region对象引用如何解决
>
>    我们已经了解了记忆集解决跨代引用的问题，但是G1的记忆集十分复杂，每个Region都需要维护记忆集。G1的记忆集本质上时哈希表，Key是别的Region的起始地址，Value是一个集合，里面存储的是卡表的索引号。这种双向卡表（即存我指向谁，又存谁指向我），通常消耗Java堆容量的10%-20%
>
> 2. 并发标记时如何与用户线程互不干扰
>
>    在CMS中使用的是增量更新算法，这里G1使用的是原始快照(SATB)实现的。同时在回收过程中程序运行肯定就会有新对象的创建，G1为每一个Region设计了两个名为TAMS的指针，把Region中的一部分空间划分出来用于并发回收过程中的新对象分配，新对象必须要在这两个指针位置以上。G1默认它们存活。与CMS类似，如果回收速度小于内存分配速度，G1也要Full GC，而产生Stop the World
>
> 3. 如何建立可靠的停顿预测模型
>
>    用户通过-XX：MaxGCPauseMillis参数指定的停顿时间只意味着垃圾收集发生之前的期望值。G1收集器的停顿预测模型是以衰减均值（Decaying Average）为理论基础来实现的，在垃圾收集过程中，G1收集器会记录每个Region的回收耗时、每个Region记忆集里的脏卡数量等各个可测量的步骤花费的成本，并分析得出平均值、标准偏差、置信度等统计信息。这里强调的“衰减平均值”是指它会比普通的平均值更容易受到新数据的影响，平均值代表整体平均状态，但衰减平均值更准确地代表“最近的”平均状态。换句话说，Region的统计状态越新越能决定其回收的价值。然后通过这些信息预测现在开始回收的话，由哪些Region组成回收集才可以在不超过期望停顿时间的约束下获得最高的收益

G1的运行过程大概可以分为以下几个阶段：

1. 初始标记：仅标记一下GCRoots能直接关联到的对象，并且修改TAMS指针的值，让下一阶段用户线程并发运行时，能正确地在可用的Region中分配对象。这个阶段需要停顿线程，但耗时很短，且是借用MinorGC时完成的，所以没有额外停顿。
2. 并发标记：从GC Root开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时较长，但可与用户程序并发执行。当对象图扫描完成以后，还要重新处理SATB记录下的在并发时有引用变动的对象。
3. 最终标记：对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留下来的最后那少量的SATB记录。
4. 筛选回收：负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集，然后把决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成的。

![image-20220815133001770](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220815133001770.png)

> 在原书中，周志明大佬是这样选择CMS和G1的：
>
> 按照笔者的实践经验，目前在小内存应用上CMS的表现大概率仍然要会优于G1，而在大内存应用上G1则大多能发挥其优势，这个优劣势的Java堆容量平衡点通常在6GB至8GB之间，当然，以上这些也仅是经验之谈，不同应用需要量体裁衣地实际测试才能得出最合适的结论，随着HotSpot的开发者对G1的不断优化，也会让对比结果继续向G1倾斜。

## 2.5 低延迟垃圾收集器

暂略，目前属于实现状态，这里暂略，后续进行补充。

### 2.5.1 Shenandoah收集器

### 2.5.2 ZGC收集器

## 2.6 选择合适的垃圾收集器



## 2.7 内存分配与回收策略

# 三、虚拟机性能检测、故障处理工具







