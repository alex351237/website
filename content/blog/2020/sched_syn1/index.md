---
title: "内核中的调度与同步"
date: 2020-06-06T15:58:26+08:00
author: "马明慧整理"
keywords: ["内核调度","内核同步"]
categories : ["内核同步"]
banner : "img/blogimg/sched.jpg"
summary : "本章将为大家介绍内核中存在的各种任务调度机理以及它们之间的逻辑关系（这里将覆盖进程调度、推后执行、中断等概念），在此基础上向大家解释内核中需要同步保护的根本原因和保护方法。最后提供一个内核共享链表同步访问的例子，帮助大家理解内核编程中的同步问题。"
---

## 摘要

本章将为大家介绍内核中存在的各种任务调度机理以及它们之间的逻辑关系（这里将覆盖进程调度、推后执行、中断等概念），在此基础上向大家解释内核中需要同步保护的根本原因和保护方法。最后提供一个内核共享链表同步访问的例子，帮助大家理解内核编程中的同步问题。

## 内核任务调度与同步关系引言

对于从事应用程序开发的朋友来说，用户空间的任务调度与同步之间的关系相对简单，无需过多考虑需要同步的原因。这一是因为在用户空间中各个进程都拥有独立的运行空间，进程内部的数据对外不可见，所以各个进程即使并发执行也不会产生对数据访问的竞争。第二是因为用户空间与内核空间独立，所以用户进程不会与**内核任务**交错执行，因此用户进程不存在与内核任务并发的可能。以上两个原因使得用户同步仅仅需要在进程间通讯和多线程编程时需要考虑。

但是在内核空间中情况要复杂得多，需要考虑同步的原因大大增加了。这是因为内核空间中的共享数据对内核中的所有任务可见，所以当在内核中访问数据时，就必须考虑是否会有其他内核任务并发访问的可能、是否会产生竞争条件、是否需要对数据同步。而内核并发的“罪魁祸首”便是内核中复杂多变的任务调度——这里的任务调度包含所有可能引起内核任务更换的情况。

**并发，竞争和同步的概念，我们假定大家都有所了解，本文不再重申。下面一段描述了上述几个概念之间的大致关系，这种关系在内核中同样适用。**

**对于多线程程序的开发者来说，往往会利用多线程访问共享数据，避免繁琐的进程间通讯。但是多线程对共享数据的并发访问有可能产生竞争，使得数据处于不一致状态，所以需要一些同步方法来保护共享数据。多线程的并发执行是由于线程被抢占式的调度——一个线程在对共享数据访问期间（还未完成）被调度程序中断，将另一个线程投入运行——如果新被调度的线程也要对这个共享数据进行访问，就将产生竞争。为了避免竞争产生，需要使线程串行地访问共享数据 ，也就是说访问需要同步——在一方对数据访问结束后，另一方才能对同一数据进行访问。**

## 内核任务

这里所定义的内核任务是指内核中执行的一切活动对象，每个内核任务都拥有一个独立的程序计数器、栈和一组寄存器。更重要的是,它们都属于内核调度（这里的调度是广义上的，不要与进程调度混淆）对象，也就是说它们是可以在内核中交错执行的。

### 内核任务分类

内核任务包含“内核线程”、“系统调用”、“硬件中断”、“半底任务”等几类。下来我们就简要地讨论上述几类内核任务的特点。

#### 系统调用

​    系统调用是用户程序通过门机制来进入内核执行的内核例程，它运行在内核态，处于进程上下文中（进程上下文包括进程的堆栈等等环境），可以认为是代表用户进程的内核任务，因此具有用户态任务的特性，比如可以执行进程调度程序（**schedule()**）、可以睡眠、可以访问当前进程数据（通过**current**）。但它属于内核任务，所以在执行过程中不能被抢占（**2.6**内核前），只能自己放弃**cpu**（睡眠）时，系统才能可能重新调度别的任务。（有关系统调用部分请看《系统调用》一章）

#### 硬中断任务

