# PTP

##### PTP报文类

事件报文：计时的报文，在发送和接收时产生精确时间戳

通用报文：不要求精确时间戳

**事件报文集**：

-Sync

-Delay_Req

-Pdelay_Req

-Pdelay_Resq

**通用报文集**：

-Announce

-Follow_Up

-Delay_Resp

-Pdelay_Resp_Follow_up

-Management

-Signaling ：用来传送一个或多个TLV（Type Length Value）实体序列

> Sync、Delay_Req、Follow_Up和Delay_Resq报文产生并传递时间信息，通过**延时请求-响应机制**来同步普通时钟和边界时钟

> Pdelay_Req、Pdelay_Resq和Pdelay_Resp_Follow_up报文用于测量实现**对等延时机制**的两个时钟端口的**链路延时**。



##### PTP设备类型

- 普通时钟：可主可从
- 边界时钟：有多个PTP端口，连接两个Domain
- 端到端**透明时钟**：不进行同步，转发所有PTP报文，计算整个链路的延时
- 点到点**透明时钟**：仅校正和转发Sync和Follow_Up报文，基于对等延时计算链路延时
- 管理节点



***

#### 同步综述

##### 协议正常执行的两个阶段

1. 建立主从层次结构
2. 同步时钟



##### 建立主从层次结构

检查端口收到的Announce报文内容，使用BMC对报文内容和与OC或BC相关的数据集进行分析，决定时钟的每个端口状态。

1. MASTER：时间源
2. SLAVE：同步到路径上具有MASTER状态的端口设备
3. PASSIVE：不是主时钟，也不同步

##### 最佳主时钟算法

本地时钟端口接收几个Announce报文，通过各个报文描述的时钟，确认最佳时钟。

BMC由两个单独算法组成

-**数据集比较算法**

比较各时钟属性。精度，稳定性等

```c
foreignMasterDS //最小5个外部主时钟记录
{
    foreignMasterPortIdentity; //接收到的Announce中sourcePortIdentity
    foreignMasterAnnounceMessages; //Announce个数
}
```



-**状态决定算法** P70

在数据集比较算法基础上确定端口的下一个状态，MASTER,SLAVE OR PASSIVE，

##### 简单主从层次结构

<img src="D:\Desktop\WorkShop\笔记截图\PTP\Snipaste_2020-06-30_19-38-36.png" style="zoom:67%;" />

#### 报文时间戳的产生

<img src="D:\Desktop\WorkShop\笔记截图\PTP\Snipaste_2020-07-01_00-52-12.png" style="zoom: 67%;" />

#### PTP通信

##### 报文属性

1. **事件报文**

   Sync
   
   Delay_Req
   
   Pdelay_Req：将Pdelay_Req报文发送给作为对等延时机制一部分的另一个端口，以确定端口间链路延时。
   
   Pdelay_Resp：发送Pdelay_Resp报文作为**Pdelay_Req的响应**，在报文中传输时间戳信息
   
2. **通用报文**

   Announce：报文提供发送节点和它的最高级时钟的状态和特性信息，在BMC时使用

   Follow_Up

   Delay_Resp

   Pdelay_Resp_Follow_Up

   Management：管理报文，传输管理时钟的信息和命令

   Signaling：信号报文，在时钟间传递信息、请求和命令

##### PTP端口

<img src="C:\Users\Ricardo\AppData\Roaming\Typora\typora-user-images\1593870478206.png" alt="1593870478206" style="zoom:67%;" />

```c
portIdentity
{
    .clockIdentiy;
    .portNumber
}
/* 设备属性 */
Priority1
Priority2
clockClass
clockAccuracy
timeSource
numberPorts
```

##### PTP方差(7.6.3)

两个方差估计：

offsetScaledLogVariance;

observedParentOffsetScaledLogVariance

##### 报文发送间隔

```c
portDS.log(Announce/Sync/MinDelayReq/MinPdelayReq)Interval
```

##### PTP数据集

1. 普通时钟和边界时钟

   ```c
   defaultDS
   {
       /* 静态成员 */
       twoStepFlag;  /* TRUE OR FALSE */
       clockIdentity; 				  	
       numberPorts;   
       
       /* 动态成员 */
       clockQuality;  
       Priority 1;
       Priority 2;
       domainNumber;
       slaveOnly;
   }
   currentDS
   {	
       /* 动态成员 */
       stepsRemoved;
       offsetFromMaster; // = （从时钟时间）-（主时钟时间）
       meanPathDelay;
   }
   parentDS
   timePropertiesDS
   portDS /* 普通时钟和边界时钟每个端口的一个数据集 */
   {
       portIdentity;
       portState;
       logMinDelayReqInterval;
       peerMeanPathDelay;
       logAnnounceInterval;
       announceReceiptTimeout;
       logSyncInterval;
       delayMechanism;
       logMinPdelaeqInterval;
       versionNumber;
   }
   ```

##### PTP状态机（9.2.5）

P76

##### PTP报文发送

Delay_Req报文通过**广播**发送，除非单播。 

#### 时钟偏移，路径延时，驻留时间，不对称校正

##### 偏移计算

oneStep: offsetFromMaster = syncEventIngressTimestamp - originTimestamp - meanPathDelay - correctionField(Sync)

twoStep: offsetFromMaster = syncEventIngressTimestamp - preciseOriginTimestamp - meanPathDelay - correctionField(Sync) - correctionField(Follow_Up)

##### 延时请求-响应机制（11.3.1）

P100

twostep: meanPathDelay = [ (t2 - t3) + ( receiveTimestamp(**Delay_Resp**) - originTimestamp(**Sync**) ) - correctionField(**Follow_Up**) - correctionField(**Delay_Resq**)] / 2

##### 对等延时机制

> 在OC和BC里，对等延时机制与端口是主或从无关

<img src="C:\Users\Ricardo\AppData\Roaming\Typora\typora-user-images\1595760205255.png" alt="1595760205255" style="zoom: 67%;" />

##### 透明时钟驻留时间校正（11.5）

P105

##### 事件报文不对称校正

Sync : 进入校正，从时钟**接收报文时** delayAsymmetry + correctionField(**Sync**)

Delay_Req : 离开校正，从时钟**发送前**，correctionField(**Delay_Req**) - delayAsymmetry



Pdelay_Req :  离开校正，**发送前**，correctionField(**Pdelay_Req**) - delayAsymmetry

Pdelay_Resp : 进入校正，**接收报文时** delayAsymmetry + correctionField(**Pdelay_Resp** )

##### PTP报文格式（13.1）

P110

##### 精度问题

- 协议栈的延时波动 ：使用PHY
- 延时的不对称 ：WR模型？
- 网络组件延时波动 ：PTP事件报文高优先级发送，求平均值
- 时间戳精度 ：使用PHY打戳
- 稳定性 ：高精度和稳定的晶振，减小syncInterval

##### 单播或非网桥和路由网络中实现

##### 驻留时间和不对称校正实例

P173