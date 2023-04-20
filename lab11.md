# Network

## ethernet（以太网地址，MAC地址）

让我从最底层开始，我们先来看一下一个以太网packet的结构是什么。当两个主机非常靠近时，或许是通过相同的线缆连接，或许连接在同一个wifi网络，或许连接到同一个以太网交换机。当局域网中的两个主机彼此间要通信时，最底层的协议是以太网协议。你可以认为Host1通过以太网将Frame发送给Host2。Frame是以太网中用来描述packet的单词，本质上这就是两个主机在以太网上传输的一个个的数据Byte。以太网协议会在Frame中放入足够的信息让主机能够识别彼此，并且识别这是不是发送给自己的Frame。每个以太网packet在最开始都有一个Header，其中包含了3个数据。Header之后才是payload数据。Header中的3个数据是：目的以太网地址，源以太网地址，以及packet的类型。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNzsFhzcgT0Lqe-M1YG%2F-MO3z5nk1S_cjrsBD1f7%2Fimage.png)

每一个以太网地址都是48bit的数字，这个数字唯一识别了一个网卡。packet的类型会告诉接收端的主机该如何处理这个packet。接收端主机侧更高层级的网络协议会按照packet的类型检查并处理以太网packet中的payload。

整个以太网packet，包括了48bit+48bit的以太网地址，16bit的类型，以及任意长度的payload这些都是通过线路传输。除此之外，虽然对于软件来说是不可见的，但是在packet的开头还有被硬件识别的表明packet起始的数据（注，Preamble + SFD），在packet的结束位置还有几个bit表明packet的结束（注，FCS）。packet的开头和结束的标志不会被系统内核所看到，其他的部分会从网卡送到系统内核。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNzsFhzcgT0Lqe-M1YG%2F-MO40ML0u3lTAh70m_1u%2Fimage.png)

如果你们查看了这门课程的最后一个lab，你们可以发现我们提供的代码里面包括了一些新的文件，其中包括了kernel/net.h，这个文件中包含了大量不同网络协议的packet header的定义。上图中的代码包含了以太网协议的定义。我们提供的代码使用了这里结构体的定义来解析收到的以太网packet，进而获得目的地址和类型值（注，实际中只需要对收到的raw data指针强制类型转换成结构体指针就可以完成解析）。

> 学生提问：硬件用来识别以太网packet的开头和结束的标志是不是类似于lab中的End of Packets？
>
> Robert教授：并不是的，EOP是帮助驱动和网卡之间通信的机制。这里的开头和结束的标志是在线缆中传输的电信号或者光信号，这些标志位通常在一个packet中是不可能出现的。以结束的FCS为例，它的值通常是packet header和payload的校验和，可以用来判断packet是否合法。

有关以太网48bit地址，是为了给每一个制造出来的网卡分配一个唯一的ID，所以这里有大量的可用数字。这里48bit地址中，前24bit表示的是制造商，每个网卡制造商都有自己唯一的编号，并且会出现在前24bit中。后24bit是由网卡制造商提供的任意唯一数字，通常网卡制造商是递增的分配数字。所以，如果你从一个网卡制造商买了一批网卡，每个网卡都会被写入属于自己的地址，并且如果你查看这些地址，你可以发现，这批网卡的高24bit是一样的，而低24bit极有可能是一些连续的数字。

虽然以太网地址是唯一的，但是出了局域网，它们对于定位目的主机的位置是没有帮助的。如果网络通信的目的主机在同一个局域网，那么目的主机会监听发给自己的地址的packet。但是如果网络通信发生在两个国家的主机之间，你需要使用一个不同的寻址方法，这就是IP地址的作用。

在实际中，你可以使用tcpdump来查看以太网packet。这将会是lab的一部分。下图是tcpdump的一个输出：

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO40Vq8POBpsayQpuQp%2F-MO8zlyQREcVbqVrEmiE%2Fimage.png)

tcpdump输出了很多信息，其中包括：

- 

  接收packet的时间

- 

  第一行的剩下部分是可读的packet的数据

- 

  接下来的3行是收到packet的16进制数

如果按照前面以太网header的格式，可以发现packet中：

- 

  前48bit是一个广播地址，0xffffffffffff。广播地址是指packet需要发送给局域网中的所有主机。

- 

  之后的48bit是发送主机的以太网地址，我们并不能从这个地址发现什么，实际上这个地址是运行在QEMU下的XV6生成的地址，所以地址中的前24bit并不是网卡制造商的编号，而是QEMU编造的地址。

- 

  接下来的16bit是以太网packet的类型，这里的类型是0x0806，对应的协议是ARP。

- 

  剩下的部分是ARP packet的payload。

## ARP