硬中断是指那些由处理器以外的外设产生的中断，这些中断被处理器接收后交给内核中的中断处理程序处理。要注意的是：第一, 硬中断是异步产生的，中断发生后立刻得到处理，也就是说中断操作可以抢占内核中正在运行的代码。这点非常重要。第二，中断操作是发生在中断上下文中的（所谓中断上下文指的是和任何进程无关的上下文环境）。中断上下文中不可以使用进程相关的资源，也不能够进行调度或睡眠。因为调度会引起睡眠，但睡眠必须针对进程而言（睡眠其实是标记进程状态，然后把当前进程推入睡眠列队），而异步发生的中断处理程序根本不知道当前进程的任何信息，也不关心当前哪个进程在运行，它完全是个过客。（有关硬件中断部分请看《硬件中断》一章）

#### 下半底任务

半底的来历完全出自上面提到的硬中断的影响。硬件中断任务（处理程序）是一个快速、异步、简单地对硬件做出迅速响应并在最短时间内完成必要操作的中断处理程序。硬中断处理程序可以抢占内核任务并且执行时还会屏蔽同级中断或其它中断，因此中断处理必须要快、不能阻塞。这样一来对于一些要求处理过程比较复杂的任务就不合适在中断任务中一次处理。比如，网卡接收数据的过程中,首先网卡发送中断信号告诉**CPU**来取数据，然后系统从网卡中读取数据存入系统缓冲区中，再下来解析数据然后送入应用层。这些如果都让中断处理程序来处理显然过程太长，造成新来的中断丢失。因此**Linux**开发人员将这种任务分割为两个部分，一个叫上底，即中断处理程序，短平快地处理与硬件相关的操作（如从网卡读数据到系统缓存）；而把对时间要求相对宽松的任务（如解析数据的工作）放在另一个部分执行，这个部分就是我们这里要讲的下半底。

下半底是一种推后执行任务，它将某些不那么紧迫的任务推迟到系统更方便的时刻运行。内核中实现下半底的手段经过不断演化，目前已经从最原始的**BH(bottom thalf)**演生出**BH**、任务队列**（Task queues）**、软中断**（Softirq）**、**Tasklet**、工作队列**（Work queues）**（**2.6**内核中新出现的）。下面我们就介绍一下他们各自的特点。

​             

#### 软中断操作

软中断**（softirq）**不象硬中断那样是由硬件中断信号触发执行的，所以也不同于硬件中断那样时随时都能够被执行，笼统来讲,软中断会在内核处理任务完毕后返回用户级程序前得到处理机会。**具体的讲，有三个时刻它将被执行(do_softirq())：硬件中断操作完成后；系统调用返回时；内核调度程序中；（另外，内核线程ksoftirqd周期执行软中断）。**从中可以看出软中断会紧随硬中断处理(好象狐假虎威),所以抢占内核任务——至少在时钟中断后总有机会运行一次。还要记得软中断可以在不同处器上并发执行。

在有对称多处理器的机器上，那么两个任务就可以真正的在临界区中同时执行了，这种类型被称为**真并发**。相对而言在,单处理器上并发其实并不是真的同时发生，而是相互交错执行，是**伪并发。但它们都同样会造成竞争条件，而且也需要同样的保护。**

软中断是很底层的机制，一般除了在网络子系统和**SCSI**子系统这样对性能要求很高以及要求并发处理的时候，才会选择使用软中断。软中断虽然灵活性高和效率高，但是你自己必须处理复杂的同步处理（因为它可在多处理器上并发），所以通常都不直接使用，而是作为支持**Tasklet**和**BH**的根本。

需要说明的是,软中断的执行也处于中断上下文中，所以中断上下文对它的限制是和硬中断一样的。

#### Tasklet 

​    Tasklet和bottom half都是建立在软中断之上的两种延迟机制，其具体不同之处在于软中断是静态分配的，而且同类软中断可以并发地在几个CPU上运行；Tasklet可以动态分配，并且不同种类的Tasklets可以并发地在几个CPU上运行，但同类的tasklets 不可以；bottom half只能静态分配，实质上,下半部分是一个不能与其它下半部分并发执行的高优先级tasklet，即使它们类型不同，而且在不同CPU上运行。Tasklet可以理解为软中断的派生，所以它的调度时机与软中断一致。

