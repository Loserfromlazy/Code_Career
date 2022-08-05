# Java虚拟机篇一自动内存管理

> 参考资料：深入理解Java虚拟机周志明第三版以及互联网资源具体引用会在文中标注

# 一、Java内存区域

对于Java程序员来说，在虚拟机自动内存管理机制下，不再需要为每一个new操作配对delete/free代码，但是正由于Java将控制内存的权力交给虚拟机，一旦出现内存泄漏或溢出等问题，如果不了解虚拟机对内存的使用就会使排查问题变得艰难。

## 1.1 运行时数据区域

Java虚拟机在执行Java程序过程中会把它所管理的内存划分为不同的数据区域这些区域有各自的用途、建立和销毁的事件，有的区域会随着虚拟机的进程而一直存在，有的区域则是随着用户线程的启动和结束而建立和销毁，根据Java虚拟机规范，Java虚拟机内存管理包含以下几个运行时数据区域。

![image-20220803095308621](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220803095308621.png)

1. 程序计数器

   程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器，在Java虚拟机的概念模型中，字节码解释器工作时就是通过改变这个计数器来选取下一条需要执行的字节码指令，它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成。

   由于Java虚拟机的多线程是通过线程轮流切换、分配处理器执行时间实现的，在任何一个时刻，一个处理器（多核的话是一个内核）都会只执行一条线程中的指令。因此，为了线程切换后能恢复到正常的执行位置，每条线程都有一个独立的程序计数器，各线程之间互不影响独立存储，这类内存区域称为线程私有内存。

   如果线程正在执行一个Java方法，这个计数器记录的是正在执行的虚拟机字节码的地址，如果正在执行本地方法，这个计数器的值为空。**此内存区域是唯一一个没有规定OOM的区域。**

2. Java虚拟机栈

   Java虚拟机栈也是线程私有，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的线程内存模型，每个方法执行的时候，虚拟机栈会同步创建一个栈帧，用于存储局部变量表、动态链接、方法出口等信息，每一个方法都有执行完毕的过程，对应着一个栈帧在虚拟机栈中从入站到出站的过程。

   > 总有人将Java内存笼统的分为堆内存和栈内存，栈通常指的就是虚拟机栈或者更多情况下指的是虚拟机栈中的局部变量表。

   局部变量表存放了编译期可知的各种Java虚拟机基本数据类型、对象引用（reference类型，它并不等同于对象本身，可能是一个指向对象起始 地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress 类型（指向了一条字节码指令的地址）。

   这些数据类型在局部变量表中的存储空间以局部变量槽（Slot）来表示，其中64位长度的long和 double类型的数据会占用两个变量槽，其余的数据类型只占用一个。局部变量表所需的内存空间在编 译期间完成分配，当进入一个方法时，这个方法需要在栈帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。请读者注意，这里说的“大小”是指变量槽的数量， 虚拟机真正使用多大的内存空间来实现一 个变量槽，这是完全由具体的虚拟机实现自行决定的事情。

   在《Java虚拟机规范》中，对这个内存区域规定了两类异常状况：**如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果Java虚拟机栈容量可以动态扩展，当栈扩展时无法申请到足够的内存会抛出OutOfMemoryError异常。**

3. 本地方法栈

   本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别只是虚拟机 栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的本地（Native） 方法服务。

   Java虚拟机规范对本地方法栈中方法使用的语言、使用方式与数据结构并没有任何强制规定，因此具体的虚拟机可以根据需要自由实现它，甚至有的Java虚拟机（譬如Hot-Spot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。**与虚拟机栈一样，本地方法栈也会在栈深度溢出或者栈扩展失败时分别抛出StackOverflowError和OutOfMemoryError异常。**

4. Java堆

   对于Java应用程序来说，Java堆Java Heap是虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，Java世界里所有的对象实例都在这里分配内存。Java堆是垃圾收集器管理的内存区域，因此一些资料中它也被称作“GC堆”。

   如果从分配内存的角度看，所有线程共享的Java堆中可以划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB），以提升对象分配时的效率。不过无论从什么角度，无论如何划分，都不会改变Java堆中存储内容的共性，无论是哪个区域，存储的都只能是对象的实例，将Java堆细分的目的只是为了更好地回收内存，或者更快地分配内存。

   Java堆可以处于物理上不连续的内存空间中，但在逻辑上它应该 被视为连续的，这点就像我们用磁盘空间去存储文件一样，并不要求每个文件都连续存放。但对于大对象（典型的如数组对象），多数虚拟机实现出于实现简单、存储高效的考虑，很可能会要求连续的内存空间。Java堆既可以被实现成固定大小的，也可以是可扩展的，不过当前主流的Java虚拟机都是按照可扩展来实现的（通过参数-Xmx和-Xms设定）。**如果在Java堆中没有内存完成实例分配，并且堆也无法再扩展时，Java虚拟机将会抛出OutOfMemoryError异常。**

5. 方法区

   方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。虽然Java虚拟机规范中把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫作“非堆”（Non-Heap），目的是与Java堆区分开来。

   > 说到方法区，不得不提一下“永久代”这个概念，尤其是在JDK 8以前，许多Java程序员都习惯在HotSpot虚拟机上开发、部署程序，很多人都更愿意把方法区称呼为“永久代”），或将两者混为一谈。本质上这两者并不是等价的，因为仅仅是当时的HotSpot虚拟机设计团队选择把收集器的分代设计扩展至方法区，或者说使用永久代来实现方法区而已，这样使得HotSpot的垃圾收集器能够像管理Java堆一样管理这部分内存，省去专门为方法区编写内存管理代码的工作。但是对于其他虚拟机实现，譬如BEA JRockit、IBM J9等来说，是不存在永久代的概念的。
   >
   > 考虑到HotSpot未来的发展，在JDK 6的时候HotSpot开发团队就有放弃永久代，逐步改为采用本地内存（Native Memory）来实现方法区的计划了，到了JDK 7的HotSpot，已经把原本放在永久代的字符串常量池、静态变量等移出，而到了JDK 8，终于完全废弃了永久代的概念，改用与JRockit、J9一样在本地内存中实现的元空间（Meta-space）来代替，把JDK 7中永久代还剩余的内容（主要是类型信息）全部移到元空间中。

   根据Java虚拟机规范的规定，如果方法区无法满足新的内存分配需求时，将抛出 OutOfMemoryError异常。

6. 运行时常量池

   运行时常量池（Runtime Constant Pool）是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池表（Constant Pool Table），用于存放编译期生成的各种字面量与符号引用，这部分内容将在类加载后存放到方法区的运行时常量池中。Java虚拟机对于Class文件每一部分（自然也包括常量池）的格式都有严格规定，如每一个字节用于存储哪种数据都必须符合规范上的要求才会被虚拟机认可、加载和执行，但对于运行时常量池，Java虚拟机规范并没有做任何细节的要求，不同提供商实现的虚拟机可以按照自己的需要来实现这个内存区域，不过一般来说，除了保存Class文件中描述的符号引用外，还会把由符号引用翻译出来的直接引用也存储在运行时常量池中。

   **既然运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出OutOfMemoryError异常。**

7. 直接内存

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

这里由于深入理解Java虚拟机中使用了jdk6、7、8等jdk进行测试，这里安装比较麻烦因此暂时不进行实验。

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



### 2.2.2 标记清除算法

### 2.2.3 标记复制算法

### 2.2.4 标记整理算法



