---
title: TuringUi系列(一)
categories:
  - 项目
  - TuringUi
tags:
  - SSD1306
  - IIC+DMA
abbrlink: 10692
date: 2024-12-17 15:34:00

---

# TuringUi系列(一)

>- **CzrturingB：**想要做一套OledUi框架，这个系列的部分内容是对大学学习期间笔记的整理，所以内容不是很完整。
>   - 主控为**STM32F4系列**
>   - Oled驱动芯片为**SSD1306** 
>   - 通信方式为**IIC**
>   - 嵌入式系统为**FreeRTOS**
>
>
>****

## 第一部分	IIC通信简述[^1]

### 第一章	相关定义

- IIC[Inter IC]：即Philips公司开发的一个简单的双向两线总线，用于实现IC芯片直接的通信控制。

- 特征：

  1. 需要两条总线线路：一条串行数据总线（SDA），一条串行时钟总线（SCL）。

  2. 连接在总线上的器件都有唯一确定的地址。

  3. 主从机关系、发射器与接收器关系。

  4. 时间同步机制：同步高速与慢速IC器件的数据通讯。

  5. 多主机仲裁机制：当多个主机同时初始化总线时，通过仲裁机制来选定主机控制总线。

  6. 连接到IIC总线上的IC数量仅收到总线的最大电容400pF限制。

  7. 芯片片上滤波器可以滤除总线数据线上的毛刺，进而保证数据传输的完整性。

  8. 串行8位双向数据传输位速率：

     |   模式   | 数据传输速率 |
     | :------: | :----------: |
     | 标准模式 |  100 kbit/s  |
     | 快速模式 |  400 kbit/s  |
     | 高速模式 |  3.4 Mbit/s  |

- 术语：

  - 发送器：将数据发送到总线上的器件。

  - 接收器：从总线上接收数据的器件。

    ```
    CzrTuringB：发送器与接收器的判定取决于IC芯片所需要完成的功能，某些IC芯片只能是接收器或发送器，例如：显示器、键盘接收。某些IC芯片既可以是接收器也可以是发送器，例如：微控制器。
    ```

  - 主机：即为控制IIC总线的器件，其负责初始化数据发送、产生时钟信号、终止数据发送等功能。

  - 从机：总线上可以被主机寻址的器件称为从机。

    ```
    CzrTuringB：
    	@IIC是支持多主机通信，但是在一个交流过程中IIC总线上只能存在一个器件控制组件，当同一时刻有多个IC申请成为主机时，需要总线对其进行仲裁以选取主机。
    	@总线通信过程中，除主机外其余皆为从机。
    ```

  - 仲裁机制：负责裁决同一时刻那个主机能够控制总线。

  - 同步机制：即两个或多个器件同步时钟信号的过程。

### 第二章	标准IIC

#### 第一节	通信协议基础

- 两条总线默认电平：高电平

  ```
  CzrTuringB：SDL和SCL都通过一个电流源或上拉电阻连接到正的电源电压，所以在总线空闲时，其默认电平为高电平。
  ```

  ![为什么IIC总线默认电平是高电平](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/001.png)

- 数据有效性：

  - SDA数据线上的数据必须在SCL时钟为高电平期间保持稳定。
  - SDA数据线上的数据必须在SCL时钟为低电平期间进行高低电平的切换。

  ![数据有效性时序图](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/002.png)

- 总线起始和停止条件：在本应该SDA数据稳定时发生了数据切换，则表明总线的起始和停止

  - 起始条件：在SCL为高电平时，SDA从高电平拉低到低电平。
  - 停止条件：在SCL为低电平时，SDA从低电平拉高到高电平。

  ![总线起始结束条件时序图](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/003.png)

- 数据传输格式：

  - 发送到SDA线上的每个字节位数为8位，每次通信过程中可以发送的字节数量不受限制。
  - 在传递一个字节时，先发送字节的最高位。
  - 通信过程中，每次发送一个字节后都要跟着一个应答位，只有在接收器应答后，发送器才能发送后面的字节。

  | 字节最高位 | 字节其余低位 | 应答位 |
  | :--------: | :----------: | :----: |

- 应答信号：

  - 即通信过程中，在传输数据每个字节后紧跟着一个应答位，在响应的时钟脉冲期间(SCL为低电平的时候)，发送器首先拉高串行数据线SDA。之后如果从机想要响应应答位，那么从机需在响应的脉冲持续期间(SCL为低电平的时候)拉低串行数据线SDA并在响应的时钟脉冲期间(SCL为高电平的时候)保持稳定即表示应答反之则表示不应答，发送器继续发送后续字节。
  - 通常情况下接收器不响应发送器的应答，即表示IC之间通信的停止，此时主机应该发送停止条件或发送重复起始条件与其他IC进行通信。

  ![应答信号时序图](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/004.png)

- 从机地址字节：

  - 在主机启动总线后，会发送一个从机地址，只能与其交流的从机。

  - 从机地址共有七位。从机地址某部分是固定的，某部分是根据器件的引脚电平状态决定的，具体信息参考器件数据手册与电路原理图。

  - 特殊地址：

    | 从机地址 | R/W' |     描述     |
    | :------: | :--: | :----------: |
    | 0000 000 | 0/1  | 广播呼叫地址 |

  - 从机地址字节最后一位指定数据的传输方向，其中0表示主机发送数据，1表示主机读取数据。

  | 前七位从机地址 | 数据传输方向位 | 应答位 |
  | :------------: | :------------: | :----: |

#### 第二节	通信协议进阶