​    对于内核中需要延迟执行的多数任务都可以利用tasklet来完成，由于同类tasklet本身已经进行了同步保护，所以使用tasklet相比软中断要简单得多，而且效率也不错。

#### bottom half

​    **是 BH**时最早的内核延迟方法，它原始、简单且容易控制，因为所有的BH处理程序都被严格地顺序执行——不允许任何两个BH处理程序同时并发执行，即使它们的类型不同也不可以，这样一来BH执行其间减少了许多同步保护。但是BH不得不被淘汰，因为它的“简便”牺牲了多处理器并发处理的高性能，等于一队人过独木桥那样速度受到牵制。

##### 任务队列

任务列队是**BH**的替代品，来自**BH**，所以它的属性也和**BH**相同。它的原意在于简化**BH**的操作接口，但它的随意性（数量随意、执行时机随意）却给系统带来了混乱，所以到今天已经被工作队列（在**2.6**内核中）所取代。

不过在**2.4**内核中任务队列还是被大量应用，尤其是调度队列、定时器队列和立即队列等三种任务队列（除了这三种系统已接管的特定任务队列外，你自己也可随心所欲的建立自己的任务队列，当然这时你要自己调度它）。调度队列的任务会在每次进程调度时得到处理，它是在进程上下文中处理的；定时器队列会在每次时钟滴答时得到处理；立即队列会在**中断**返回或调度时获得处理（所以处理最快），他们都是在中断上下文中处理的。

这些任务队列在内核内由一个统一的内核线程调度，该线程名为**keventd**，进程号是**2（2.4.18）**。你可用**ps**命令查看到该进程。

#### 内核线程

内核线程可以理解成在内核中运行的特殊进程，它有自己的“进程上下文”（借用调用它的用户进程的上下文），所以同样被进程调度程序调度，也可以睡眠——它和用户进程属性何其相似，不同之处就在于内核线程运行于内核空间，可访问内核数据，运行期间不能被抢占。

传统的Unix系统把一些重要的任务委托给周期性执行的进程，这些任务包括刷新磁盘高速缓存，交换出不用的页面，维护网络链接等等。事实上，以严格线性的方式执行这些任务的确效率不高，如果把他们放在后台调度，不管是对它们的函数还是对终端用户进程都能得到较好地响应。因为一些系统进程只运行在内核态，现代操作系统把它们的函数委托给内核线程（Kernel   Thread），内核线程不受不必要的用户态上下文的拖累。

## 内核中的同步

内核只要存在任务交错执行，就必然会存在对共享数据的并发问题，也就必然存在对数据的保护。而内核中任务交错执行的原因归根结底还是由于内核任务调度造成的。我们下面归纳一下内核中同步的原因。

### 同步原因

- 中断——中断几乎可以在任何时刻异步发生，也就可能随时打断当前正在执行的代码。
- 睡眠及与用户空间的同步——在内核执行的进程可能会睡眠，这就会唤醒调度程序，从而导致调度一个新的用户进程执行。
- 对称多处理——两个或多个处理器可以同时执行代码。
- 内核抢占——因为内核具有抢占性，所以内核中的任务可能会被另一任务抢占（在2.6内核引进的新力）。             

**后两种情况大大增加了内核任务并发执行的可能性，使得并发随时随刻都有可能发生，而且不可清晰预见，规律难寻。**

### 内核任务之间的并发关系

上述内核任务很多情况是可以交错执行的——记住，一个下半部实际上可能在任何时候执行，所以很有可能产生竞争（都要访问同一个数据结构时，就产生了竞争）。下面分析这些内核任务之间有哪些可能的并发行为。

可以抽象出，程序（用户态和内核态一样）并发执行的总原因无非是正在运行中的程序被其它程序*抢占*，所以我们必须看看内核任务之间的抢占关系：