下一个与以太网通信相关的协议是ARP。在以太网层面，每个主机都有一个以太网地址。但是为了能在互联网上通信，你需要有32bit的IP地址。为什么需要IP地址呢？因为IP地址有额外的含义。IP地址的高位bit包含了在整个互联网中，这个packet的目的地在哪。所以IP地址的高位bit对应的是网络号，虽然实际上要更复杂一些，但是你可以认为互联网上的每一个网络都有一个唯一的网络号。路由器会检查IP地址的高bit位，并决定将这个packet转发给互联网上的哪个路由器。IP地址的低bit位代表了在局域网中特定的主机。当一个经过互联网转发的packet到达了局域以太网，我们需要从32bit的IP地址，找到对应主机的48bit以太网地址。这里是通过一个动态解析协议完成的，也就是Address Resolution Protocol，ARP协议。

当一个packet到达路由器并且需要转发给同一个以太网中的另一个主机，或者一个主机将packet发送给同一个以太网中的另一个主机时，发送方首先会在局域网中广播一个ARP packet，来表示任何拥有了这个32bit的IP地址的主机，请将你的48bit以太网地址返回过来。如果相应的主机存在并且开机了，它会向发送方发送一个ARP response packet。

下图是一个ARP packet的格式：

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO40Vq8POBpsayQpuQp%2F-MO95Pzg6uAEFzE35KkY%2Fimage.png)

它会出现在一个以太网packet的payload中。所以你们看到的将会是这样的结构：首先是以太网header，它包含了48bit的目的以太网地址，48bit的源以太网地址，16bit的类型；之后的以太网的payload会是ARP packet，包含了上图的内容。

接收到packet的主机通过查看以太网header中的16bit类型可以知道这是一个ARP packet。在ARP中类型值是0x0806。通过识别类型，接收到packet的主机就知道可以将这个packet发送给ARP协议处理代码。

有关ARP packet的内容，包含了不少信息，但是基本上就是在说，现在有一个IP地址，我想将它转换成以太网地址，如果你拥有这个IP地址，请响应我。

同样的，我们也可以通过tcpdump来查看这些packet。在网络的lab中，XV6会在QEMU模拟的环境下发送IP packet。所以你们可以看到在XV6和其他主机之间有ARP的交互。下图中第一个packet是我的主机想要知道XV6主机的以太网地址，第二个packet是XV6在收到了第一个packet之后，并意识到自己是IP地址的拥有者，然后返回response。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO40Vq8POBpsayQpuQp%2F-MO97sCnlq2lUQVrRkzq%2Fimage.png)

tcpdump能够解析出ARP packet，并将数据打印在第一行。对应ARP packet的格式，在第一个packet中，10.0.2.2是SIP，10.0.2.15是DIP。在第二个packet中，52:54:00:12:34:56对应SHA。

同时，我们也可以自己分析packet的原始数据。对于第一个packet：

- 

  前14个字节是以太网header，包括了48bit目的以太网地址，48bit源以太网地址，16bit类型。

- 

  从后往前看，倒数4个字节是TIP，也就是发送方想要找出对应以太网地址的IP地址。每个字节对应了IP地址的一块，所以0a00 020f对应了IP地址10.0.2.15。

- 

  再向前数6个字节，是THA，也就是目的地的以太网地址，现在还不知道所以是全0。

- 

  再向前数4个字节是SIP，也就是发送方的IP地址，0a000202对应了IP地址10.0.2.2。

- 

  再向前数6个字节是SHA，也就是发送方的以太网地址。

- 

  剩下的8个字节表明了我们感兴趣的是以太网和IP地址格式。

第二个packet是第一个packet的响应。

> 学生提问：ethernet header中已经包括了发送方的以太网地址，为什么ARP packet里面还要包含发送方的以太网地址？
>
> Robert教授：我并不清楚为什么ARP packet里面包含了这些数据，我认为如果你想的话是可以精简一下ARP packet。或许可以这么理解，ARP协议被设计成也可以用在其他非以太网的网络中，所以它被设计成独立且不依赖其他信息，所以ARP packet中包含了以太网地址。现在我们是在以太网中发送ARP packet，以太网packet也包含了以太网地址，所以，如果在以太网上运行ARP，这些信息是冗余的。但是如果在其他的网络上运行ARP，你或许需要这些信息，因为其他网络的packet中并没有包含以太网地址。
>
> 学生提问：tcpdump中原始数据的右侧是什么内容？
>
> Robert教授：这些是原始数据对应的ASCII码，“.”对应了一个字节并没有相应的ASCII码，0x52对应了R，0x55对应了U。当我们发送的packet包含了ASCII字符时，这里的信息会更加有趣。