- 主机的职责：

  - 控制总线：

    ```
    @主机产生起始和结束条件以控制总线状态，当IIC总线为空闲状态时，IC器件发送起始信号(即判定为主机)将IIC总线切换为忙碌状态。当通信结束时，主机发送停止条件，将IIC总线切换为空闲状态。
    @在与某个从机通信完成后，主机可重复发送起始信号并紧跟另一个从机地址(或只改变了数据的传输方向)来进行二次通信;
    ```

  - 控制通信：

    ```
    @在与一个从机通信完成后，主机可以不发送停止条件而发送一个重复起始条件并紧跟着发送另一个从机地址，来更换与主机通信的器件。
    ```

  - 产生时钟信号，用于慢速与快速器件的时钟同步。

- 从机的权限：

  - 暂停总线（用时钟同步机制作为握手）：

    ```
    @在通信过程中从机接收完一个数据字节后，想要处理某些程序代码，那么从机可以拉低SCL信号线并保持，暂停IIC通信。当从机准备好接收下一个数据时可以拉高SCL，恢复IIC总线的工作。
    ```

- 多主机仲裁机制：

  - 为什么要有主机仲裁机制？

    ```
    CzrTuringB：
    	IIC是一个支持多主机的总线协议，在一个嵌入式系统内部，器件的工作频率是不同的，有的器件处理信息快，有的器件处理信息慢，所以为了同步两个不同处理速度器件的通信过程，IIC总线协议规定，主机负责产生时钟信号用于同步。因此同一时刻总线上有且只能有一个主机将自己的时钟信号传递到SCL线上，为了避免多个主机同时抢占总线的现象发生，所以引入仲裁机制。
    	主机只能在总线空闲时启动数据传输(发送起始条件)。在起始条件时间内，总线上可能会有多个器件发送起始信号，此时需要对其进行冲裁，选择由那个器件作为主机。
    ```

  - 仲裁机制：仲裁会比较两个申请成为主机的器件发送的数据，第一个发送想要拉高SDA的器件即丢失主机资格。【因为起始条件将SDA拉低了，那么第一个想要控制SDA数据线的器件就不能成为主机】

    ```
    @首先比较地址字节，如果主机们都想要寻址同一个器件，那么接着比较后续的数据位。
    ```

- 时间同步机制：	

  - 为什么要有时间同步机制？

    ```
    CzrTuringB：
    	IIC总线上器件的工作频率是不同的，所以可能会产生发送器发送数据过快而接收器接收数据过慢导致数据丢失情况，或发生发送器发送数据过慢二接收器接收数据过快导致时效性降低的情况。因此需要引入同步机制，同步两方的工作频率。
    ```

  - 时钟信号低电平同步：

    - 当时钟信号低电平时，数据信号线上的数据进行高低转换，因此工作频率快的器件高低电平转换速度快，所以为了同步工作频率慢的器件，需要让低频率器件控制总线SCL信号线的拉高。

    - 流程：

      ```
      说明：
      	1、CLK1时钟信号由快速器件产生，CLK2时钟信号由慢速器件产生，SCL即为IIC总线上的时钟信号线。
      流程描述：
      	1、刚开始SCL由快速器件控制，当快速器件将SCL信号线拉低时，此时SDA信号线需要完成高低电平的转换。
      	2、在SCL切换为低电平时，快速器件释放SCL的控制权，慢速器件在此时进行SDA的数据转换，当慢速器件SDA数据转换完成后，慢速器件拉高SCL时钟信号线，并将SCL控制权交给快速器件。
      	3、同步完成。
      ```

  - 时钟信号高电平同步：

    - 当时钟信号高电平时，数据信号线上数据有效用于数据传输，因此工作频率快的器件接收数据更快，所以为了同步工作频率慢的器件，需要让高频率器件控制总线SCL信号线的拉低。

      ```
      流程描述：
      	1、刚开始SCL由慢速器件控制，当慢速器件将SCL信号线拉高时，此时发送器将数据发送给总线，接收方从总线上接收数据。
      	2、在SCL切换为高电平时，慢速器件释放SCL的控制权，快速器件在此时进行SDA的数据接收，当快速器件SDA数据接收完成后，快速器件拉低SCL时钟信号线，并将SCL控制权交给慢速器件。
      	3、同步完成。
      ```

  - 核心思想：同步SCL时钟的低电平周期由低电平时钟周期最长的器件决定而高电平周期由高电平时钟周期最短的器件决定。

    ```
    CzrTuringB：
    	IIC总线上的器件是线与连接在SCL信号线上的，因此只有当所有器件都从低电平转为高电平时SCL才转换为高电平。当且仅当有一个器件从高电平转换为低电平，那么SCL即转换为低电平。
    ```

- 广播呼叫：

  - 广播即主机呼叫IIC总线上的所有器件，然后IIC总线上的所有器件根据具体情况决定是否响应主机的广播呼叫。
  - 当R/W'位为0时：
    - 紧跟第二个字节为0000 0110，表明通过硬件写入和复位从机地址的可编程部分。所有打算响应广播的器件会复位并接收他们的地址可编程部分。
    - 紧跟第二个字节为0000 0100，表明通过硬件写从机的可编程部分，所有打算响应广播的器件会所存他们的地址可编程部分。
  - 当R/W'位为1时：
    - 这种情况适用于主机无法编程发送自己期望的从机地址(键盘扫描器)。
    - 由于主机无法向IIC发送自己期望的从机地址，那么他就会默认发送这个广播呼叫地址，挂载在IIC总线上的微控制器接收到这个广播呼叫时，会发送从机地址至IIC总线。
    - 第二个字节即为这个硬件主机的地址，其能被微控制器识别并指引硬件主机的信息。