- 中断处理程序可以抢占内核中的所有程序（当没有锁保护时），包括软中断，**tasklet，bottom half**和系统的调用、内核线程，甚至也包括硬中断处理程序。也就是说中断处理程序可以和这些所有的内核任务并发执行，如果被抢占的程序和中断处理程序都要访问同一个资源，就必然有可能产生竞争。
- 中断处理程序可以抢占内核中的所有程序（当没有锁保护时），包括软中断，**tasklet，bottom half**和系统的调用、内核线程，甚至也包括硬中断处理程序。也就是说中断处理程序可以和这些所有的内核任务并发执行，如果被抢占的程序和中断处理程序都要访问同一个资源，就必然有可能产生竞争。
- 软件中断也可以抢占内核中的所有任务，所以内核代码（比如，系统调用、内核线程等）中有数据和软中断共享，就会有竞争——除此外硬件中断处理程序也有可能被软中断打断，条件是硬中断被其它硬中断打断，软中断随即便获得了执行机会，因为软中断是跟在硬中断后执行的。此外要注意的是，软中断即使是同种类型的也可以并发地运行在不同处理器上，所以它们之间共享数据都会产生竞争。（如果在同一个处理器上,软中断之间是不能相互抢占的）。
-  同类的tasklet不可能同时运行，所以对于同类**tasklet**之间是串行运行的，他们不会产生并发；但两个不同种类的**tasklet**有可能在不同处理器上并发运行，如果之间有数据共享就会产生竞争（在同一个处理器上运行的**tasklet**不发生相互抢占的情况）。
- **Bottom half**无论是否是同类的，即使在不同处理器上也都不能并发执行，它是绝对串行化的，所以它们之间永远不能产生竞争。任务列队属性基本同**BH**。
- 系统调用和内核线程这种运行在进程上下文中的内核任务可能和各种内核任务并发，除了上面提到的中断（软，硬）来抢占它而产生并发外，它也有可能自发性地主动睡眠（比如在一些阻塞性的操作中），放弃处理器，重新调度其它任务，所以系统调用和内核线程除会与软硬中断（半底等）发生竞争，也会与其他（包括自己）系统调用与内核线程发生竞争。我们尤其要注意这种情况。        

**注意：tasklet和bottom   half是建立在软中断之上的，所以它们也都遵从软中断的调度规则——都可以打断进程上下文中的内核代码（系统调用），都可被硬中断打断——这些情况下都可能产生并发。**

## **内核同步措施**

为了避免并发，防止竞争。内核提供了一组同步方法来提供对共享数据的保护。 我们的重点不是介绍这些方法的详细用法，而是强调为什么使用这些方法和它们之间的差别。

**Linux**使用的同步机制可以说从**2.0**到**2.6**以来不断发展完善。从最初的原子操作，到后来的信号量，从大内核锁到今天的自旋锁。这些同步机制的发展伴随    **Linux**从单处理器到对称多处理器的过度；伴随着从非抢占内核到抢占内核的过度。锁机制越来越有效，也越来越复杂。

​    目前来说内核中原子操作多用来做计数使用，其它情况最常用的是两重锁以及它们的变种，一个是自旋锁，另一个是信号量。我们下面就来着重介绍一下这两种锁机制。

### 自旋锁

自旋锁是专为防止多处理器并发而引入的一种锁，它在内核中大量应用于中断处理等部分（对于单处理器来说，防止中断处理中的并发可简单采用关闭中断的方式，不需要自旋锁）。

自旋锁最多只能被一个内核任务持有，如果一个内核任务试图请求一个已被争用（已经被持有）的自旋锁，那么这个任务就会一直进行忙循环——旋转——等待锁重新可用。要是锁未被争用，请求它的内核任务便能立刻得到它并且继续进行。自旋锁可以在任何时刻防止多于一个的内核任务同时进入临界区，因此这种锁可有效地避免多处理器上并发运行的内核任务竞争共享资源。

