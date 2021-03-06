---
title: 《计算机导论与javascript编程》读书总结
date: 2017-11-03 10:23:24
tags: ["计算机基础"]
summary: 主要记述读完这本书以后学到的一些内容，侧重于计算机基础，javascript暂不涉及。
category: "读书"
---

最近立了个flag，要把计算机基础，网络基础，数据算法这部分的内容学起来，以便更好地了解计算机，弥补非科班出身的基础知识的不足。大致的规划应该是计算机导论，计算机系统 -> 计算机网络 -> 数据结构与算法 -> 数据库，先这么来吧。最近跳着读完了《计算机导论与javascript编程》，记录一下学到的内容。

-------------------

主要讲下面几个方面。
- 计算机发展的历史
- 计算机的组成
- 因特网和万维网的发展

--------------------

### 计算机历史

计算机发展至今，经过了至少四代的迭代。从最初的机械式计算机到现在以超大规模集成电路为基础的并行处理，计算机世界发生了翻天覆地的变化。

第一代计算机，主要是以ENIAC为代表的电子管计算机。它采用了18000真空电子管，每秒可以计算5000次加法运算。此时的计算机，只是能简单的计算数据，不能对数据进行存储；如果要执行不同的计算，需要手动改变电子管之间的连线；而且电子管极易发热，非常容易损坏，整台计算机耗电量惊人。尽管如此，ENIAC在当时仍具有重大意义。
后来，参与ENIAC的设计的约翰·冯·诺依曼提出了一种新的计算机体系结构。在这种体系结构中，计算机可以读入指令并存在存储器内，避免了重新连接导线进行编程的繁琐。冯·诺依曼体系中，计算机由存储器，控制器，运算器和输入输出设备组成，这种体系结构为现代计算机的结构奠定了基础，他本人也被成为计算机科学之父。

{% img /images/computer1.jpg [300] %}

第二代计算机，晶体管计算机代替电子管计算机。相比于电子管，晶体管更小巧，便宜，能量效率更高，这就使得在一个计算机内，能够使用更多的晶体管，具有更强大的计算能力。在这期间，高级编程语言被创造出来（如FORTRAN, LISP），开发人员可以不用再编写复杂的二进制程序进行计算，提高了编程效率，便利了开发。

第三代计算机，计算机采用了集成电路。随着技术的发展，晶体管可以被造得更小，可以被分层排列在同一圆片上连接起来构成简单的电路，这种圆片就是集成电路或者ic芯片，这样，更多的晶体管可以被放在集成电路中，计算能力更为强大。到20世纪70年代，随着大规模集成电路的发展，用于计算机的微处理器上市，进一步提高了计算机的计算能力和速度。

第四代计算机，超大规模集成电路用于计算机。随着超大规模集成电路的发展，微处理器能够被大量生产出来，生产成本不断降低，使得个人计算机迅速发展和普及。与此同时，利用高级编程语言开发的商用软件和操作系统也开始出现并不断发展。

第五代计算机，并行处理和网络。现在，一个计算机处理器芯片上可以放置多个微处理器，多个处理器可以分散计算量，提高计算效率和速度。如果同时运行多个任务，多个处理器可以并行处理这些任务，减少了单处理器下执行多任务需要排队等待的情况，我们可以用计算机一边听音乐，一边浏览网页，一边打游戏，这些都得益于处理器的并行处理。同时，随着万维网和因特网的发展，我们可以通过它们与世界各地的人共享资源，互发邮件，进行虚拟通信等。

--------------------

### 计算机的组成

根据冯·诺依曼体系，计算机主要由处理器，输入输出设备和存储器。
##### 处理器也就是cpu，包括控制器，算术逻辑单元和寄存器。
控制器主要控制算数逻辑单元读取寄存器中的指令并将输出存在寄存器中。
算术逻辑单元(ALU)主要就是从寄存器中读取指令，执行指令。
寄存器通过总线取得主存储器中的数据，然后由ALU处理后存入寄存器，再通过总线将计算后的数据传到主存储器中。

{% img /images/cpu.jpg [200] %}

##### 存储器指各种各样的存储设备，包括主存储器和辅存储器。
- 主存储器包括高速缓存和随机存储器，也就是我们所说的RAM(内存)和Cache。主存储器主要是通过高速电路来传输数据的，所以一旦发生停电，在主存储器中的数据会丢失，这也就解释了为什么我们有时候在编辑word文档时如果忘记保存突然关机时之前编辑的内容再打开时就不存在的原因，在编辑word文档时，word文档的编辑的内容是保存在主存储器中的，保存后编辑的内容会从主存储器传输并保存在辅助存储器（磁盘，u盘等）中。不过现在一些做的比较好的软件通常隔一段时间会固定保存内容，防止突然断电时文档内容丢失。