#### 第三节	通信实例

- 主机作为发送器：
  1. 主机在SCL高电平器件拉低SDA发送起始信号，使IIC总线变为忙碌状态。
  2. 主机发送从机地址字节，指明要交流的从机和数据传输方向(写)。
  3. 从机应答从机地址字节。
  4. 主机发送字节数据。
  5. 从机应答字节数据。
  6. ………
  7. 主机发送停止信号。
- 主机作为接收器：
  1. 主机在SCL高电平器件拉低SDA发送起始信号，使IIC总线变为忙碌状态。
  2. 主机发送从机地址字节，指明要交流的从机和数据传输方向(读)。
  3. 从机应答从机地址字节。
  4. 从机发送字节数据。
  5. 主机应答字节数据。
  6. ………
  7. 主机发送停止信号。
- 主机刚开始作为发送器，之后作为接收器：
  1. 主机在SCL高电平器件拉低SDA发送起始信号，使IIC总线变为忙碌状态。
  2. 主机发送从机地址字节，指明要交流的从机和数据传输方向(写)。
  3. 从机应答从机地址字节。
  4. 主机发送字节数据。
  5. 从机应答字节数据。
  6. ………
  7. 主机发送重复起始信号。
  8. 主机发送从机地址字节，指明要交流的从机和数据传输方向(读)。
  9. 从机应答从机地址字节。
  10. 从机发送字节数据。
  11. 主机应答字节数据。
  12. ………
  13. 主机发送停止信号。

### 第三章	注意事项

- 软件模拟IIC时，连接到总线的器件输出级必须是漏极开漏或集电极开路的，只有这样才能执行线与功能。

## 第二部分	SSD1306驱动[^2]

### 第一章	我的想法

>- 快速模式I2C
>- DMA传输
>- 双缓冲技术
>- 基于CRC算法的局部块区域更新技术

- 相关规定：

  - 坐标系定义：

    ![屏幕坐标系定义](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/007.png)

  - 块大小定义：8×8大小的区域，于是屏幕由8×16个块组成。每个块共八个字节的数据，使用CRC算法对每个块计算校验码，对比连续两帧每个块的校验码变化与否，来进行区域更新。

- 图像帧的理解：

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/010.png)

  动画由连续的帧图像组成，可将图像帧周期分为图像计算和屏幕刷新两部分。在图像计算时屏幕显示当前图像，若图像计算时间过长则动画会出现卡顿现象。若屏幕刷新时间过程则动画会出现屏幕闪烁现象，为了减小这种现象的发生可采用更新传输手段、局部刷新算法以减小屏幕刷新时间。

### 第二章	相关数据结构的实现

#### 第一节	环形队列的实现

>**CzrTuringB：**如何实现可适应多数据类型的队列？**Void*指针**[^3]

- C文件：

  ```c
  /**
    *@ FileName: QUEUE.c
    *@ Author: CzrTuringB
    *@ Brief: 队列
    *@ Time: Sep 22, 2024
    *@ Requirement：
    */
  /* Includes ------------------------------------------------------------------*/
  #include "Queue.h"
  /* Data ------------------------------------------------------------------*/
  /* Functions ------------------------------------------------------------------*/
  BspState QueueInit(pQueue sQueue, size_t size, size_t capacity, void* pdQueue)
  {
  	if(pdQueue == NULL)
  	{
  		//内存分配错误
  		return BspError;
  	}
      sQueue->elementSize = size;
      sQueue->capacity = capacity;
      sQueue->count = 0;
      sQueue->head = 0;
      sQueue->tail = 0;
      sQueue->pdQueue = pdQueue;
      return BspOk;
  }
  BspState QueueInElement(pQueue sQueue, const void *data)
  {
  	if(sQueue->count == sQueue->capacity)
  	{
  		//队列满了
  		return BspError;
  	}
  	//计算数据应存入队列的所在位置
      void *dest = (char *)sQueue->pdQueue + (sQueue->tail * sQueue->elementSize);
      memcpy(dest, data, sQueue->elementSize);
      //使用取模运算，实现环形队列的效果
      sQueue->tail = (sQueue->tail + 1) % sQueue->capacity;
      sQueue->count++;
      return BspOk;
  }
  BspState QueueOutElement(pQueue sQueue, void *data)
  {
  	if(sQueue->count == 0)
  	{
  		//队列为空
  		return BspError;
  	}
  	//计算应从队列的那个位置取出数据
      void *src = (char *)sQueue->pdQueue + (sQueue->head * sQueue->elementSize);
      memcpy(data, src, sQueue->elementSize);
      //使用取模运输算，实现环形队列的效果
      sQueue->head = (sQueue->head + 1) % sQueue->capacity;
      sQueue->count--;
      return BspOk;
  }
  
  ```