事实上，自旋锁的初衷就是：在短期间内进行轻量级的锁定。一个被争用的自旋锁使得请求它的线程在等待锁重新可用的期间进行自旋（特别浪费处理器时间），所以自旋锁不应该被持有时间过长。如果需要长时间锁定的话, 最好使用信号量。

自旋锁的基本形式如下：

```c
spin_lock(&mr_lock);

/*临界区*/

spin_unlock(&mr_lock);
```

因为自旋锁在同一时刻只能被最多一个内核任务持有，所以一个时刻只有一个线程允许存在于临界区中。这点很好地满足了对称多处理机器需要的锁定服务。在单处理器上，自旋锁仅仅当作一个设置内核抢占的开关。如果内核抢占也不存在，那么自旋锁会在编译时被完全剔除出内核。

自旋锁在内核中有许多变种，如对**bottom half** 而言，可以使用**spin_lock_bh()**用来获得特定锁并且关闭半底执行。相反的操作由**spin_unlock_bh()**来执行；如果临界区的访问逻辑可以被清晰的分为读和写这种模式，那么可以使用读者/写者自旋锁，调用形式为：

**读者的代码路径：**

```c
read_lock(&mr_rwlock);

/*只读临界区*/

read_unlock(&mr_rwlock);
```

*写者的代码路径：*

```c
write_lock(&mr_rwlock);

/*读写临界区*/

write_unlock(&mr_rwlock);
```

​        简单的说，自旋锁在内核中主要用来防止多处理器中并发访问临界区，防止内核抢占造成的竞争。另外自旋锁不允许任务睡眠（持有自旋锁的任务睡眠会造成自死锁——因为睡眠有可能造成持有锁的内核任务被重新调度，而再次申请自己已持有的锁），它能够在中断上下文中使用。

​    **死锁：假设有一个或多个内核任务和一个或多个资源，每个内核都在等待其中的一个资源，但所有的资源都已经被占用了。这便会发生所有内核任务都在相互等待，但它们永远不会释放已经占有的资源，于是任何内核任务都无法获得所需要的资源，无法继续运行，这便意味着死锁发生了。自死琐是说自己占有了某个资源，然后自己又申请自己已占有的资源，显然不可能再获得该资源，因此就自缚手脚了。**

### 信号量 

**Linux**中的信号量是一种睡眠锁。如果有一个任务试图获得一个已被持有的信号量时，信号量会将其推入等待队列，然后让其睡眠。这时处理器获得自由去执行其它代码。当持有信号量的进程将信号量释放后，在等待队列中的一个任务将被唤醒，从而便可以获得这个信号量。

信号量的睡眠特性，使得信号量适用于锁会被长时间持有的情况；只能在进程上下文中使用，因为中断上下文中是不能被调度的；另外当代码持有信号量时，不可以再持有自旋锁。

信号量基本使用形式为：

```c
static DECLARE_MUTEX(mr_sem);//声明互斥信号量
…
if(down_interruptible(&mr_sem))
    
/*可被中断的睡眠，当信号来到，睡眠的任务被唤醒 */
    
/*临界区…*/
    
up(&mr_sem);
```

同自旋锁一样，信号量在内核中也有许多变种，比如读者－写者信号量等，这里不再做介绍了。

 **信号量和自旋锁区别**

虽然听起来两者之间的使用条件复杂，其实在实际使用中信号量和自旋锁并不易混淆。注意以下原则。

如果代码需要睡眠——这往往是发生在和用户空间同步时——使用信号量是唯一的选择。由于不受睡眠的限制，使用信号量通常来说更加简单一些。如果需要在自旋锁和信号量中作选择，应该取决于锁被持有的时间长短。理想情况是所有的锁都应该尽可能短的被持有，但是如果锁的持有时间较长的话，使用信号量是更好的选择。另外，信号量不同于自旋锁，它不会关闭内核抢占，所以持有信号量的代码可以被抢占。这意味者信号量不会对影响调度反应时间带来负面影响。

