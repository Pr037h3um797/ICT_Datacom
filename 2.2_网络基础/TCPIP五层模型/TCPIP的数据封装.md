数据在传输过程中，会沿着协议堆栈向下传递，在每一层添加各种协议信息。这个过程成为封装。

数据块在TCP/IP的每一层的形式被称为(Packet Data Unit，协议数据单元)。

* 应用层：
    Data数据
* 传输层
    Segment段
* 网络层
    Packet包
* 数据链路层
    Frame帧
* 物理层
    Bit位



## 发送方数据封装
封装过程自上而下完成：
* 数据分为若干个数据段
* TCP数据段封装在IP数据包中
* IP数据包封装在以太网帧中

    应用层↓     **数据 DATA**
    DATA

    传输层↓     **段 Segment**
    TCP Header | DATA

    网络层↓     **包 Packet**
    IP Header | TCP Header | DATA

    数据链路层↓ **帧 Frame**
    Eth Header | IP Header | TCP Header | DATA | FCS

    物理层 **位 Bit、比特流Bitstream** → 传输介质
    bit流

交换机只解封装到二层头部（数据链路层），路由器只解封装到三层头部（网络层）