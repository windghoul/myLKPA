##**2.3 分页机制**

分页机制在段机制之后进行，以完成线性——物理地址的转换过程。段机制把虚拟地址转换为线性地址，分页机制进一步把该线性地址再转换为物理地址。

如果不允许分页(CR0的最高位置0)，那么经过段机制转化而来的32位线性地址就是物理地址。但如果允许分页(CR0的最高位置1)，就要将32位线性地址通过一个地址变换机构转化成物理地址。80X86规定，分页机制是可选的，但很多操作系统主要采用分页机制。

###**2．3．1页与页表**

**1．页、物理页面及页大小**

为了效率起见，将线性地址空间划分成若干大小相等的片，称为页（Page），并给各页加以编号，从0开始，如第0页、第1页等。相应地，也把物理地址空间分成与页大小相等的若干存储块，称为（物理）块或页面（Page Frame），也同样为它们加以编号，如0＃页面、1＃页面等。如图 2.10所示，图中用箭头把线性地址空间中的页，与对应的物理地址空间中的页面联系起来，表示把线性地址空间中若干页将分别装入到多个可以不连续的物理页面中。例如第0页将装入到第2页面，第1页将装入到第0页面，但是第2页也将装入到第2个页面，这似乎是一种错误，但学过内存管理一章后会理解这是一种正常现象。本节只涉及分页机制的一般原理，更多的内容将在第四章内存管理一章讲述。