**自旋锁对信号量**

**―――――――――――――――――――――――――――――――**

​    **需求                                               建议的加锁方法**

低开销加锁                                    优先使用自旋锁

短期锁定                                       优先使用自旋锁

长期加锁                                             优先使用信号量

中断上下文中加锁                               使用自旋锁

持有锁是需要睡眠、调度                     使用信号量

**―――――――――――――――――――――――――――――――**

引自 《**Linux**内核开发》

防止并发的方式除了上面提到的外还有很多，我们不详细介绍了。说了这么多,希望大家认识到，并发控制在内核编程中是个特别难缠的问题，要驾御它必须清楚地认识到内核中各种任务的调度时机与特点，并且在开发初期就应特别小心保护共享数据（一切共享数据、一切能被别人看到的数据都要注意保护），别等到开发完成才去亡羊补牢。

## 并发控制实例

我们下面给出一个多内核任务访问共享资源的具体例子，其中会用到上面提到的各种同步方法，希望能给大家一个形象的记忆。

该例子的具体场景描述如下。

我们主要的共享资源是链表**（mine）**，操作它的内核任务有三种：一个是100个内核线程**（sharelist）**，它们负责从表头将新节点**（struct my_struct）**插入链表。二是定时器任务**(qt_task)**，它负责每个时钟滴答时从链表头删除一个节点。三是系统调用（由**rmmod**命令调用的**share_exit**），它负责销毁链表并卸载模块。

我们利用模块**（sharelist.o）**实现上述场景。加载模块时会建立定时器任务列队，并将要执行的任务**(task.rounting=qt_task)**插入定时器队列**（tq_timer）**，然后反复调度执行（但别不停地执行）。与此同时利用系统中的**keventd**内核线程（它的目的是执行任务队列,由**schedule_task**激活，**PID=2**），创建100个内核线程(创建函数**kernel_thread**)执行插入链表的工作（由**sharelist**完成）——但当链表长度超过100时，则从链表尾删除节点。最后当你需要卸载模块时，调用**share_exit**函数销毁整个链表，并做一些诸如销毁我们建立的内核进程的收尾工作。

下面我们具体看看在程序中该如何保护我们的链表。上述场景中存在的内核并发包括——内核线程之间的并发、内核任务与定时器任务的并发。要知道内核线程执行在进程上下文中，而定时器任务属于下半部分，执行在中断上下文中。在这两部分交错执行中进行保护则需要采用自旋锁。我们例子中使用了**spin_lock_bh()**锁在内核线程的执行路径中对链表进行保护；在下半部分,由于任务队列是串行执行并且不能被内核任务或系统调用打断，所以不必加锁。另外在卸载模块时，删除链表中仍然存在系统调用与下半部分的并发可能，因此也需要按上述方式加锁。

​    除了对共享链表访问使用自旋锁以外，还有两个需要同步的地方，一是计数**（count）**，该变量属于原子类型，用于记录链表接点的id。另外一个是利用信号量同步内核创建线程，调度**keventd**后执行被堵塞住**(down)**，等内核线程实际启动后, 才可继续执行**（up）**。下面的图给出了任务的基本关系和对链表的操作。

![文本框: kernel_thread](http://wwww.kerneltravel.net/journal/vi/syn.files/image001.gif)![img](http://wwww.kerneltravel.net/journal/vi/syn.files/image002.gif)

### 结束

并发的发生随处都有，但是由它引起的错误可并非每次都有，因为并发过程中引起错误的地方往往就一两步，因此交错执行这一两步要靠“运气”，出错的几率有时很小。但是一旦发生后果都是灾难性的，比如宕机，破坏数据完整性等。所以我们对并发绝不能掉以轻心，必须拿出“把纸老虎当真老虎的”决心来对待一切内核代码中可能的并发，即便在单处理器上编程也需要考虑到移植到多处理器的情况，总之一切都要谨慎小心。



实例代码见 [syn.tar](http://wwww.kerneltravel.net/journal/vi/syn.tar)