- 辅助存储器包括硬盘，CD, DVD和U盘等。机械硬盘是将位作为磁化点和非磁化点来存储信息的。读取信息时，通过硬盘里的电机带动磁盘转动，磁头能探测到磁化点并翻译成位实现数据读取。

##### 输出输出(I/O)设备
输入输出设备主要包括鼠标，键盘，触摸板，麦克风，摄像头，显示器，音响，打印机等设备。通过输入设备输入信息，经过处理器读取处理后，输出到输出设备上，由此大大方便了人机交互，也使计算机的功能更加强大。

除了上面提到的硬件设备，计算机还包括软件。软件主要包括操作系统，应用软件。

操作系统比较重要的部分是内核，文件系统。
内核管理cpu的运作，控制数据和指令从内存中加载并由cpu访问；现在的处理器大部分都是多任务的（也就是俗称的多核心处理器），内核也负责不同任务之间的资源调用，共享cpu的资源；协调其它硬件组件，允许软件访问内存并和I/O设备交互。

文件系统管理计算机的内存，通过文件和目录组织存储。目录下可以放置目录或文件表明它们之间的位置和层级关系。

应用软件是专门处理某些特定任务的软件。比如浏览器是提供网页浏览的软件；adobe photoshop是专门进行图像处理制作的软件；应用软件由特定的功能以满足人们日常的工作生活的需要。


----------------------

### 万维网和因特网

平时，我们把网络叫做Internet，而Internet就源于因特网。
因特网属于硬件层面的，是指通过电缆相互连接的计算机组成的网络。因特网起源于20世纪60年代末，在之后的一段时间内缓慢发展，主要用于学术研究和军事行动。随着越来越多的计算机加入到因特网，因特网越来越大，不同地方的计算机相互连接，共享资料和资源。因特网采用了分布式网络和包交换技术。
为什么采用分布式网络呢？
在网络中，相互连接的计算机网络中，如果有一台位于干线上的计算机连接出现故障，不能传递数据，这时候分布式网络可以通过其它路径来把数据传送到目标机器，如果采用集中式网络，主干网络上出现问题，将会导致相当一部分计算机不能对外发送数据和与其它干线上的计算机进行通信。下面是分布式与集中式网络的示意图：

{% img /images/net.jpg [200] %}

在传输数据时，如果数据量非常大，会阻塞其它数据的传递，所以需要用到包交换技术。对于大数据，在发送时将其拆分成几个小的数据包，并做上排序标记，然后通过分布式网络到达目标机器，根据之前的标记重新进行排序组合成完整的数据。

##### 因特网协议
由于不同计算机操作系统，处理器，所处的位置，所用的语言可能不同，而且因特网上的计算机有很多，怎样实现数据的精确传递和数据的可达性，保证数据到达请求的计算机那里而不是其它计算机，数据在传递时必须要有一定的标示，需要遵循统一的标准，这就是TCP/IP协议。

在TCP/IP协议中，在因特网上的每一台计算机都被分配到一个独一无二的IP地址。在数据发送时，TCP控制数据包的分包以及到达目的地时的组包方式；IP则对包进行标注，标注了目标机器的IP和发送方的IP，控制包到达接受者的路径。


万维网是文档在其中通过因特网实现无缝链接的一个多媒体环境。后来，随着web服务器和浏览器的出现，万维网获得快速发展。通过浏览器，我们可以访问服务器上的web文档，音频，视频等资源。起初浏览器是收费的，后来微软推出了免费的internet explore浏览器，逐渐击败了netscape，占据了浏览器市场，之后，又出现了mozlia,safari,chrome等浏览器，用户的选择更加丰富。
如今，我们可以通过浏览器听音乐，看视频，浏览网页，都得益于万维网的发展。

##### 万维网协议
万维网协议指http协议。我们通过在浏览器地址栏中输入网址，向服务器请求所需要的资源，服务器接收到请求后，定位资源位置，通过消息将资源发送回去。决定浏览器和服务器交换的消息采用什么格式的协议就是http协议。

今天，通过万维网，我们可以通过基础的html知识，做一个自己的网页，放在服务器上，与他人共享自己的页面了。

-------------------

除了上边提到的这些，这本书里还讲述了算法，基础的门电路，数据存储与表示等，以后有机会在介绍吧......