- H文件：

  ```h
  /**
    *@ FileName: Queue.h
    *@ Author: CzrTuringB
    *@ Brief:
    *@ Time: Sep 22, 2024
    *@ Requirement：
    */
  #ifndef __QUEUE_H
  #define __QUEUE_H
  /* Includes ------------------------------------------------------------------*/
  #include <stdlib.h>
  #include <string.h>
  #include "BspState.h"
  /* Data ------------------------------------------------------------------*/
  typedef struct Queue
  {
  	    size_t elementSize;  // 单个元素的大小
  	    size_t capacity;     // 队列容量（最大元素数）
  	    size_t head;         // 队列头指针
  	    size_t tail;         // 队列尾指针
  	    size_t count;        // 队列当前元素数
  	    void* pdQueue;		 // 存储数据的数组指针
  }Queue,*pQueue;
  /* Functions------------------------------------------------------------------*/
  /**
    *@ FunctionName: BspState QueueInit(pQueue sQueue, size_t size, size_t capacity, void* pdQueue)
    *@ Author: CzrTuringB
    *@ Brief: 初始化队列相关配置
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  BspState QueueInit(pQueue sQueue, size_t size, size_t capacity, void* pdQueue);
  /**
    *@ FunctionName: BspState QueueInElement(pQueue sQueue, const void *data)
    *@ Author: CzrTuringB
    *@ Brief: 将一个数据送入队列中
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  BspState QueueInElement(pQueue sQueue, const void *data);
  /**
    *@ FunctionName: BspState QueueOutElement(pQueue sQueue, const void *data)
    *@ Author: CzrTuringB
    *@ Brief: 将一个数据从队列中读出
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  BspState QueueOutElement(pQueue sQueue, void *data);
  #endif
  ```

### 第三章	SSD1306驱动程序设计

#### 第一节	相关参数

```c
typedef struct
{
	Queue cmdQueue;				    //定义命令队列
	uint8_t disBuffer[8][128];		//屏幕显示缓冲数组
	uint8_t markBlock[16];			//缓冲块更新标记数组,一个字节有八位，每位1表示块更新，0表示块没更新
	uint32_t lastCrc[8][16];         //记录每个块的CRC校验码
	Bool dataValid; 			    //数据有效性判断变量
	SemaphoreHandle_t dmaIdle;		 //DMA空闲判断
}SsdConfigStruct;
```

#### 第二节	数据传输

- 数据传输方式：

  1. 方式一【阻塞式单字节命令、数据发送】：不推荐使用
  2. 方式二【DMA式多字节命令、数据发送】

- DMA技术存在的问题：为了避免DMA数据还没发送完，再次调用DMA相关API进行数据传输导致的数据传输丢失错误，需要在每次开启DMA后，查询DMA的状态信息以确保DMA处于空闲状态，故而在程序中引入dmaIdle信号量，其数据传输流程图如下图所示：

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/005.png)
  
- 命令的DMA发送：创建一个环形队列来存储命令流，然后在通过DMA技术将存储在队列中的所有命令发送出去。

- 数据的DMA发送：

  >**CzrTuring：**可以使用DMA将所有的缓存数据发送给屏幕，但是如果屏幕更新内容较少，直接发送整个缓存会造成一定的更新资源的浪费，因此引入基于CRC检验的区域块更新技术。

  - CRC校验：一种用于检测数据在存储或传输过程中是否发生错误的算法。它通过对数据执行特定的多项式运算生成校验值，接收端通过同样的运算对数据进行验证。

  - CRC算法[^4]：

    - 在数据通信中，通信链路并不是理想化的，因此可能会出现传输差错，将1传输成0或将0传输成1，为了接收方能及时检测到传输错误，故而引入差错检测码来检测传输错误与否。常用的校验算法有奇偶校验、CRC校验等。

    - CRC校验即收发双方约定好一个生成多项式$$G(x)$$，发送方基于生成多项式计算出差错检测码，将其添加到待传输数据的后面一起传输，而接收方则通过生成多项式来计算受到的数据是否产生了无码。

    - 发送方算法【本质上是异或运算[^5]】：

      ![发送方算法](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/008.png)

    - 接收方算法：

      ![接收方算法](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%80)/009.png)

  - 基于CRC算法的区域块变更检测：首先将整个屏幕分为128个8×8的区域块，类比数据传输的过程，将相邻两个图像帧分别视为发送和接收过程，使用CRC校验算法检测每个块是否发生数据变化，然后仅更新发生变化的屏幕块数据即可。

#### 第三节	SSD1306驱动程序

>**CzrTuringB：**在TuringUI框架中会尽量避免使用过的SSD1306指令，以适配多种驱动芯片。