我希望你们在刚刚的讨论中注意到这一点，网络协议和网络协议header是嵌套的。我们刚刚看到的是一个packet拥有了ethernet header和ethernet payload。在ethernet payload中，首先出现的是ARP header，对于ARP来说并没有的payload。但是在ethernet packet中还可以包含其他更复杂的结构，比如说ethernet payload中包含一个IP packet，IP packet中又包含了一个UDP packet，所以IP header之后是UDP header。如果在UDP中包含另一个协议，那么UDP payload中又可能包含其他的packet，例如DNS packet。所以发送packet的主机会按照这样的方式构建packet：DNS相关软件想要在UDP协议之上构建一个packet；UDP相关软件会将UDP header挂在DNS packet之前，并在IP协议之上构建另一个packet；IP相关的软件会将IP heade挂在UDP packet之前；最后Ethernet相关的软件会将Ethernet header挂在IP header之前。所以整个packet是在发送过程中逐渐构建起来的。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOEQdZYLWbFzmTOqCsg%2Fimage.png)

类似的，当一个操作系统收到了一个packet，它会先解析第一个header并知道这是Ethernet，经过一些合法性检查之后，Ethernet header会被剥离，操作系统会解析下一个header。在Ethernet  header中包含了一个类型字段，它表明了该如何解析下一个header。同样的在IP header中包含了一个protocol字段，它也表明了该如何解析下一个header。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOEQnqX1sjdNoswyO7F%2Fimage.png)

软件会解析每个header，做校验，剥离header，并得到下一个header。一直重复这个过程直到得到最后的数据。这就是嵌套的packet header。

## Internet

Ethernet header足够在一个局域网中将packet发送到一个host。如果你想在局域网发送一个IP packet，那么你可以使用ARP获得以太网地址。但是IP协议更加的通用，IP协议能帮助你向互联网上任意位置发送packet。下图是一个IP packet的header，你们可以在lab配套的代码中的net.h文件找到。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOESlRbJsr7WEE4pJzi%2Fimage.png)

如果IP packet是通过以太网传输，那么你可以看到，在一个以太网packet中，最开始是目的以太网地址，源以太网地址，以太网类型是0x0800，之后是IP header，最后是IP payload。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOETibgLGajiB82bzej%2Fimage.png)

在一个packet发送到世界另一端的网络的过程中，IP header会被一直保留，而Ethernet header在离开本地的以太网之后会被剥离。或许packet在被路由的过程中，在每一跳（hop）会加上一个新的Ethernet header。但是IP header从源主机到目的主机的过程中会一直保留。

IP header具有全局的意义，而Ethernet header只在单个局域网有意义。所以IP header必须包含足够的信息，这样才能将packet传输给互联网上遥远的另一端。对于我们来说，关键的信息是三个部分，目的IP地址（ip_dst），源IP地址（ip_src）和协议（ip_p）。目的IP地址是我们想要将packet送到的目的主机的IP地址。地址中的高bit位是网络号，它会帮助路由器完成路由。IP header中的协议字段会告诉目的主机如何处理IP payload。

如果你们看到过MIT的IP地址，你们可以看到IP地址是18.x.x.x，虽然最近有些变化，但是在很长一段时间18是MIT的网络号。所以MIT的大部分主机的IP地址最高字节就是18。全世界的路由器在看到网络号18的时候，就知道应该将packet路由到离MIT更近的地方。

接下来我们看一下包含了IP packet的tcpdump输出。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOEXfK0lywrVj1OVgD7%2Fimage.png)

因为这个IP packet是在以太网上传输，所以它包含了以太网header。呃……，实际上这个packet里面有点问题，我不太确定具体的原因是什么，但是Ethernet header中目的以太网地址不应该是全f，因为全f是广播地址，它会导致packet被发送到所有的主机上。一个真实网络中两个主机之间的packet，不可能出现这样的以太网地址。所以我提供的针对network lab的方案，在QEMU上运行有点问题。不管怎么样，我们可以看到以太网目的地址，以太网源地址，以及以太网类型0x0800。0x0800表明了Ethernet payload是一个IP packet。

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOEZ4BXyrGBTXmWOQvC%2Fimage.png)

IP header的长度是20个字节，所以中括号内的是IP header，

![img](lab11.assets/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MOEZ9iqKkt92zbewl4r%2F-MOPw1qnEjiOAkSTe3Yi%2Fimage.png)

从后向前看：

- 

  目的IP地址是0x0a000202，也就是10.0.2.2。

- 

  源IP地址是0x0a00020f，也就是10.0.2.15。

- 

  再向前有16bit的checksum，也就是0x3eae。IP相关的软件需要检查这个校验和，如果结果不匹配应该丢包。

- 

  再向前一个字节是protocol，0x11对应的是10进制17，表明了下一层协议是UDP

- 

  其他的就是我们不太关心的一些字段了，例如packet的长度。

IP header中的protocol字段告诉了目的主机的网络协议栈，这个packet应该被UDP软件处理。

## UDP