![](http://i.imgur.com/AAQzURm.png)



那么，页的大小应该为多少？页过大或过小都会影响内存的使用率。其大小在设计硬件分页机制时就必须确定下来，例如80X86支持的标准页大小为4KB（也支持4MB），从后面的内容可以看出，选择4KB大小既巧妙又高效。

**2．页表**

   
页表是把线性地址映射到物理地址的一种数据结构。参照段描述符表，页表中应当包含如下内容：

（1）	物理页面基地址：线性地址空间中的一个页装入内存后所对应的物理页面的起始地址。

（2）	页的属性：表示页的特性。例如该页是否在内存，是否可被读出或写入等。

由于页面的大小为4KB，它的物理页面基地址（32位）必定是4K的倍数，因此其地址的最低12位总是0，那么就可以用这12位存放页的属性，这样用32位完全可以描述页的映射关系，也就是页表中每一项（简称页表项）占4个字节就足够。

不过，4 GB的线性空间可以被划分为1M个4K大小的页，每个页表项占4个字节，则1M个页表项的页表就需要占用4 MB空间，而且还要求是连续的，显然这是不现实的。我们可以采用两级页表来解决这个问题。

**3．两级页表**

所谓两级页表就是对页表再进行分页。第一级称为页目录，其中存放的是关于页表的信息。4MB的页表再次分页（4MB／4K）可以分为1K个页，同样对每个页的描述需要4个字节，于是可以算出页目录最多占用4KB个字节，正好是一个页，其示意图如2.11所示。

![](http://i.imgur.com/KSeEC29.png)

页目录共有1K个表项， 于是，线性地址的最高10位(即22位~ 31位)用来产生第一级的索引。两级表结构的第二级称为页表，每个页表也刚好存放在一个4K字节的页中，包含1K个字节的表项。第二级页表由线性地址的中间10位(即21位~ 12位)进行索引，最低12位表示页内偏量。具有两级页表的线性地址结构如图2.12所示。

![](http://i.imgur.com/nhzAhu6.png)
   
 **4．页表项结构**


不管是页目录还是页表，每个表项占四个字节，其表项结构基本相同，如图2.13所示：

![](http://i.imgur.com/XsTFcGM.png)
       
物理页面基地址： 对页目录而言，指的是页表所在的物理页面在内存的起始地址，对页表而言，指的是页所对应的物理页面在内存的起始物理地址。因为其最低12位全部为0，因此用高20位来描述32位的地址。
属性包括：

（1）	第0位是P（Present），如果P=1，表示页装入到内存中，如果P=0，表示不在内存中。

（2）	第1位是R/W（Read/Write），第2位是U/S（User/Supervisor）位，这两位为页表或页提供硬件保护。

（3）	第3位是PWT（Page Write-Through）位，表示是否采用写透方式，写透方式就是既写内存（RAM）也写高速缓存,该位为1表示采用写透方式

（4）	第4位是PCD（Page Cache Disable）位，表示是否启用高速缓存,该位为1表示启用高速缓存。

（5）	第5位是访问位，当对相应的物理页面进行访问时，该位置1。

（6）	第7位是Page Size标志，只适用于页目录项。如果置为1，页目录项指的是4MB的页

（7）	第9~11位由操作系统专用，Linux也没有做特殊之用。

  **5．硬件保护机制**

对于页表，页的保护是由U/S标志和R/W标志来控制的。当U/S标志为0时，只有处于内核态的操作系统才能对此页或页表进行寻址。当这个标志为1时，则不管在内核态还是用户态，总能对此页进行寻址。

此外，与段的三种存取权限（读、写、执行）不同，页的存取权限只有两种（读、写）。如果页目录项或页表项的读写标志为0，说明相应的页表或页是只读的，否则是可读写的。

###**2．3．2线性地址到物理地址的转换**

当访问线性地址空间的一个操作单元时，如何把32位的线性地址通过分页机制转化成32位物理地址呢？过程如图2.14所示。

![](http://i.imgur.com/bbnzBns.png)

第一步，用32位线性地址的最高10位第31~22位作为页目录项的索引，将它乘以4，与CR3中的页目录的起始地址相加，获得相应目录项在内存的地址。

第二步，从这个地址开始读取32位页目录项，取出其高20位，再给低12位补0，形成的32位就是页表在内存的起始地址。

第三步，用32位线性地址中的第21~12位作为页表中页表项的索引，将它乘以4，与页表的起始地址相加，获得相应页表项在内存的地址。

第四步，从这个地址开始读取32位页表项，取出其高20位，再将线性地址的第11~0位放在低12位，形成最终32位页面物理地址。
    
###**2．3．3 分页举例**

下面举一个简单的例子，这将有助于读者理解分页机制是怎样工作的。

假如操作系统给一个正在运行的进程分配的线性地址空间范围是0x20000000 到 0x2003ffff。这个空间由64页组成。我们暂且不关心这些页所在的物理页面的地址，只关注页表项中的几个域。

我们从分配给进程的线性地址的最高10位（分页硬件机制把它自动解释成页目录域）开始。这两个地址都以2开头，后面跟着0，因此高10位有相同的值，即十六进制的0x080或十进制的128。因此，这两个地址的页目录域都指向进程页目录的第129项。相应的目录项中必须包含分配给进程的页表的物理地址,如图2.15。如果给这个进程没有分配其它的线性地址，则页目录的其余1023项都为0，也就是这个进程在页目录中只占一项。

![](http://i.imgur.com/68AVQKS.png)
 
中间10位的值（即页表域的值）范围从0到0x03f，或十进制的从0到63。因而只有页表的前64个表项是有意义的，其余960表项填为0。

假设进程需要读线性地址0x20021406中的内容。这个地址由分页机制按下面的方法进行处理:

1．目录域的0x80用于选择页目录的第0x80目录项,此目录项指向页表。

2．页表域的第0x21项用于选择页表的第0x21表项,此表项指向页所对应的内存物理页面。

3．最后,偏移量0x406用于在目标物理页面中读偏移量为0x406中的字节。

如果页表第0x21表项的Present标志为0，说明此页还没有装入内存中；在这种情况下，分页机制在转换线性地址的同时产生一个缺页异常（参见内存管理一章）。无论何时，当进程试图访问限定在0x20000000到0x2003ffff范围之外的线性地址时，都将产生一个缺页异常，因为这些页表项都填充了0，尤其是，它们的Present标志都为0。  
                       
###**2．3．4 页面高速缓存**

由于在分页情况下，页表是放在内存中的，这使CPU在每次存取一个数据时,都要至少两次访问内存,从而大大降低了访问速度。所以，为了提高速度，在80X86中设置一个最近存取页的高速缓存硬件机制，它自动保持32项处理器最近使用的页表项，因此，可以覆盖128K字节的内存地址。当访问线性地址空间的某个地址时，先检查对应的页表项是否在高速缓存中，如果在，就不必经过两级访问了，如果不在，再进行两级访问。平均来说，页面高速缓存大约有90%的命中率，也就是说每次访问存储器时，只有10%的情况必须访问两级分页机构。这就大大加快了速度，页面高速缓存的作用如图2.16所示。有些书上也把页面高速缓存叫做“联想存储器”或“转换旁视缓冲器（TLB）”。

![](http://i.imgur.com/3YLFk45.png)