- **下面的代码不一定能及时更新，具体实现可以查看我的仓库：**[TuringUi/Software/MyBsp/Oled/TuringUi/Device at main · ChenZR0509/TuringUi](https://github.com/ChenZR0509/TuringUi/tree/main/Software/MyBsp/Oled/TuringUi/Device)

- SSD1306.c：

  ```c
  /**
    *@ FileName: SSD1306.c
    *@ Author: CzrTuringB
    *@ Brief: IIC接口的SSD1306源文件
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  /* Includes ------------------------------------------------------------------*/
  #include "Ssd1306.h"
  #include "DataStructure/Queue.h"
  #include "BspState.h"
  /* Data(作用域为当前C文件)-----------------------------------------------------*/
  //-define
  #define	SSD1306I2cHandle hi2c1			//IIC接口
  #define SSD1306Address 0x78				//从机地址
  #define cmdBufferSize  30				//命令流数组元素个数
  //-variable
  static uint8_t cmdBuffer[cmdBufferSize]; //创建命令数组，用于存储SSD1306的命令流
  //-struct
  typedef struct
  {
  	SsdPublicConfig publicConfig;       			//公有部分
  	Queue cmdQueue;									//定义命令队列
  	uint8_t markBlock[16];							//缓冲块更新标记数组,一个字节有八位，每位1表示块更新，0表示块没更新
  	uint32_t lastCrc[8][16];        				//记录每个块的CRC校验码
  	SemaphoreHandle_t dmaIdle;						//DMA空闲判断
  }SsdConfigStruct;
  static SsdConfigStruct sUiDevice;						//静态结构体变量
  SsdPublicConfig* uiDevice = &sUiDevice.publicConfig;	//结构体共有部分指针变量
  //-enum
  typedef enum
  {
  	Data = 0x40,
  	Command = 0x00,
  }CommunicationMode;
  typedef enum
  {
  	DisplayOn = 0xAF,
  	DisplayOff = 0xAE,
  }DisplayState;
  typedef enum
  {
  	HorizontalMode = 0,	//水平寻址
  	VerticalMode = 1,	//垂直寻址
  	PageMode = 2,		//页寻址
  }AddressMode;
  typedef enum
  {
  	MapOrder = 0xA0,	//顺序映射
  	MapReverse = 0xA1,	//逆序映射
  }ColumnMap;
  typedef enum
  {
  	ScanOrder = 0xC0,	//顺序扫描
  	ScanReverse = 0xC8,	//逆序扫描
  }ComScan;
  typedef enum
  {
  	SnLR = 0x02,	//COM与引脚顺序映射
  	SLR	 = 0x22,	//COM与引脚顺序映射，芯片左右两边映射颠倒
  	AnLR = 0x12,	//COM与引脚自选映射
  	ALR	 = 0x32,	//COM与引脚自选映射，芯片左右两边映射颠倒
  }ComConfig;
  /* Functions ------------------------------------------------------------------*/
  void SSDSetPageAddress(uint8_t pageStart,uint8_t pageEnd);
  void SSDSetColumnAddress(uint8_t columnStart,uint8_t columnEnd);
  /* Functions ------------------------------------------------------------------*/
  /**
    *@ FunctionName: void SSDWriteCmdByte(uint8_t cmd)
    *@ Author: CzrTuringB
    *@ Brief: 向SSD1306芯片中写入一个命令
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、阻塞式传输
    *@ 	2、尽量不要使用这个API
    */
  void SSDWriteCmdByte(uint8_t cmd)
  {
  	HAL_I2C_Mem_Write(&SSD1306I2cHandle, SSD1306Address, Command, I2C_MEMADD_SIZE_8BIT, &cmd, 1, 100);
  }
  /**
    *@ FunctionName: void SSDWriteDataByte(uint8_t data)
    *@ Author: CzrTuringB
    *@ Brief: 向SSD1306芯片中写入一个字节数据
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、阻塞式传输
    *@ 	2、尽量不要使用这个API
    */
  void SSDWriteDataByte(uint8_t data)
  {
  	HAL_I2C_Mem_Write(&SSD1306I2cHandle, SSD1306Address, Data, I2C_MEMADD_SIZE_8BIT, &data, 1, 100);
  }
  /**
    *@ FunctionName: void SSDWriteCmds(void)
    *@ Author: CzrTuringB
    *@ Brief: 使用DMA技术发送多个命令字节
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDWriteCmds(void)
  {
  	HAL_I2C_Mem_Write_DMA(&SSD1306I2cHandle, SSD1306Address, Command, I2C_MEMADD_SIZE_8BIT, cmdBuffer, sUiDevice.cmdQueue.count);
  	xSemaphoreTake(sUiDevice.dmaIdle,portMAX_DELAY);
  	sUiDevice.cmdQueue.head = 0;
  	sUiDevice.cmdQueue.tail = 0;
  	sUiDevice.cmdQueue.count = 0;
  }
  /**
    *@ FunctionName: void SSDBlockDetect(void)
    *@ Author: CzrTuringB
    *@ Brief: 区域块数据变更检测
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  void SSDBlockDetect(void)
  {
      for (uint8_t row = 0; row < 8; row++)
      {
          uint16_t rowMark = 0; // 用16位变量表示一行的更新标记
  
          for (uint8_t col = 0; col < 16; col++)
          {
              uint8_t* pBlockData = &uiDevice->disBuffer[row][col * 8]; // 获取块数据指针
  
              // 使用HAL_CRC_Calculate计算CRC
              uint32_t currentCrc = HAL_CRC_Calculate(&hcrc, (uint32_t *)pBlockData, 2); // 计算CRC
              // 判断当前CRC是否和上次的CRC相同，如果不同则标记该块为已更新
              if (currentCrc != sUiDevice.lastCrc[row][col])
              {
              	sUiDevice.lastCrc[row][col] = currentCrc; // 更新CRC记录
                  rowMark |= (1 << col); // 在对应位置标记更新
              }
          }
  
          // 将16位行标记写入对应的两个 `markBlock` 元素
          sUiDevice.markBlock[row * 2] = (uint8_t)(rowMark & 0xFF);         // 低8位
          sUiDevice.markBlock[row * 2 + 1] = (uint8_t)((rowMark >> 8) & 0xFF); // 高8位
      }
  }
  /**
    *@ FunctionName: void SSDWriteDataAll()
    *@ Author: CzrTuringB
    *@ Brief: 使用DMA技术发送整个数据缓冲区
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDWriteDataAll(void)
  {
  	SSDSetPageAddress(0, 7);
  	SSDSetColumnAddress(0, 127);
  	SSDWriteCmds();
  	HAL_I2C_Mem_Write_DMA(&SSD1306I2cHandle, SSD1306Address, Data, I2C_MEMADD_SIZE_8BIT, (uint8_t*)(uiDevice->disBuffer), 1024);
  	xSemaphoreTake(sUiDevice.dmaIdle,portMAX_DELAY);
  	SSDBlockDetect();								//更新CRC
      for (uint8_t i = 0; i < 16; i++)				//去除标记
      {
      	sUiDevice.markBlock[i] = 0;
      }
  }
  /**
    *@ FunctionName: void SSDUpdateScreen(void)
    *@ Author: CzrTuringB
    *@ Brief: 屏幕刷新函数
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  void SSDUpdateScreen(void)
  {
      SSDBlockDetect(); // 检测哪些块需要更新
      for (uint8_t page = 0; page < 8; page++) // 遍历每个页
      {
          uint16_t pageMark = ((uint16_t)sUiDevice.markBlock[page * 2 + 1] << 8) | sUiDevice.markBlock[page * 2]; // 当前页的更新标记
          while (pageMark) // 只处理有标记的部分
          {
              // 找到最低位的1（起始列块）
              uint8_t startCol = __builtin_ctz(pageMark); // 获取最低位1的索引
  
              // 找到连续的1块
              uint16_t mask = (1 << startCol);
              uint8_t endCol = startCol; // 默认结束列为起始列
  
              // 扩展连续的1块
              while (pageMark & mask)
              {
                  mask <<= 1; // 扩展范围
  
                  // 如果 mask 超过有效位数（16位），则退出
                  if (mask == 0)
                  {
                      break;  // 防止超出有效位范围
                  }
              }
  
              // 如果 mask 变为 0，表示已经处理到最后一列，endCol应该是15
              if (mask == 0)
              {
                  endCol = 15; // 确保最后列的 endCol 为 15
              } else
              {
                  endCol = __builtin_ctz(mask) - 1; // 结束列块
              }
              // 计算起始和结束地址
              uint8_t startColAddr = startCol * 8;
              uint8_t endColAddr = endCol * 8 + 7;
  
              // 设置页和列范围
              SSDSetPageAddress(page, page); // 设置页地址
              SSDSetColumnAddress(startColAddr, endColAddr); // 设置列地址范围
              SSDWriteCmds();
  
              // 发送连续块列的数据
              uint8_t* pBlockData = &uiDevice->disBuffer[page][startColAddr];
              uint16_t dataLength = (endCol - startCol + 1) * 8;
              HAL_I2C_Mem_Write_DMA(&SSD1306I2cHandle, SSD1306Address, Data, I2C_MEMADD_SIZE_8BIT, pBlockData, dataLength);
              xSemaphoreTake(sUiDevice.dmaIdle, portMAX_DELAY); // 等待DMA完成
  
              // 清除已处理的块标记
              uint16_t clearMask = ((1 << (endCol - startCol + 1)) - 1) << startCol; // 清除连续标记位
              pageMark &= ~clearMask; // 更新 pageMark，清除已处理标记
          }
  
          // 清除 `markBlock` 的对应标记位
          sUiDevice.markBlock[page * 2] = 0;     // 清零低8位
          sUiDevice.markBlock[page * 2 + 1] = 0; // 清零高8位
      }
  }
  /**
    *@ FunctionName: void SSDPower(DisplayState cmd)
    *@ Author: CzrTuringB
    *@ Brief: 打开\关闭Oled的显示
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDPower(DisplayState cmd)
  {
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  }
  /**
    *@ FunctionName: void SSDClean(void)
    *@ Author: CzrTuringB
    *@ Brief: 清空OLED的显示
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDClean(void)
  {
  	uint8_t temp;
  	if (uiDevice->dataValid == False)
  	{
  		temp = 0x00;
  	}
  	else
  	{
  		temp = 0xFF;
  	}
  	for(uint8_t x = 0; x<128 ; x++)
  	{
  		for(uint8_t y = 0; y<8 ; y++)
  		{
  			uiDevice->disBuffer[y][x] = temp;
  		}
  	}
  }
  /**
    *@ FunctionName: void SSDClkConfig(uint8_t divide,uint8_t fosc)
    *@ Author: CzrTuringB
    *@ Brief:	配置SSD1306芯片的时钟
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、设置时钟分频系数Divide，其范围是：0xX0~0xXF
    *@ 	2、设置时钟频率Fosc，其范围是：0x0X~0xFX
    *@ 	3、一般只有在初始化的时候调用这个函数，并且其配置为：Divide=0x00,Fosc=0x80
    */
  void SSDClkConfig(uint8_t divide,uint8_t fosc)
  {
  	uint8_t result = (0x0F&divide)|(0xF0&fosc);
  	uint8_t cmd = 0xD5;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &result);
  }
  /**
    *@ FunctionName: void SSDSetMux(uint8_t mux)
    *@ Author: CzrTuringB
    *@ Brief:	设置显示映射到显存的范围,即有显存中有多少行数据映射到显示器上
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、设置显示范围Mux，其范围是：0x00~0x3F
    *@ 	2、显示范围对应的是RAM行，设置后RAM行范围：0~Mux
    */
  void SSDSetMux(uint8_t mux)
  {
  	uint8_t cmd = 0xA8;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &mux);
  }
  /**
    *@ FunctionName: SSDSetOffset(uint8_t offset)
    *@ Author: CzrTuringB
    *@ Brief:	设置显示的哪个行是起始行
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、偏移量Offset，其范围是：0x00~0x3F
    *@ 	2、偏移量指的是COM0对应的那个Offset显示行
    */
  void SSDSetOffset(uint8_t offset)
  {
  	uint8_t cmd = 0xD3;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &offset);
  }
  /**
    *@ FunctionName: void SSDSetStartline(uint8_t startline)
    *@ Author: CzrTuringB
    *@ Brief: 设置显示的起始行对应显存中的第几行
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、起始行Startline，其范围是0x40~0x7F
    *@ 	2、显示起始线指的是Row0对应的那个RamRow
    */
  void SSDSetStartline(uint8_t startline)
  {
  	QueueInElement(&sUiDevice.cmdQueue, &startline);
  }
  /**
    *@ FunctionName: void SSDColConfig(ColumnMap mode)
    *@ Author: CzrTuringB
    *@ Brief:	设置列Cloumn与段Segment的映射模式
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、设置显示列与显存列的对应情况
    *@ 	2、顺序：显示列0对应显存列0
    *@ 	3、逆序：显示列0对应显存列127
    */
  void SSDColConfig(ColumnMap mode)
  {
  	QueueInElement(&sUiDevice.cmdQueue, &mode);
  }
  /**
    *@ FunctionName: void SSDComScan(ComScan mode)
    *@ Author: CzrTuringB
    *@ Brief: 设置COM的扫描方式
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDComScan(ComScan mode)
  {
  	QueueInElement(&sUiDevice.cmdQueue, &mode);
  }
  /**
    *@ FunctionName: void SSDComConfig(ComConfig mode)
    *@ Author: CzrTuringB
    *@ Brief: SSD的COM与引脚的映射配置
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDComConfig(ComConfig mode)
  {
  	uint8_t cmd = 0xDA;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &mode);
  }
  /**
    *@ FunctionName: void SSDSetContrast(uint8_t contrast)
    *@ Author: CzrTuringB
    *@ Brief:	设置显示对比度
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、设置对比度contrast，其范围是：0x00~0xff
    */
  void SSDSetContrast(uint8_t contrast)
  {
  	uint8_t cmd = 0x81;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &contrast);
  }
  /**
    *@ FunctionName: void SSDPreChargeConfig(uint8_t phase1,uint8_t phase2)
    *@ Author: CzrTuringB
    *@ Brief:	配置SSD提前充电周期
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、Phase1：表示阶段1的充电周期，其范围是0xX0~0xXF
    *@ 	2、Phase2：表示阶段2的充电周期，其范围是0x0X~0XFX
    */
  void SSDPreChargeConfig(uint8_t phase1,uint8_t phase2)
  {
  	uint8_t result = (0x0F&phase1)|(0xF0&phase2);
  	uint8_t cmd = 0xD9;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &result);
  }
  /**
    *@ FunctionName: void SSDComhConfig(uint8_t level)
    *@ Author: CzrTuringB
    *@ Brief: 配置Comh引脚取消选择电压等级
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、等级：0x00 0x20 0x30
    */
  void SSDComhConfig(uint8_t level)
  {
  	uint8_t cmd = 0xDB;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &level);
  }
  /**
    *@ FunctionName: void SSDSetHold(EnDis holdOn)
    *@ Author: CzrTuringB
    *@ Brief: 设置屏幕显示是否保持不变
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、输入：Enale or Disable
    */
  void SSDSetHold(EnDis holdOn)
  {
  	uint8_t cmd;
  	if(holdOn == Enable)
  	{
  		cmd = 0xA5;
  		//保持屏幕内容不变，显示输出不跟随显存
  		QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	}
  	else
  	{
  		cmd = 0xA4;
  		//显示输出跟随显存
  		QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	}
  }
  /**
    *@ FunctionName: void SSDSetReverse(EnDis reverse)
    *@ Author: CzrTuringB
    *@ Brief: 设置屏幕翻转显示
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、输入：Enale or Disable
    */
  void SSDSetReverse(EnDis reverse)
  {
  	if(reverse == Enable)
  	{
  		//0 表示显示	1表示不显示
  		uint8_t cmd = 0xA7;
  		uiDevice->dataValid = True;
  		QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	}
  	else
  	{
  		//1 表示显示	0表示不显示
  		uint8_t cmd = 0xA6;
  		uiDevice->dataValid = False;
  		QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	}
  }
  /**
    *@ FunctionName: void SSDPreChargePump(void)
    *@ Author: CzrTuringB
    *@ Brief:	使能充电调节
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void SSDChargePump(void)
  {
  	uint8_t cmd = 0x8D;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	cmd = 0x14;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  }
  /**
    *@ FunctionName: void SSDAddressConfig(AddressMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 配置寻址模式
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、除非特别设置，默认都为水平寻址模式
    */
  void SSDAddressConfig(AddressMode mode)
  {
  	uint8_t cmd = 0x20;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	QueueInElement(&sUiDevice.cmdQueue, &mode);
  }
  /**
    *@ FunctionName: void SSDSetSEColumnAddress(uint8_t columnStart,uint8_t columnEnd)
    *@ Author: CzrTuringB
    *@ Brief:	设置起始列地址和终止列地址
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、数据范围：0~127
    *@ 	2、适用于非页寻址模式
    */
  void SSDSetColumnAddress(uint8_t columnStart,uint8_t columnEnd)
  {
  	uint8_t cmd = 0x21;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	columnStart = 0x7F & columnStart;
  	QueueInElement(&sUiDevice.cmdQueue, &columnStart);
  	columnEnd = 0x7F & columnEnd;
  	QueueInElement(&sUiDevice.cmdQueue, &columnEnd);
  }
  /**
    *@ FunctionName: void SSDSetSEPageAddress(uint8_t PageStart,uint8_t PageEnd)
    *@ Author: CzrTuringB
    *@ Brief:	设置起始页地址和终止页地址
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ 	1、数据范围：0~7
    *@ 	2、适用于非页寻址模式
    */
  void SSDSetPageAddress(uint8_t pageStart,uint8_t pageEnd)
  {
  	uint8_t cmd = 0x22;
  	QueueInElement(&sUiDevice.cmdQueue, &cmd);
  	pageStart = 0x07 & pageStart;
  	QueueInElement(&sUiDevice.cmdQueue, &pageStart);
  	pageEnd = 0x07 & pageEnd;
  	QueueInElement(&sUiDevice.cmdQueue, &pageEnd);
  }
  /**
    *@ FunctionName: void HAL_I2C_MemTxCpltCallback(I2C_HandleTypeDef *hi2c)
    *@ Author: CzrTuringB
    *@ Brief: IIC+DMA发送完成后产生的回调函数
    *@ Time: Sep 20, 2024
    *@ Requirement：
    */
  void HAL_I2C_MemTxCpltCallback(I2C_HandleTypeDef *hi2c)
  {
  	//当DMA数据发送完后，发送DMA空闲信号表明DMA处于空闲状态
  	BaseType_t xHigherPriorityTaskWoken;
  	xHigherPriorityTaskWoken = pdFALSE;
  	xSemaphoreGiveFromISR(sUiDevice.dmaIdle,&xHigherPriorityTaskWoken);
  	portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
  }
  /**
    *@ FunctionName: void SSDUpdateScreen(void)
    *@ Author: CzrTuringB
    *@ Brief: 屏幕刷新函数
    *@ Time: Dec 23, 2024
    *@ Requirement：
    */
  void SSDInit(void)
  {
  	//创建DMA空闲信号量
  	sUiDevice.dmaIdle = xSemaphoreCreateBinary();
  	//初始化命令缓冲区
  	QueueInit(&sUiDevice.cmdQueue, sizeof(uint8_t), cmdBufferSize, cmdBuffer);
  	SSDPower(DisplayOff);
  	SSDClkConfig(0x00, 0x70);
  	SSDSetMux(0x3F);
  	SSDSetOffset(0x00);
  	SSDSetStartline(0x40);
  	SSDColConfig(MapReverse);
  	SSDComScan(ScanOrder);
  	SSDComConfig(AnLR);
  	SSDSetContrast(0xCF);
  	SSDPreChargeConfig(0x01, 0xE0);
  	SSDComhConfig(0x30);
  	SSDSetHold(Disable);
  	SSDSetReverse(Disable);
  	SSDAddressConfig(HorizontalMode);
  	SSDWriteCmds();
  	SSDClean();
  	SSDWriteDataAll();
  	SSDChargePump();
  	SSDPower(DisplayOn);
  	SSDWriteCmds();
  }
  ```

- SSD1306.h：

  ```c
  /**
    *@ FileName: Ssd1306.h
    *@ Author: CzrTuringB
    *@ Brief:	IIC接口的SSD1306头文件
    *@ Time: Sep 20, 2024
    *@ Requirement：
    *@ Note:
    *@ 	1、PinCom表示引脚行，一共64个
    *@ 	2、RamRow表示显存行，一共64个
    *@ 	3、Row表示显示行，一共64个
    *@ 	4、Segment表示显示列
    *@ 	5、RamClo表示显存列
    */
  #ifndef SSD1306_H_
  #define SSD1306_H_
  /* Includes ------------------------------------------------------------------*/
  #include "i2c.h"
  #include "crc.h"
  /* Data(对外接口)-----------------------------------------------------*/
  //-define
  #define ScreenWidth		128			//屏幕宽度
  #define ScreenHeight	64			//屏幕高度
  #define PageNumber		8			//页数量
  //-typedef(类型重命名)
  typedef uint8_t (*SSDBuffer)[ScreenWidth];  //SSDBuffer为指向128列数组的指针类型
  //-struct(结构体对外接口)
  typedef struct
  {
  	uint8_t disBuffer[PageNumber][ScreenWidth];		//屏幕显示缓冲数组
  	uint8_t maskBuffer[PageNumber][ScreenWidth];	//蒙版
  	Bool dataValid; 								//数据有效性判断变量
  }SsdPublicConfig;
  extern SsdPublicConfig* uiDevice;
  /* Functions(对外接口函数)------------------------------------------------------------------*/
  void SSDInit(void);
  void SSDUpdateScreen(void);
  void SSDClean(void);
  #endif
  ```

### 第四章	常见SSD1306问题及其处理方法

- 问题一：手机拍摄OLED屏幕存在波纹现象
  - 解决方法：[用上画点函数和蒙版，滚动显示更丝滑一些了^_^_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1ZZ421W7rb/?spm_id_from=333.337.search-card.all.click&vd_source=c706c24a5bec5f2b0d006d2505e30519)
- 问题二：播放动画时候屏幕不停闪烁
  1. 不要在循环中频繁调用clean函数并把缓存中的数据发送给屏幕
  2. 屏幕刷新间隔过长，检查SSD1306驱动函数中数据发送函数的编写，使用u8g2库的话，不要用普通I2C，用快速I2C。

----

## 参考

[^1]: IIC官方参考手册[UM10204.pdf (nxp.com)](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
[^2]:[SSD1306-Revision 1.1 (Charge Pump).pdf](https://github.com/ChenZR0509/TuringUi/blob/main/Library/SSD1306-Revision 1.1 (Charge Pump).pdf)
[^3]:[C语言 空指针和void指针_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1cM411z7Bp/?spm_id_from=333.337.search-card.all.click&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^4]:[计算机网络微课堂第023讲 差错检测（有字幕无背景音乐版）_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1xJ411K7Wx/?spm_id_from=333.337.search-card.all.click&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^5]:[【计算机网络期末复习】5分钟左右让你明白CRC循环冗余校验_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1rJ411V7L5/?spm_id_from=333.337.search-card.all.click&vd_source=c706c24a5bec5f2b0d006d2505e30519)