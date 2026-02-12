---
title: TuringUi系列(二)
categories:
  - 项目
  - TuringUi
tags:
  - 计算机图形学
abbrlink: 10692
date: 2024-12-18 08:41:00

---

# TuringUi系列(二)

----

## 第一部分	前提

- OLED像素屏幕是128×64的，因此创建了一个8×128的二维数组，数组中每个字节表示SSD1306一页中的某一列。
- OLED显示可以配置为数据1有效和数据0有效，因此在绘制图形时需要考虑有效数据类型。
- 坐标系定义：坐标系原点为Oled左下角像素，向上为y轴，向右为x轴。

## 第二部分	相关元素的绘制

### 第一章	模拟灰度[^1]

- **灰度阶（Gray Level）** 是指图像中不同灰度级别的数量。在图像处理中，灰度阶通常用于描述图像中每个像素的亮度值的离散化程度，或者说是图像中不同亮度的层级数。每个像素的灰度值表示该像素的亮度强度，通常以0到255的整数来表示，其中0代表黑色，255代表白色，0到255之间的其他值表示不同程度的灰色。

- Oled实现灰度阶原理：以2×2为一个单位，对其进行填充可以有十六种填充模式如下图所示，依据其填充比例，可将其划分为：0%、25%、50%、75%、100%五个等级对于整个屏幕填充的图像而言就会形成五个等级的灰度变化，同等级的灰度间的切换就会造成屏幕的抖动效果。

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/005.png)

- 基于上述，原理可以设计出一下填充模式，用户在绘制时可以通过相关形参自行设置：

  ```c
  typedef enum
  {
  	Fill0,	//填充0：灰度为0%
  	Fill1,	//填充1：灰度为100%
  	Fill5,	//交替填充0、1：灰度为50%
  	FillA,	//交替填充1、0：灰度为50%
  	FillB,  	//灰度为75%
  	Fill7,		//灰度为75%
  	FillE,		//灰度为75%
  	FillD,		//灰度为75%
  	FillEBD7,	//灰度为75%：四种75%灰度交替填充
  }FillMode;
  ```


### 第二章	点的绘制与点数据的读取

#### 第一节	代码

- 点的结构体：**OLED屏幕的像素是128×64的，所以以int8_t来实现，当点坐标小于零时，通过边界检测，以取消绘制**

  ```c
  //二维坐标点
  typedef struct
  {
      int8_t x, y;
  }Point2D;
  ```

- 点的边界检测：

  ```c
  /**
    *@ FunctionName: Bool EdgeDetect(Point2D p)
    *@ Author: CzrTuringB
    *@ Brief: 待绘制点有效性检测
    *@ Time: Dec 24, 2024
    *@ Requirement：
    *@ 	1、超出屏幕范围返回True;
    *@ 	2、没有超出屏幕范围返回False;
    */
  Bool EdgeDetect(Point2D p)
  {
  	if(p.x < 0 || p.y < 0) return True;
  	if(p.y >= ScreenHeight) return True;
  	return False;
  }
  ```

- 点的绘制：

  ```c
  /**
    *@ FunctionName: void PlotPoint(Point2D p,SSDBuffer buffer,FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 根据P点x,y坐标绘制点
    *@ Time: Oct 30, 2024
    *@ Requirement：
    *@ 	1、坐标系：
    *@ 		y63
    *@		 |
    *@		 |
    *@		 |
    *@	   y0/x0-------------y127
    *@	2、buffer为一个指向屏幕显示缓冲区或蒙版缓冲区的二维数组的地址
    */
  void PlotPoint(Point2D p,SSDBuffer buffer,FillMode mode)
  {
  	//超出屏幕边界
  	if(EdgeDetect(p) == True)	return;
  	uint8_t index = p.y / 8;
  	uint8_t shift = p.y % 8;
  	switch(mode)
  	{
  		case Fill0:
  			buffer[index][p.x] &= (~((1 << shift)));
  			break;
  		case Fill1:
  			buffer[index][p.x] |= (1 << shift);
  			break;
  		case Fill5:
  			if(p.x%2 == 0)	buffer[index][p.x] |= (0xAA & (1 << shift));
  			else			buffer[index][p.x] |= (0x55 & (1 << shift));
  			break;
  		case FillA:
  			if(p.x%2 == 0)	buffer[index][p.x] |= (0x55 & (1 << shift));
  			else			buffer[index][p.x] |= (0xAA & (1 << shift));
  			break;
  		case FillE:
  			if(p.x%2 != 0)	buffer[index][p.x] |= ((1 << shift));
  			else			buffer[index][p.x] |= (0xAA & (1 << shift));
  			break;
  		case FillB:
  			if(p.x%2 != 0)	buffer[index][p.x] |= (1 << shift);
  			else			buffer[index][p.x] |= (0x55 & (1 << shift));
  			break;
  		case FillD:
  			if(p.x%2 == 0)	buffer[index][p.x] |= ((1 << shift));
  			else			buffer[index][p.x] |= (0xAA & (1 << shift));
  			break;
  		case Fill7:
  			if(p.x%2 == 0)	buffer[index][p.x] |= (1 << shift);
  			else			buffer[index][p.x] |= (0xAA & (1 << shift));
  			break;
  		case FillEBD7:
  			if(p.x%4 == 0)	buffer[index][p.x] |= (0xEE & (1 << shift));
  			if(p.x%4 == 1)	buffer[index][p.x] |= (0xBB & (1 << shift));
  			if(p.x%4 == 2)	buffer[index][p.x] |= (0xDD & (1 << shift));
  			if(p.x%4 == 3)	buffer[index][p.x] |= (0x77 & (1 << shift));
  			break;
  	}
  }
  ```

- 读取坐标点存储的元素：

  ```c
  /**
    *@ FunctionName: uint8_t GetPoint(uint8_t x, uint8_t y)
    *@ Author: CzrTuringB
    *@ Brief: 读取显存内容
    *@ Time: Dec 17, 2024
    *@ Requirement：
    *@ 	1、如果返回值非零则为1
    *@ 	2、如果返回值为零则为0
    */
  uint8_t GetPoint(Point2D p,SSDBuffer buffer)
  {
  	//超出屏幕边界
  	if(EdgeDetect(p) == True)	return 0;
  	uint8_t index = p.y / 8;
  	uint8_t shift = p.y % 8;
  	return (buffer[index][p.x] & (1 << shift)) ? 1 : 0;
  }
  ```

#### 第二节	说明

- 点的绘制：已知想要在(x,y)坐标上绘制一个像素点，但是我们的显存存储方式是一个8×128的二维数组，因此对于x坐标而言其直接对应数组索引的第二个数字，对于y坐标而言，我们需要计算其所在页，以及所在页的字节的第几位，至此我们便可以定位点在二维数组中的位置。使用或运算与取反运算的组合即可实现点的绘制。
- 点数据的读取，已知想要读取(x,y)坐标点所存储的数据，根据上述计算，即可定位点在二维数组的未知，并通过与运算消去其他无关位数据，若结果为零则点数据为零，反之则为1。

### 第三章	任意角度直线的绘制

#### 第一节	代码

- 任意角度直线绘制：

  ```c
  /**
    *@ FunctionName: void PlotLine(Point2D p0, Point2D p1, SSDBuffer buffer,FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 根据起始点\结束点坐标坐标绘制直线
    *@ Time: Oct 30, 2024
    *@ Requirement：
    */
  void PlotLine(Point2D p0, Point2D p1, SSDBuffer buffer,FillMode mode)
  {
  	//超出屏幕边界
  	if(EdgeDetect(p0) == True || EdgeDetect(p1) == True)	return;
  	//直线在x和y方向上的变化
      int8_t dx = abs(p1.x - p0.x);
      int8_t dy = abs(p1.y - p0.y);
      //步进方向
      int8_t sx = (p0.x < p1.x) ? 1 : -1;
      int8_t sy = (p0.y < p1.y) ? 1 : -1;
      //绘制直线、
      if (dx > dy)
      {
          //当 dx > dy 时，x 轴是主轴
      	int8_t err = dx / 2;
          while (p0.x != p1.x)
          {
          	PlotPoint(p0,buffer,mode);
              err -= dy;
              if (err < 0)
              {
              	p0.y += sy;
                  err += dx;
              }
              p0.x += sx;
          }
      }
      else
      {
          //当 dy > dx 时，y 轴是主轴
      	int8_t err = dy / 2;
          while (p0.y != p1.y)
          {
          	PlotPoint(p0,buffer,mode);
              err -= dx;
              if (err < 0)
              {
              	p0.x += sx;
                  err += dy;
              }
              p0.y += sy;
          }
      }
      //绘制终点
      PlotPoint(p1,buffer,mode);
  }
  ```

- 任意角度虚线的绘制：

  ```c
  /**
    *@ FunctionName: void PlotDashedLine(Point2D p0, Point2D p1, SSDBuffer buffer, uint8_t dashLength, uint8_t gapLength, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 根据起始点\结束点坐标坐标绘制虚线
    *@ Time: Oct 30, 2024
    *@ Requirement：
    *@ 	1、dashLength表示虚线线段长度
    *@ 	2、gapLength表示虚线间隔长度
    */
  void PlotDashedLine(Point2D p0, Point2D p1, SSDBuffer buffer, uint8_t dashLength, uint8_t gapLength, FillMode mode)
  {
  	//超出屏幕边界
  	if(EdgeDetect(p0) == True || EdgeDetect(p1) == True)	return;
      // 计算直线在 x 和 y 方向上的变化
      int16_t dx = abs(p1.x - p0.x);
      int16_t dy = abs(p1.y - p0.y);
  
      // 确定步进方向
      int8_t sx = (p0.x < p1.x) ? 1 : -1;
      int8_t sy = (p0.y < p1.y) ? 1 : -1;
  
      // 初始化误差项
      int16_t err = (dx > dy ? dx : -dy) / 2;
      int16_t err2;
  
      // 初始化虚线绘制状态
      uint8_t segmentLength = dashLength + gapLength;  // 总周期长度
      uint8_t stepCount = 0;                           // 当前步数计数
  
      // 循环绘制虚线
      while (1)
      {
          // 判断当前步是否在虚线段内
          if (stepCount < dashLength)
          {
              PlotPoint(p0,buffer,mode); // 绘制点
          }
  
          // 更新步数计数
          stepCount = (stepCount + 1) % segmentLength;
  
          // 到达终点时退出
          if (p0.x == p1.x && p0.y == p1.y)
              break;
  
          // 更新误差和坐标
          err2 = err;
          if (err2 > -dx)
          {
              err -= dy;
              p0.x += sx;
          }
          if (err2 < dy)
          {
              err += dx;
              p0.y += sy;
          }
      }
  }
  ```

- 箭头直线的绘制：**用于绘制带箭头的线条**

  ```c
  /**
    *@ FunctionName: void PlotArrow(Point2D p0, Point2D p1, SSDBuffer buffer, ArrowMode aMode, FillMode fMode)
    *@ Author: CzrTuringB
    *@ Brief: 根据起始点\结束点坐标坐标绘制箭头
    *@ Time: Oct 30, 2024
    *@ Requirement：
    */
  void PlotArrow(Point2D p0, Point2D p1, SSDBuffer buffer, ArrowMode aMode, FillMode fMode)
  {
      // 如果起点或终点超出屏幕边界，直接返回
      if (EdgeDetect(p0) == True || EdgeDetect(p1) == True) return;
  
      // 内部函数：绘制箭头
      void DrawArrow(Point2D base, Point2D tip)
      {
          // 计算箭头方向的角度（弧度制）
          float angle = atan2(tip.y - base.y, tip.x - base.x);
  
          // 计算箭头的两条边端点
          int8_t x1 = tip.x + (int8_t)(3 * cos(angle + M_PI - 0.5)); // 箭头左边
          int8_t y1 = tip.y + (int8_t)(3 * sin(angle + M_PI - 0.5));
          int8_t x2 = tip.x + (int8_t)(3 * cos(angle + M_PI + 0.5)); // 箭头右边
          int8_t y2 = tip.y + (int8_t)(3 * sin(angle + M_PI + 0.5));
  
          // 绘制箭头的两条边
          PlotLine((Point2D){x1, y1}, tip, buffer, fMode);
          PlotLine((Point2D){x2, y2}, tip, buffer, fMode);
      }
  
      // 根据箭头模式绘制箭头
      switch (aMode)
      {
          case StartArrow:
              // 在起点绘制箭头
              DrawArrow(p1, p0);
              break;
          case EndArrow:
              // 在终点绘制箭头
              DrawArrow(p0, p1);
              break;
          case StaEndArrow:
              // 在起点和终点都绘制箭头
              DrawArrow(p1, p0);
              DrawArrow(p0, p1);
              break;
          default:
              // 如果是无效模式，不绘制箭头
              break;
      }
  }
  ```

#### 第二节	说明[^2]

- 算法名称：Bresenham直线绘制算法

- 已知条件：起始点坐标$$(x_0,y_0)$$和结束点坐标$$(x_1,y_1)$$，对于这两个点并没有多余的限制。

- 直线的数学模型：
  $$
  y=kx+b
  $$

  - k表示直线的斜率，其可以通过起点坐标和终点坐标计算出来。

  - 直线的斜率决定了直线的倾斜程度的大小，斜率的绝对值小于1则说明越靠近x轴，反之则越靠近y轴。如下图所示：

    ![001](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/001.png)

  - 由于像素点的离散特性，所以我们绘制直线的时候就是以一个方向为标准(x轴方向或y轴方向)然后每画一个点就在这个方向上自增或自减，另一个方向则需要根据实际情况去判断他是保持不变还是增加、减小。于是我们引入一个误差变量$$Error=离散值-通过模型计算出的连续值$$以此大小来判断另一个方向坐标的变换。为了减小误差对于最终结果的影响，因此需要根据斜率去判断来选取自增或自减方向。

- 以斜率小于1解释误差的更新思想：

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/002.png)

  ​	当斜率小于1时x是主轴，每次迭代递增x（假设起点在左下角，终点在右上角）并根据误差判断是否自增y的值或不变。从上图可以看出，自增y和不自增y会与原先的连续模型都有一个误差d，判断误差d1和d2的大小即可确定是否递增y。

  ​	为了避免浮点数参与运算，以累计误差来控制y的步进，以上图为列：
  $$
  \text{判断依据：}d2-d1?0
  $$
  $$
  Error=d2-d1=y_k+1-\left| \frac{dy}{dx} \right|\left( x_{k+1} \right) +b-\left| \frac{dy}{dx} \right|\left( x_{k+1} \right) -b+y_k
  $$
  $$
  Error=2y_k+1-2\left| \frac{dy}{dx} \right|\left( x_{k+1} \right) 
  $$
  $$
  Error=y_k+\frac{1}{2}-\left| \frac{dy}{dx} \right|\left( x_{k+1} \right) 
  $$
  $$
  \text{为了避免浮点数运算，在不影响大小符号的变换下不等式两边乘}dx
  $$
  $$
  Error=\frac{1}{2}\left| dx \right|-\left| dy \right|+y_k\left| dx \right|-x_k\left| dy \right|
  $$

  ​	选择$$Error=\frac{1}{2}\left| dx \right|-\left| dy \right|$$为初值，进行误差迭代更新，若Error小于零则y方向自增一个像素单位，然后按照下式更新误差：
  $$
  Error_{k+1}=Error_k+\left| dx \right|
  $$
  ​	若Error大于零则y方向不自增，然后按照下式更新误差：
  $$
  Error_{k+1}=Error_k-\left| dy \right|
  $$

### 第四章	圆形的绘制

#### 第一节	代码

- 算法名称：Bresenham圆形绘制算法[^3][^4]

- 圆形的绘制：

  ```c
  /**
    *@ FunctionName: void PlotCircle(Point2D pc, uint8_t r, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 根据起始点坐标、半径绘制圆
    *@ Time: Oct 30, 2024
    *@ Requirement：
    */
  void PlotCircle(Point2D pc, uint8_t r, SSDBuffer buffer, FillMode mode)
  {
  	uint8_t x = 0, y = r;
  	int8_t d = 1 - r;
      // 使用八分之一圆对称绘制
      while (x <= y)
      {
          // 绘制八个对称点
      	PlotPoint((Point2D){pc.x + x, pc.y + y}, buffer, mode);
      	PlotPoint((Point2D){pc.x - x, pc.y + y}, buffer, mode);
      	PlotPoint((Point2D){pc.x + x, pc.y - y}, buffer, mode);
      	PlotPoint((Point2D){pc.x - x, pc.y - y}, buffer, mode);
      	PlotPoint((Point2D){pc.x + y, pc.y + x}, buffer, mode);
      	PlotPoint((Point2D){pc.x - y, pc.y + x}, buffer, mode);
      	PlotPoint((Point2D){pc.x + y, pc.y - x}, buffer, mode);
      	PlotPoint((Point2D){pc.x - y, pc.y - x}, buffer, mode);
          // 更新参数，确定下一个点
          if (d < 0)
          {
              d = d + 2 * x + 3;
          }
          else
          {
              d = d + 2 * (x - y) + 5;
              y--;
          }
          x++;
      }
  }
  ```

- 填充圆形的绘制：

  ```c
  /**
    *@ FunctionName: void PlotFillCircle(Point2D pc, uint8_t r, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制填充圆
    *@ Time: Dec 20, 2024
    *@ Requirement：
    */
  void PlotFillCircle(Point2D pc, uint8_t r, SSDBuffer buffer, FillMode mode)
  {
  	uint8_t x = 0, y = r;
  	int8_t d = 1 - r;
  
      // 使用八分之一圆对称绘制
      while (x <= y)
      {
      	PlotLine((Point2D){pc.x - x, pc.y + y}, (Point2D){pc.x + x, pc.y + y}, buffer, mode);
      	PlotLine((Point2D){pc.x - x, pc.y - y}, (Point2D){pc.x + x, pc.y - y}, buffer, mode);
      	PlotLine((Point2D){pc.x - y, pc.y + x}, (Point2D){pc.x + y, pc.y + x}, buffer, mode);
      	PlotLine((Point2D){pc.x - y, pc.y - x}, (Point2D){pc.x + y, pc.y - x}, buffer, mode);
          // 更新参数，确定下一个点
          if (d < 0)
          {
              d = d + 2 * x + 3;
          }
          else
          {
              d = d + 2 * (x - y) + 5;
              y--;
          }
          x++;
      }
  }
  ```

#### 第二节	说明

- 算法名称：Bresenham中点圆生成算法

- 如下图所示：

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/003.png)

  - 已知圆的方程如下所示，直接使用这个方程涉及到求方根运算和浮点数运算，因此需要对其进行简化分析，由于像素点的离散特性，因此算法需要判断下一个像素点是沿x方向移动还是同时沿x,y方向移动【假设每次迭代x都要自增，而y是否自增则要根据相关参数进行判定】。
    $$
    \left( x-x_c \right) ^2-\left( y-y_c \right) ^2=r^2
    $$

  - 由于圆具有八分对称特性，即可以把圆平均分成八份，这样就可以在计算出一个点的同时，计算出其余七个点，只需要改变坐标位置和符号即可：
    $$
    \left( x,y \right) \ \left( -x,y \right) \ \left( x,-y \right) \ \left( -x,-y \right) \ \left( y,x \right) \,\,\left( -y,x \right) \,\,\left( y,-x \right) \,\,\left( -y,-x \right)
    $$

  - 假设第一次迭代开始值为(xc，yc+r)即圆心水平线与圆的交点，由于像素的离散特性因此需要根据这个点的位置为于圆上、圆内、圆外来决定下一个像素点坐标的递增与否。
    $$
    \left( a-x_c \right) ^2+\left( b-y_c \right) ^2-r^2\left\{ \begin{array}{l}
    	>0\ \text{圆外\ 下一个像素要靠近圆心\ }\\
    	=0\ \text{圆上}\\
    	<0\ \text{圆内\ 下一个像素要远离圆心}\\
    \end{array} \right.
    $$

  - 如下图点m所示，由于圆的边框是一个弧形，因此为了计算其与原点的离散距离，则以$${(x_i+1,y_i-0.5)}$$代入计算像素点与圆心之间的距离：

    ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/004.png)
    $$
    \text{以}\left( x_c+1,y_c+r-0.5 \right) \text{为起始点开始迭代}
    $$

    $$
    \left( x_c+1-x_c \right) ^2+\left( y_c+r-0.5-y_c \right) ^2-r^2=1.25-r\approx 1-r=d
    $$

  - 当d小于0时，像素点位于圆内，以$${(x_i+1,y_i)}$$的方式进行迭代时，则有误差d更新方程为：
    $$
    \left( x_i+1-x_c \right) ^2+\left( y_i-0.5-y_c \right) ^2-\left( x_i-x_c \right) ^2-\left( y_i-0.5-y_c \right) ^2=2x_i+3
    $$

    $$
    d_{i+1}=d_i+2x_i+3
    $$

  - 当d大于等于0时，像素点位于圆外或圆上，以$${(x_i+1,y_i-1)}$$的方式进行迭代时，则有误差d更新方程为：
    $$
    \left( x_i+1-x_c \right) ^2+\left( y_i-1-0.5-y_c \right) ^2-\left( x_i-x_c \right) ^2-\left( y_i-0.5-y_c \right) ^2=2x_i-2y_i+5
    $$

    $$
    d_{i+1}=d_i+2x_i-2y_i+5
    $$

### 第五章	任意角度圆弧的绘制

- 判断角度是否在设定范围内：

  ```c
  /**
    *@ FunctionName: int8_t inArcRange(int8_t angle, int8_t startAngle, int8_t endAngle)
    *@ Author: CzrTuringB
    *@ Brief: 判断角度是否在范围内
    *@ Time: Oct 30, 2024
    *@ Requirement：
    */
  int8_t inArcRange(int16_t angle, int16_t startAngle, int16_t endAngle)
  {
      // 将角度限制在 0~360
      angle = (angle + 360) % 360;
      startAngle = (startAngle + 360) % 360;
      endAngle = (endAngle + 360) % 360;
      // 判断当前角度是否在指定范围内
      if (startAngle < endAngle)
      {
          return angle >= startAngle && angle <= endAngle;
      }
      else
      {
          return angle >= startAngle || angle <= endAngle;  // 跨0度情况
      }
  }
  ```

- 任意角度圆弧的绘制：

  ```c
  /**
    *@ FunctionName: void PlotArc(Point2D pc, uint8_t r, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 根据起始点坐标、半径、起始角度和终止角度绘制圆弧
    *@ Time: Oct 30, 2024
    *@ Requirement：
    *@ 	1、采用角度制表示角度
    */
  void PlotArc(Point2D pc, uint8_t r, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode)
  {
  	uint8_t x = 0, y = r;
  	int16_t d = 1 - r;
  
      while (x <= y)
      {
          // 计算当前点的八个对称点的角度
          int angles[8] =
          {
              (int16_t)(atan2(y, x) * 180 / M_PI),
              (int16_t)(atan2(y, -x) * 180 / M_PI),
              (int16_t)(atan2(-y, x) * 180 / M_PI),
              (int16_t)(atan2(-y, -x) * 180 / M_PI),
              (int16_t)(atan2(x, y) * 180 / M_PI),
              (int16_t)(atan2(x, -y) * 180 / M_PI),
              (int16_t)(atan2(-x, y) * 180 / M_PI),
              (int16_t)(atan2(-x, -y) * 180 / M_PI)
          };
          // 绘制每个符合角度条件的对称点
          if (inArcRange(angles[0], startAngle, endAngle)) PlotPoint((Point2D){pc.x + x, pc.y + y}, buffer, mode);
          if (inArcRange(angles[1], startAngle, endAngle)) PlotPoint((Point2D){pc.x - x, pc.y + y}, buffer, mode);
          if (inArcRange(angles[2], startAngle, endAngle)) PlotPoint((Point2D){pc.x + x, pc.y - y}, buffer, mode);
          if (inArcRange(angles[3], startAngle, endAngle)) PlotPoint((Point2D){pc.x - x, pc.y - y}, buffer, mode);
          if (inArcRange(angles[4], startAngle, endAngle)) PlotPoint((Point2D){pc.x + y, pc.y + x}, buffer, mode);
          if (inArcRange(angles[5], startAngle, endAngle)) PlotPoint((Point2D){pc.x - y, pc.y + x}, buffer, mode);
          if (inArcRange(angles[6], startAngle, endAngle)) PlotPoint((Point2D){pc.x + y, pc.y - x}, buffer, mode);
          if (inArcRange(angles[7], startAngle, endAngle)) PlotPoint((Point2D){pc.x - y, pc.y - x}, buffer, mode);
  
          // 更新决策参数
          if (d < 0)
          {
              d = d + 2 * x + 3;
          }
          else
          {
              d = d + 2 * (x - y) + 5;
              y--;
          }
          x++;
      }
  }
  ```

- 填充圆弧的绘制：

  ```c
  /**
    *@ FunctionName: void PlotFillArc(Point2D pc, uint8_t r, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode, uint8_t Boundry)
    *@ Author: CzrTuringB
    *@ Brief: 填充圆弧
    *@ Time: Dec 17, 2024
    *@ Requirement：
    */
  void PlotFillArc(Point2D pc, uint8_t r, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode, uint8_t Boundry)
  {
  	uint8_t x, y;
  	if(Boundry == 1)
  	{
  		//绘制圆弧
  		PlotArc(pc, r, buffer, startAngle, endAngle, Fill1);
  	    //绘制圆边
  		PlotLine(pc, (Point2D){pc.x+r*cos(startAngle/180.0*M_PI), pc.y+r*sin(startAngle/180.0*M_PI)}, buffer, Fill1);
  		PlotLine(pc, (Point2D){pc.x+r*cos(endAngle/180.0*M_PI), pc.y+r*sin(endAngle/180.0*M_PI)}, buffer, Fill1);
  	}
  	else
  	{
  		//绘制圆弧
  		PlotArc(pc, r, buffer, startAngle, endAngle, Fill0);
  	    //绘制圆边
  		PlotLine(pc, (Point2D){pc.x+r*cos(startAngle/180.0*M_PI), pc.y+r*sin(startAngle/180.0*M_PI)}, buffer, Fill0);
  		PlotLine(pc, (Point2D){pc.x+r*cos(endAngle/180.0*M_PI), pc.y+r*sin(endAngle/180.0*M_PI)}, buffer, Fill0);
  	}
      //计算递归原点
      x = pc.x + 0.5*r*cos((startAngle+endAngle)/360.0*M_PI);
      y = pc.y + 0.5*r*sin((startAngle+endAngle)/360.0*M_PI);
      //递归填充
      PlotFillPolygon((Point2D){x, y}, buffer, mode, Boundry);
  }
  ```

### 第六章	圆角圆环的绘制

```c
/**
  *@ FunctionName: void PlotRingWithRoundedEnds(Point2D pc, uint8_t outerRadius, uint8_t innerRadius, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 绘制圆角圆环
  *@ Time: Jan 10, 2025
  *@ Requirement：
  */
void PlotRingWithRoundedEnds(Point2D pc, uint8_t outerRadius, uint8_t innerRadius, SSDBuffer buffer, int16_t startAngle, int16_t endAngle, FillMode mode)
{
    uint8_t ringThickness = outerRadius - innerRadius;
    uint8_t halfThickness = ringThickness / 2;
    // 绘制外圆弧
    PlotArc(pc, outerRadius, buffer, startAngle, endAngle, mode);
    // 绘制内圆弧
    PlotArc(pc, innerRadius, buffer, startAngle, endAngle, mode);
    // 起始半圆圆心计算
    Point2D startCapCenter = {
        .x = pc.x + (int16_t)((outerRadius - halfThickness) * cos(startAngle * M_PI / 180)),
        .y = pc.y + (int16_t)((outerRadius - halfThickness) * sin(startAngle * M_PI / 180))
    };
    // 结束半圆圆心计算
    Point2D endCapCenter = {
        .x = pc.x + (int16_t)((outerRadius - halfThickness) * cos(endAngle * M_PI / 180)),
        .y = pc.y + (int16_t)((outerRadius - halfThickness) * sin(endAngle * M_PI / 180))
    };
    // 绘制起始半圆
    PlotArc(startCapCenter, halfThickness, buffer, startAngle-180, startAngle, mode);
    // 绘制结束半圆
    PlotArc(endCapCenter, halfThickness, buffer, endAngle, endAngle+180, mode);
}
```

### 第七章	三角形的绘制

- 三角形绘制：

  ```c
  /**
    *@ FunctionName: void PlotTriangle(Point2D* p, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制三角形
    *@ Time: Dec 26, 2024
    *@ Requirement：
    */
  void PlotTriangle(Point2D* p, SSDBuffer buffer, FillMode mode)
  {
  	PlotLine(p[0], p[1], buffer, mode);
  	PlotLine(p[1], p[2], buffer, mode);
  	PlotLine(p[2], p[0], buffer, mode);
  }
  ```

- 填充三角形绘制：**扫描线填充算法**[^5]

  ```c
  /**
    *@ FunctionName: void PlotFillTriangle(Point2D* p, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制填充三角形
    *@ Time: Dec 26, 2024
    *@ Requirement：
    */
  void PlotFillTriangle(Point2D* p, SSDBuffer buffer, FillMode mode)
  {
      // 获取边界框
      int8_t minX = p[0].x, maxX = p[0].x, minY = p[0].y, maxY = p[0].y;
      for (uint8_t i = 1; i < 3; i++)  // 三角形只有三个顶点
      {
          if (p[i].x < minX) minX = p[i].x;
          if (p[i].x > maxX) maxX = p[i].x;
          if (p[i].y < minY) minY = p[i].y;
          if (p[i].y > maxY) maxY = p[i].y;
      }
  
      // 扫描线填充
      for (int8_t y = minY; y <= maxY; y++)
      {
          // 存储交点
          int8_t intersections[6], n = 0;  // 三角形最多3条边
          // 计算与每条边的交点
          for (uint8_t i = 0; i < 3; i++)
          {
              Point2D p1 = p[i];
              Point2D p2 = p[(i + 1) % 3];
  
              // 检查扫描线是否穿过当前边
              if (y >= fmin(p1.y, p2.y) && y <= fmax(p1.y, p2.y))
              {
                  // 如果边与扫描线重合（水平边），跳过
                  if (p1.y == p2.y)
                      continue;
  
                  // 计算交点 x 坐标，避免浮动误差
                  int16_t x = p1.x + (int16_t)(y - p1.y) * (p2.x - p1.x) / (p2.y - p1.y);
  
                  // 仅在 x 坐标在 minX 和 maxX 范围内时才记录交点
                  if (x >= minX && x <= maxX)
                  {
                      intersections[n++] = x;
                  }
              }
          }
  
          // 去重交点，避免重复交点的影响
          for (uint8_t i = 0; i < n; i++) {
              for (uint8_t j = i + 1; j < n; j++)
              {
                  if (intersections[i] == intersections[j])
                  {
                      // 将重复的交点移到数组的末尾并减少数量
                      for (uint8_t k = j; k < n - 1; k++)
                      {
                          intersections[k] = intersections[k + 1];
                      }
                      n--; // 更新交点数量
                      j--; // 检查下一个元素
                  }
              }
          }
  
          // 排序交点，确保从左到右
          for (uint8_t i = 0; i < n - 1; i++)
          {
              for (uint8_t j = i + 1; j < n; j++)
              {
                  if (intersections[i] > intersections[j])
                  {
                      uint8_t temp = intersections[i];
                      intersections[i] = intersections[j];
                      intersections[j] = temp;
                  }
              }
          }
  
          // 在交点之间填充区域
          for (uint8_t i = 0; i < n - 1; i += 2)
          {
              // 填充交点之间的区域
              for (uint8_t x = intersections[i]; x <= intersections[i + 1]; x++)
              {
                  PlotPoint((Point2D){x, y}, buffer, mode);
              }
          }
      }
  }
  ```

### 第八章	矩形的绘制

- 指定左下角点，长度、高度的绘制函数：**只能绘制水平或垂直的矩形**

  ```c
  /**
    *@ FunctionName: void PlotRectangle(Point2D p0, uint8_t width, uint8_t height, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制矩形
    *@ Time: Oct 30, 2024
    *@ Requirement：
    *@ 	1、起始点为矩形左下角
    */
  void PlotRectangle(Point2D p0, uint8_t width, uint8_t height, SSDBuffer buffer, FillMode mode)
  {
  	PlotLine(p0, (Point2D){p0.x + width, p0.y}, buffer, mode);
  	PlotLine((Point2D){p0.x + width , p0.y}, (Point2D){p0.x + width, p0.y + height}, buffer, mode);
  	PlotLine((Point2D){p0.x + width, p0.y + height}, (Point2D){p0.x, p0.y + height}, buffer, mode);
  	PlotLine((Point2D){p0.x, p0.y + height}, p0, buffer, mode);
  }
  ```

- 四边形的绘制：**根据四点坐标绘制四边形**

  ```c
  /**
    *@ FunctionName: void PlotQuadrilateral(Point2D* p, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制四边形
    *@ Time: Dec 25, 2024
    *@ Requirement：
    *@ 	1、四个点坐标要按照逆时针或顺时针方向来给
    */
  void PlotQuadrilateral(Point2D* p, SSDBuffer buffer, FillMode mode)
  {
  	PlotLine(p[0], p[1], buffer, mode);
  	PlotLine(p[1], p[2], buffer, mode);
  	PlotLine(p[2], p[3], buffer, mode);
  	PlotLine(p[3], p[0], buffer, mode);
  }
  ```

- 圆角矩形的绘制：

  ```c
  /**
    *@ FunctionName: void PlotArcRectangle(Point2D p0, uint8_t width, uint8_t height, uint8_t r, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制圆角矩形
    *@ Time: Oct 31, 2024
    *@ Requirement：
    *@	1、起始点为矩形左下角
    */
  void PlotArcRectangle(Point2D p0, uint8_t width, uint8_t height, uint8_t r, SSDBuffer buffer, FillMode mode)
  {
  	PlotLine((Point2D){p0.x + r, p0.y}, (Point2D){p0.x + width - r, p0.y}, buffer, mode);  // 顶边
  	PlotLine((Point2D){p0.x + r, p0.y + height}, (Point2D){p0.x + width - r, p0.y + height}, buffer, mode);  // 底边
  	PlotLine((Point2D){p0.x, p0.y + r}, (Point2D){p0.x,p0.y + height - r}, buffer, mode);  // 左边
  	PlotLine((Point2D){p0.x + width, p0.y + r}, (Point2D){p0.x + width, p0.y + height - r}, buffer, mode);  // 右边
  
  	PlotArc((Point2D){p0.x + r, p0.y + height - r}, r, buffer, 90, 180, mode);
  	PlotArc((Point2D){p0.x + r, p0.y + r}, r, buffer, 180, 270, mode);
  	PlotArc((Point2D){p0.x + width - r, p0.y + r}, r, buffer, 270, 360, mode);
  	PlotArc((Point2D){p0.x + width - r, p0.y + height - r}, r, buffer, 0, 90, mode);
  }
  ```

- 填充矩形的绘制：

  ```c
  /**
    *@ FunctionName: void PlotFillRectangle(uint8_t x0, uint8_t y0, uint8_t width, uint8_t height, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 绘制填充矩形
    *@ Time: Dec 17, 2024
    *@ Requirement：
    *@ 	1、起始点为矩形左下角
    */
  void PlotFillRectangle(Point2D p0, uint8_t width, uint8_t height, SSDBuffer buffer, FillMode mode)
  {
      //绘制填充矩形
      for (uint8_t y = p0.y; y <= p0.y + height; y++)
      {
      	PlotLine((Point2D){p0.x, y}, (Point2D){p0.x + width, y}, buffer, mode);
      }
  }
  ```

- 填充四边形的绘制：扫描线填充算法

  ```c
  /**
    *@ FunctionName: void PlotFillQuadrilateral(Point2D* p, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 填充四边形区域
    *@ Time: Dec 25, 2024
    *@ Requirement：
    *@ 	1、四个点坐标要按照逆时针或顺时针方向来给
    */
  void PlotFillQuadrilateral(Point2D* p, SSDBuffer buffer, FillMode mode)
  {
      // 获取边界框
      int8_t minX = p[0].x, maxX = p[0].x, minY = p[0].y, maxY = p[0].y;
      for (uint8_t i = 1; i < 4; i++)
      {
          if (p[i].x < minX) minX = p[i].x;
          if (p[i].x > maxX) maxX = p[i].x;
          if (p[i].y < minY) minY = p[i].y;
          if (p[i].y > maxY) maxY = p[i].y;
      }
  
      // 扫描线填充
      for (int8_t y = minY; y <= maxY; y++)
      {
          // 存储交点
          int8_t intersections[8], n = 0;
          // 计算与每条边的交点
          for (uint8_t i = 0; i < 4; i++)
          {
              Point2D p1 = p[i];
              Point2D p2 = p[(i + 1) % 4];
  
              // 检查扫描线是否穿过当前边
              if (y >= fmin(p1.y, p2.y) && y <= fmax(p1.y, p2.y))
              {
                  // 如果边与扫描线重合（水平边），跳过或添加该交点
                  if (p1.y == p2.y) // 水平边
                  {
                      continue;
                  }
                  // 计算交点 x 坐标，避免浮动误差
                  int16_t x = p1.x + (int16_t)(y - p1.y) * (p2.x - p1.x) / (p2.y - p1.y);
                  // 仅在 x 坐标在 minX 和 maxX 范围内时才记录交点
                  if (x >= minX && x <= maxX)
                  {
                      intersections[n++] = x;
                  }
              }
          }
          // 去重交点，避免重复交点的影响
          for (uint8_t i = 0; i < n; i++) {
              for (uint8_t j = i + 1; j < n; j++)
              {
                  if (intersections[i] == intersections[j])
                  {
                      // 将重复的交点移到数组的末尾并减少数量
                      for (uint8_t k = j; k < n - 1; k++)
                      {
                          intersections[k] = intersections[k + 1];
                      }
                      n--; // 更新交点数量
                      j--; // 检查下一个元素
                  }
              }
          }
          // 排序交点，确保从左到右
          for (uint8_t i = 0; i < n - 1; i++)
          {
              for (uint8_t j = i + 1; j < n; j++)
              {
                  if (intersections[i] > intersections[j])
                  {
                      uint8_t temp = intersections[i];
                      intersections[i] = intersections[j];
                      intersections[j] = temp;
                  }
              }
          }
          // 在交点之间填充区域
          for (uint8_t i = 0; i < n - 1; i += 2)
          {
              // 填充交点之间的区域
              for (uint8_t x = intersections[i]; x <= intersections[i + 1]; x++)
              {
                  PlotPoint((Point2D){x, y}, buffer, mode);
              }
          }
      }
  }
  ```

### 第九章	任意边数的正多边形绘制

#### 第一节	代码

```c
/**
  *@ FunctionName: void PlotPolygon(Point2D pc, uint8_t radius, uint8_t sides, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 绘制正多边形
  *@ Time: Oct 31, 2024
  *@ Requirement：
  */
void PlotPolygon(Point2D pc, uint8_t radius, uint8_t sides, SSDBuffer buffer, FillMode mode)
{
	//角度间隔
    double angleStep = 2 * M_PI / sides;
    int x[sides], y[sides];

    // 计算每个顶点的坐标
    for (int i = 0; i < sides; i++)
    {
        x[i] = pc.x + (int)(radius * cos(i * angleStep + 0.5 * M_PI) + 0.5);  // 四舍五入为整数
        y[i] = pc.y + (int)(radius * sin(i * angleStep + 0.5 * M_PI) + 0.5);  // 四舍五入为整数
    }

    // 绘制多边形的边
    for (int i = 0; i < sides; i++)
    {
        PlotLine((Point2D){x[i], y[i]}, (Point2D){x[(i + 1) % sides], y[(i + 1) % sides]}, buffer, mode);
    }
}
```

#### 第二节	说明

- 根据正多边形外接圆心点坐标和外接圆半径，平均分配角度以计算各边的点坐标，并绘制。

### 第十章	三角函数绘制

```c
/**
  *@ FunctionName: void PlotWave(Point2D p0, uint8_t length, uint8_t amplitude, uint8_t frequency, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 绘制波形
  *@ Time: Oct 31, 2024
  *@ Requirement：
  */
void PlotWave(Point2D p0, uint8_t length, uint8_t amplitude, uint8_t frequency, SSDBuffer buffer, FillMode mode)
{
	int8_t prevX = p0.x, prevY = p0.y;  // 起点
	uint8_t x, y;

    // 逐步绘制波浪线
    for (x = p0.x; x <= p0.x + length; x++)
    {
        // 使用正弦函数计算波形的 y 值
        y = p0.y + (int)(amplitude * sin(2 * M_PI * frequency * (x - p0.x) / length));
        // 绘制当前点和前一个点之间的线段
        PlotLine((Point2D){prevX,prevY}, (Point2D){x,y}, buffer, mode);
        // 更新前一个点的坐标
        prevX = x;
        prevY = y;
    }
}
```

### 第十一章	椭圆的绘制

```c
/**
  *@ FunctionName: void PlotEllipse(Point2D p0, Point2D p1, Point2D p2, Point2D p3, SSDBuffer buffer, FillMode mode, Bool isDashed)
  *@ Author: CzrTuringB
  *@ Brief: 绘制椭圆
  *@ Time: Oct 31, 2024
  *@ Requirement：
  */
void PlotEllipse(Point2D p0, Point2D p1, Point2D p2, Point2D p3, SSDBuffer buffer, FillMode mode, Bool isDashed)
{
    // 计算椭圆的中心点
    int xc = (p0.x + p2.x) / 2;  // 长轴中心
    int yc = (p1.y + p3.y) / 2;  // 短轴中心

    // 计算长轴和短轴的半径（直接计算两点间的距离）
    float a = sqrt((p2.x - p0.x) * (p2.x - p0.x) + (p2.y - p0.y) * (p2.y - p0.y)) / 2;  // 长轴半径
    float b = sqrt((p3.x - p1.x) * (p3.x - p1.x) + (p3.y - p1.y) * (p3.y - p1.y)) / 2;  // 短轴半径

    // 如果长轴小于短轴，交换它们
    if (a < b)
    {
        float temp = a;
        a = b;
        b = temp;
    }

    // 计算旋转角度，使用 atan2 来计算长轴方向的角度
    float angle = atan2(p2.y - p0.y, p2.x - p0.x);  // 使用 atan2 计算旋转角度

    // 预计算旋转矩阵的cos和sin值
    float cosTheta = cos(angle);
    float sinTheta = sin(angle);

    // 椭圆绘制的核心：Bresenham算法
    int x = 0, y = b;  // 起点
    int a2 = a * a, b2 = b * b;
    int two_a2 = 2 * a2, two_b2 = 2 * b2;
    int p;
    int dx = y * two_a2, dy = x * two_b2;

    // 用来控制虚线间隔的计数器
    int dashCount = 0;
    const int dashInterval = 4;  // 每隔两个像素绘制一个点

    // 第一阶段
    p = (b2 - a2 * b + 0.25 * a2);  // 初始决策参数
    while (dx > dy)
    {
        // 根据isDashed判断是否绘制虚线
        if (!isDashed || dashCount % dashInterval == 0) // 判断是否绘制点
        {
            int xRot = (int)(xc + x * cosTheta - y * sinTheta);
            int yRot = (int)(yc + x * sinTheta + y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc - x * cosTheta - y * sinTheta);
            yRot = (int)(yc - x * sinTheta + y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc + x * cosTheta + y * sinTheta);
            yRot = (int)(yc + x * sinTheta - y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc - x * cosTheta + y * sinTheta);
            yRot = (int)(yc - x * sinTheta - y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);
        }

        dashCount++;  // 增加计数器
        x++;
        dy += two_b2;
        if (p < 0) {
            p += b2 + dy;
        } else {
            y--;
            dx -= two_a2;
            p += b2 + dy - dx;
        }
    }

    // 第二阶段
    p = (b2 * (x + 0.5) * (x + 0.5) + a2 * (y - 1) * (y - 1) - a2 * b2);  // 更新决策参数
    while (y >= 0)
    {
        // 根据isDashed判断是否绘制虚线
        if (!isDashed || dashCount % dashInterval == 0) // 判断是否绘制点
        {
            int xRot = (int)(xc + x * cosTheta - y * sinTheta);
            int yRot = (int)(yc + x * sinTheta + y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc - x * cosTheta - y * sinTheta);
            yRot = (int)(yc - x * sinTheta + y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc + x * cosTheta + y * sinTheta);
            yRot = (int)(yc + x * sinTheta - y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);

            xRot = (int)(xc - x * cosTheta + y * sinTheta);
            yRot = (int)(yc - x * sinTheta - y * cosTheta);
            PlotPoint((Point2D){xRot, yRot}, buffer, mode);
        }

        dashCount++;  // 增加计数器
        y--;
        dx -= two_a2;
        if (p > 0)
        {
            p += a2 - dx;
        }
        else
        {
            x++;
            dy += two_b2;
            p += a2 - dx + dy;
        }
    }
}
```



### 第十二章	字符的绘制与取模

#### 第一节	字符取模

- 取模软件：[PCtoLCD](https://github.com/ChenZR0509/TuringUi/blob/main/Tools/PCtoLCD.zip)
- 字符制作格式：行列式、阴码、逆向高位在前，索引顺序为ASCII

#### 第二节	字模【索引ASCII】

- 6×8大小字符：

  ```c
  const unsigned char Font6x8[]=
  {
  		0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  		0x00, 0x00, 0x00, 0xF4, 0x00, 0x00,
  		0x00, 0x00, 0xE0, 0x00, 0xE0, 0x00,
  		0x00, 0x28, 0xFE, 0x28, 0xFE, 0x28,
  		0x00, 0x24, 0x54, 0xFE, 0x54, 0x48,
  		0x00, 0x46, 0x26, 0x10, 0xC8, 0xC4,
  		0x00, 0x6C, 0x92, 0xAA, 0x44, 0x0A,
  		0x00, 0x00, 0xA0, 0xC0, 0x00, 0x00,
  		0x00, 0x00, 0x38, 0x44, 0x82, 0x00,
  		0x00, 0x00, 0x82, 0x44, 0x38, 0x00,
  		0x00, 0x28, 0x10, 0x7C, 0x10, 0x28,
  		0x00, 0x10, 0x10, 0x7C, 0x10, 0x10,
  		0x00, 0x00, 0x00, 0x05, 0x06, 0x00,
  		0x00, 0x10, 0x10, 0x10, 0x10, 0x10,
  		0x00, 0x00, 0x06, 0x06, 0x00, 0x00,
  		0x00, 0x04, 0x08, 0x10, 0x20, 0x40,
  		0x00, 0x7C, 0x8A, 0x92, 0xA2, 0x7C,
  		0x00, 0x00, 0x42, 0xFE, 0x02, 0x00,
  		0x00, 0x42, 0x86, 0x8A, 0x92, 0x62,
  		0x00, 0x84, 0x82, 0xA2, 0xD2, 0x8C,
  		0x00, 0x18, 0x28, 0x48, 0xFE, 0x08,
  		0x00, 0xE4, 0xA2, 0xA2, 0xA2, 0x9C,
  		0x00, 0x3C, 0x52, 0x92, 0x92, 0x0C,
  		0x00, 0x80, 0x8E, 0x90, 0xA0, 0xC0,
  		0x00, 0x6C, 0x92, 0x92, 0x92, 0x6C,
  		0x00, 0x60, 0x92, 0x92, 0x94, 0x78,
  		0x00, 0x00, 0x6C, 0x6C, 0x00, 0x00,
  		0x00, 0x00, 0x6A, 0x6C, 0x00, 0x00,
  		0x00, 0x10, 0x28, 0x44, 0x82, 0x00,
  		0x00, 0x28, 0x28, 0x28, 0x28, 0x28,
  		0x00, 0x00, 0x82, 0x44, 0x28, 0x10,
  		0x00, 0x40, 0x80, 0x8A, 0x90, 0x60,
  		0x00, 0x4C, 0x92, 0x9A, 0x8A, 0x7C,
  		0x00, 0x3E, 0x48, 0x88, 0x48, 0x3E,
  		0x00, 0xFE, 0x92, 0x92, 0x92, 0x6C,
  		0x00, 0x7C, 0x82, 0x82, 0x82, 0x44,
  		0x00, 0xFE, 0x82, 0x82, 0x44, 0x38,
  		0x00, 0xFE, 0x92, 0x92, 0x92, 0x82,
  		0x00, 0xFE, 0x90, 0x90, 0x90, 0x80,
  		0x00, 0x7C, 0x82, 0x92, 0x92, 0x5E,
  		0x00, 0xFE, 0x10, 0x10, 0x10, 0xFE,
  		0x00, 0x00, 0x82, 0xFE, 0x82, 0x00,
  		0x00, 0x04, 0x02, 0x82, 0xFC, 0x80,
  		0x00, 0xFE, 0x10, 0x28, 0x44, 0x82,
  		0x00, 0xFE, 0x02, 0x02, 0x02, 0x02,
  		0x00, 0xFE, 0x40, 0x30, 0x40, 0xFE,
  		0x00, 0xFE, 0x20, 0x10, 0x08, 0xFE,
  		0x00, 0x7C, 0x82, 0x82, 0x82, 0x7C,
  		0x00, 0xFE, 0x90, 0x90, 0x90, 0x60,
  		0x00, 0x7C, 0x82, 0x8A, 0x84, 0x7A,
  		0x00, 0xFE, 0x90, 0x98, 0x94, 0x62,
  		0x00, 0x62, 0x92, 0x92, 0x92, 0x8C,
  		0x00, 0x80, 0x80, 0xFE, 0x80, 0x80,
  		0x00, 0xFC, 0x02, 0x02, 0x02, 0xFC,
  		0x00, 0xF8, 0x04, 0x02, 0x04, 0xF8,
  		0x00, 0xFC, 0x02, 0x1C, 0x02, 0xFC,
  		0x00, 0xC6, 0x28, 0x10, 0x28, 0xC6,
  		0x00, 0xE0, 0x10, 0x0E, 0x10, 0xE0,
  		0x00, 0x86, 0x8A, 0x92, 0xA2, 0xC2,
  		0x00, 0x00, 0xFE, 0x82, 0x82, 0x00,
  		0x00, 0xAA, 0x54, 0xAA, 0x54, 0xAA,
  		0x00, 0x00, 0x82, 0x82, 0xFE, 0x00,
  		0x00, 0x20, 0x40, 0x80, 0x40, 0x20,
  		0x00, 0x02, 0x02, 0x02, 0x02, 0x02,
  		0x00, 0x00, 0x80, 0x40, 0x20, 0x00,
  		0x00, 0x04, 0x2A, 0x2A, 0x2A, 0x1E,
  		0x00, 0xFE, 0x12, 0x22, 0x22, 0x1C,
  		0x00, 0x1C, 0x22, 0x22, 0x22, 0x04,
  		0x00, 0x1C, 0x22, 0x22, 0x12, 0xFE,
  		0x00, 0x1C, 0x2A, 0x2A, 0x2A, 0x18,
  		0x00, 0x10, 0x7E, 0x90, 0x80, 0x40,
  		0x00, 0x18, 0x25, 0x25, 0x25, 0x3E,
  		0x00, 0xFE, 0x10, 0x20, 0x20, 0x1E,
  		0x00, 0x00, 0x22, 0xBE, 0x02, 0x00,
  		0x00, 0x02, 0x01, 0x21, 0xBE, 0x00,
  		0x00, 0xFE, 0x08, 0x14, 0x22, 0x00,
  		0x00, 0x00, 0x82, 0xFE, 0x02, 0x00,
  		0x00, 0x3E, 0x20, 0x18, 0x20, 0x1E,
  		0x00, 0x3E, 0x10, 0x20, 0x20, 0x1E,
  		0x00, 0x1C, 0x22, 0x22, 0x22, 0x1C,
  		0x00, 0x3F, 0x24, 0x24, 0x24, 0x18,
  		0x00, 0x18, 0x24, 0x24, 0x18, 0x3F,
  		0x00, 0x3E, 0x10, 0x20, 0x20, 0x10,
  		0x00, 0x12, 0x2A, 0x2A, 0x2A, 0x04,
  		0x00, 0x20, 0xFC, 0x22, 0x02, 0x04,
  		0x00, 0x3C, 0x02, 0x02, 0x04, 0x3E,
  		0x00, 0x38, 0x04, 0x02, 0x04, 0x38,
  		0x00, 0x3C, 0x02, 0x0C, 0x02, 0x3C,
  		0x00, 0x22, 0x14, 0x08, 0x14, 0x22,
  		0x00, 0x38, 0x05, 0x05, 0x05, 0x3E,
  		0x00, 0x22, 0x26, 0x2A, 0x32, 0x22,
  		0x00, 0x00, 0x08, 0x7F, 0x41, 0x00,
  		0x00, 0x00, 0x00, 0xFF, 0x00, 0x00,
  		0x00, 0x41, 0x7F, 0x08, 0x00, 0x00,
  		0x40, 0x80, 0x80, 0x40, 0x40, 0x80,
  };
  ```

- 8×16大小字符：

  ```c
  const unsigned char Font8x16[]=
  {
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,//
    0x00,0x00,0x00,0x1F,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0xCC,0x00,0x00,0x00,0x00,// !
    0x00,0x08,0x30,0x40,0x08,0x30,0x40,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,// "
    0x00,0x02,0x03,0x1E,0x02,0x03,0x1E,0x00,0x00,0x20,0xFC,0x20,0x20,0xFC,0x20,0x00,// #
    0x00,0x0E,0x11,0x11,0x3F,0x10,0x0C,0x00,0x00,0x18,0x04,0x04,0xFF,0x84,0x78,0x00,// $
    0x0F,0x10,0x0F,0x01,0x06,0x18,0x00,0x00,0x00,0x8C,0x30,0xC0,0x78,0x84,0x78,0x00,// %
    0x00,0x0F,0x10,0x11,0x0E,0x00,0x00,0x00,0x78,0x84,0xC4,0x34,0x98,0xE4,0x84,0x08,// &
    0x00,0x48,0x70,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,// '
    0x00,0x00,0x00,0x07,0x18,0x20,0x40,0x00,0x00,0x00,0x00,0xE0,0x18,0x04,0x02,0x00,// (
    0x00,0x40,0x20,0x18,0x07,0x00,0x00,0x00,0x00,0x02,0x04,0x18,0xE0,0x00,0x00,0x00,// )
    0x02,0x02,0x01,0x0F,0x01,0x02,0x02,0x00,0x40,0x40,0x80,0xF0,0x80,0x40,0x40,0x00,// *
    0x00,0x00,0x00,0x00,0x07,0x00,0x00,0x00,0x00,0x80,0x80,0x80,0xF0,0x80,0x80,0x80,// +
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x09,0x0E,0x00,0x00,0x00,0x00,0x00,// ,
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x80,0x80,0x80,0x80,0x80,0x80,0x00,// -
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x0C,0x0C,0x00,0x00,0x00,0x00,0x00,// .
    0x00,0x00,0x00,0x00,0x03,0x1C,0x20,0x00,0x00,0x06,0x18,0xE0,0x00,0x00,0x00,0x00,// /
    0x00,0x07,0x08,0x10,0x10,0x08,0x07,0x00,0x00,0xF0,0x08,0x04,0x04,0x08,0xF0,0x00,// 0
    0x00,0x00,0x08,0x08,0x1F,0x00,0x00,0x00,0x00,0x00,0x04,0x04,0xFC,0x04,0x04,0x00,// 1
    0x00,0x0E,0x10,0x10,0x10,0x10,0x0F,0x00,0x00,0x0C,0x14,0x24,0x44,0x84,0x0C,0x00,// 2
    0x00,0x0C,0x10,0x10,0x10,0x11,0x0E,0x00,0x00,0x18,0x04,0x84,0x84,0x44,0x38,0x00,// 3
    0x00,0x00,0x01,0x02,0x0C,0x1F,0x00,0x00,0x00,0x60,0xA0,0x24,0x24,0xFC,0x24,0x24,// 4
    0x00,0x1F,0x11,0x11,0x11,0x10,0x10,0x00,0x00,0x98,0x04,0x04,0x04,0x88,0x70,0x00,// 5
    0x00,0x07,0x08,0x11,0x11,0x09,0x00,0x00,0x00,0xF0,0x88,0x04,0x04,0x04,0xF8,0x00,// 6
    0x00,0x18,0x10,0x10,0x11,0x16,0x18,0x00,0x00,0x00,0x00,0x7C,0x80,0x00,0x00,0x00,// 7
    0x00,0x0E,0x11,0x10,0x10,0x11,0x0E,0x00,0x00,0x38,0x44,0x84,0x84,0x44,0x38,0x00,// 8
    0x00,0x0F,0x10,0x10,0x10,0x08,0x07,0x00,0x00,0x80,0x48,0x44,0x44,0x88,0xF0,0x00,// 9
    0x00,0x00,0x00,0x03,0x03,0x00,0x00,0x00,0x00,0x00,0x00,0x0C,0x0C,0x00,0x00,0x00,// :
    0x00,0x00,0x00,0x01,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x07,0x00,0x00,0x00,0x00,// ;
    0x00,0x00,0x01,0x02,0x04,0x08,0x10,0x00,0x00,0x80,0x40,0x20,0x10,0x08,0x04,0x00,// <
    0x00,0x02,0x02,0x02,0x02,0x02,0x02,0x00,0x00,0x40,0x40,0x40,0x40,0x40,0x40,0x00,// =
    0x00,0x10,0x08,0x04,0x02,0x01,0x00,0x00,0x00,0x04,0x08,0x10,0x20,0x40,0x80,0x00,// >
    0x00,0x0E,0x12,0x10,0x10,0x11,0x0E,0x00,0x00,0x00,0x00,0x0C,0xEC,0x00,0x00,0x00,// ?
    0x03,0x0C,0x13,0x14,0x17,0x08,0x07,0x00,0xE0,0x18,0xE4,0x14,0xF4,0x14,0xE8,0x00,// @
    0x00,0x00,0x03,0x1C,0x07,0x00,0x00,0x00,0x04,0x3C,0xC4,0x40,0x40,0xE4,0x1C,0x04,// A
    0x10,0x1F,0x11,0x11,0x11,0x0E,0x00,0x00,0x04,0xFC,0x04,0x04,0x04,0x88,0x70,0x00,// B
    0x03,0x0C,0x10,0x10,0x10,0x10,0x1C,0x00,0xE0,0x18,0x04,0x04,0x04,0x08,0x10,0x00,// C
    0x10,0x1F,0x10,0x10,0x10,0x08,0x07,0x00,0x04,0xFC,0x04,0x04,0x04,0x08,0xF0,0x00,// D
    0x10,0x1F,0x11,0x11,0x17,0x10,0x08,0x00,0x04,0xFC,0x04,0x04,0xC4,0x04,0x18,0x00,// E
    0x10,0x1F,0x11,0x11,0x17,0x10,0x08,0x00,0x04,0xFC,0x04,0x00,0xC0,0x00,0x00,0x00,// F
    0x03,0x0C,0x10,0x10,0x10,0x1C,0x00,0x00,0xE0,0x18,0x04,0x04,0x44,0x78,0x40,0x00,// G
    0x10,0x1F,0x10,0x00,0x00,0x10,0x1F,0x10,0x04,0xFC,0x84,0x80,0x80,0x84,0xFC,0x04,// H
    0x00,0x10,0x10,0x1F,0x10,0x10,0x00,0x00,0x00,0x04,0x04,0xFC,0x04,0x04,0x00,0x00,// I
    0x00,0x00,0x10,0x10,0x1F,0x10,0x10,0x00,0x03,0x01,0x01,0x01,0xFE,0x00,0x00,0x00,// J
    0x10,0x1F,0x11,0x03,0x14,0x18,0x10,0x00,0x04,0xFC,0x04,0x80,0x64,0x1C,0x04,0x00,// K
    0x10,0x1F,0x10,0x00,0x00,0x00,0x00,0x00,0x04,0xFC,0x04,0x04,0x04,0x04,0x0C,0x00,// L
    0x10,0x1F,0x1F,0x00,0x1F,0x1F,0x10,0x00,0x04,0xFC,0x80,0x7C,0x80,0xFC,0x04,0x00,// M
    0x10,0x1F,0x0C,0x03,0x00,0x10,0x1F,0x10,0x04,0xFC,0x04,0x00,0xE0,0x18,0xFC,0x00,// N
    0x07,0x08,0x10,0x10,0x10,0x08,0x07,0x00,0xF0,0x08,0x04,0x04,0x04,0x08,0xF0,0x00,// O
    0x10,0x1F,0x10,0x10,0x10,0x10,0x0F,0x00,0x04,0xFC,0x84,0x80,0x80,0x80,0x00,0x00,// O
    0x07,0x08,0x10,0x10,0x10,0x08,0x07,0x00,0xF0,0x08,0x14,0x14,0x0C,0x0A,0xF2,0x00,// Q
    0x10,0x1F,0x11,0x11,0x11,0x11,0x0E,0x00,0x04,0xFC,0x04,0x00,0xC0,0x30,0x0C,0x04,// R
    0x00,0x0E,0x11,0x10,0x10,0x10,0x1C,0x00,0x00,0x1C,0x04,0x84,0x84,0x44,0x38,0x00,// S
    0x18,0x10,0x10,0x1F,0x10,0x10,0x18,0x00,0x00,0x00,0x04,0xFC,0x04,0x00,0x00,0x00,// T
    0x10,0x1F,0x10,0x00,0x00,0x10,0x1F,0x10,0x00,0xF8,0x04,0x04,0x04,0x04,0xF8,0x00,// U
    0x10,0x1E,0x11,0x00,0x00,0x13,0x1C,0x10,0x00,0x00,0xE0,0x1C,0x70,0x80,0x00,0x00,// V
    0x10,0x1F,0x00,0x1F,0x00,0x1F,0x10,0x00,0x00,0xC0,0x7C,0x80,0x7C,0xC0,0x00,0x00,// W
    0x10,0x18,0x16,0x01,0x01,0x16,0x18,0x10,0x04,0x0C,0x34,0xC0,0xC0,0x34,0x0C,0x04,// X
    0x10,0x1C,0x13,0x00,0x13,0x1C,0x10,0x00,0x00,0x00,0x04,0xFC,0x04,0x00,0x00,0x00,// Y
    0x08,0x10,0x10,0x10,0x13,0x1C,0x10,0x00,0x04,0x1C,0x64,0x84,0x04,0x04,0x18,0x00,// Z
    0x00,0x00,0x00,0x7F,0x40,0x40,0x40,0x00,0x00,0x00,0x00,0xFE,0x02,0x02,0x02,0x00,// [
    0x00,0x20,0x1C,0x03,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x80,0x60,0x1C,0x03,0x00,// '\'
    0x00,0x40,0x40,0x40,0x7F,0x00,0x00,0x00,0x00,0x02,0x02,0x02,0xFE,0x00,0x00,0x00,// ]
    0x00,0x00,0x20,0x40,0x40,0x20,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,// ^
    0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x01,0x01,0x01,0x01,0x01,0x01,0x01,0x01,// _
    0x00,0x40,0x40,0x20,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,// `
    0x00,0x00,0x01,0x01,0x01,0x00,0x00,0x00,0x00,0x98,0x24,0x24,0x48,0xFC,0x04,0x00,// a
    0x08,0x0F,0x00,0x01,0x01,0x00,0x00,0x00,0x00,0xFC,0x88,0x04,0x04,0x88,0x70,0x00,// b
    0x00,0x00,0x00,0x01,0x01,0x01,0x00,0x00,0x00,0x70,0x88,0x04,0x04,0x04,0x88,0x00,// c
    0x00,0x00,0x01,0x01,0x01,0x09,0x0F,0x00,0x00,0xF8,0x04,0x04,0x04,0x08,0xFC,0x04,// d
    0x00,0x00,0x01,0x01,0x01,0x01,0x00,0x00,0x00,0xF8,0x24,0x24,0x24,0x24,0xE8,0x00,// e
    0x00,0x01,0x01,0x07,0x09,0x09,0x04,0x00,0x00,0x04,0x04,0xFC,0x04,0x04,0x00,0x00,// f
    0x00,0x00,0x01,0x01,0x01,0x01,0x01,0x00,0x00,0xD6,0x29,0x29,0x29,0xC9,0x06,0x00,// g
    0x08,0x0F,0x00,0x01,0x01,0x01,0x00,0x00,0x04,0xFC,0x84,0x00,0x00,0x04,0xFC,0x04,// h
    0x00,0x01,0x19,0x19,0x00,0x00,0x00,0x00,0x00,0x04,0x04,0xFC,0x04,0x04,0x00,0x00,// i
    0x00,0x00,0x00,0x01,0x19,0x19,0x00,0x00,0x00,0x03,0x01,0x01,0x01,0xFE,0x00,0x00,// j
    0x08,0x0F,0x00,0x00,0x01,0x01,0x01,0x00,0x04,0xFC,0x24,0x60,0x94,0x0C,0x04,0x00,// k
    0x00,0x08,0x08,0x1F,0x00,0x00,0x00,0x00,0x00,0x04,0x04,0xFC,0x04,0x04,0x00,0x00,// l
    0x01,0x01,0x01,0x01,0x01,0x01,0x01,0x00,0x04,0xFC,0x04,0x00,0xFC,0x04,0x00,0xFC,// m
    0x01,0x01,0x00,0x01,0x01,0x01,0x00,0x00,0x04,0xFC,0x84,0x00,0x00,0x04,0xFC,0x04,// n
    0x00,0x00,0x01,0x01,0x01,0x01,0x00,0x00,0x00,0xF8,0x04,0x04,0x04,0x04,0xF8,0x00,// o
    0x01,0x01,0x00,0x01,0x01,0x00,0x00,0x00,0x01,0xFF,0x89,0x04,0x04,0x88,0x70,0x00,// p
    0x00,0x00,0x00,0x01,0x01,0x00,0x01,0x00,0x00,0x70,0x88,0x04,0x04,0x89,0xFF,0x01,// q
    0x01,0x01,0x01,0x00,0x01,0x01,0x01,0x00,0x04,0x04,0xFC,0x84,0x04,0x00,0x80,0x00,// r
    0x00,0x00,0x01,0x01,0x01,0x01,0x01,0x00,0x00,0xCC,0x24,0x24,0x24,0x24,0x98,0x00,// s
    0x00,0x01,0x01,0x07,0x01,0x01,0x00,0x00,0x00,0x00,0x00,0xF8,0x04,0x04,0x08,0x00,// t
    0x01,0x01,0x00,0x00,0x00,0x01,0x01,0x00,0x00,0xF8,0x04,0x04,0x04,0x08,0xFC,0x04,// u
    0x01,0x01,0x01,0x00,0x01,0x01,0x01,0x00,0x00,0xC0,0x30,0x0C,0x30,0xC0,0x00,0x00,// v
    0x01,0x01,0x00,0x01,0x01,0x00,0x01,0x01,0x80,0x70,0x0C,0x30,0xE0,0x1C,0x60,0x80,// w
    0x00,0x01,0x01,0x01,0x00,0x01,0x01,0x00,0x00,0x04,0x8C,0x70,0x74,0x8C,0x04,0x00,// x
    0x01,0x01,0x01,0x00,0x00,0x01,0x01,0x01,0x00,0x81,0x61,0x1E,0x18,0x60,0x80,0x00,// y
    0x00,0x01,0x01,0x01,0x01,0x01,0x01,0x00,0x00,0x84,0x0C,0x34,0x44,0x84,0x0C,0x00,// z
    0x00,0x00,0x00,0x00,0x00,0x3F,0x40,0x40,0x00,0x00,0x00,0x00,0x80,0x7C,0x02,0x02,// {
    0x00,0x00,0x00,0x00,0xFF,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0xFF,0x00,0x00,0x00,// |
    0x40,0x40,0x3F,0x00,0x00,0x00,0x00,0x00,0x02,0x02,0x7C,0x80,0x00,0x00,0x00,0x00,// }
    0x00,0x40,0x80,0x40,0x40,0x20,0x40,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,// ~
  };
  ```

#### 第三节	字符输出

- 字符的绘制：

  ```c
  /**
    *@ FunctionName: void PlotChar(Point2D p0, uint8_t chr, FontSize charSize, SSDBuffer buffer)
    *@ Author: CzrTuringB
    *@ Brief: 绘制字符
    *@ Time: Oct 31, 2024
    *@ Requirement：
    *@ 	1、p0为左下角点坐标
    */
  void PlotChar(Point2D p0, uint8_t chr, FontSize charSize, SSDBuffer buffer)
  {
  	uint8_t fontIndex = chr - ' ';
  	uint8_t yShift = p0.y%8;
  	uint8_t yIndex = p0.y/8;
  	if(charSize == C8x16)
  	{
  		//绘制大字体
  		for(uint8_t i=0; i<16; i++)
  		{
  			if(uiDevice->dataValid == False)
  			{
  				if (i < 8)
  				{
  					if(p0.x < 0) 	continue;
  					if(yIndex+1 < 0) continue;
  					if(yIndex+2 < 0) continue;
  					buffer[yIndex+2][p0.x] |= (Font8x16[fontIndex * 16 + i] >> (7 - yShift));
  					buffer[yIndex+1][p0.x] |= (Font8x16[fontIndex * 16 + i] << yShift);
  				}
  				else
  				{
  					if(p0.x-8 < 0) 	continue;
  					if(yIndex < 0) continue;
  					if(yIndex+1 < 0) continue;
  					buffer[yIndex+1][p0.x-8] |= (Font8x16[fontIndex * 16 + i] >> (7 - yShift));
  					buffer[yIndex][p0.x-8] |= (Font8x16[fontIndex * 16 + i] << yShift);
  				}
  			}
  			else
  			{
  				if (i < 8)
  				{
  					if(p0.x < 0) 	continue;
  					if(yIndex+1 < 0) continue;
  					if(yIndex+2 < 0) continue;
  					buffer[yIndex+2][p0.x] = ~((Font8x16[fontIndex * 16 + i] >> (7 - yShift))
  							| (~buffer[yIndex+2][p0.x]));
  					buffer[yIndex+1][p0.x] = ~((Font8x16[fontIndex * 16 + i] << yShift)
  							| (~buffer[yIndex+1][p0.x]));
  				}
  				else
  				{
  					if(p0.x-8 < 0) 	continue;
  					if(yIndex < 0) continue;
  					if(yIndex+1 < 0) continue;
  					buffer[yIndex+1][p0.x-8] = ~((Font8x16[fontIndex * 16 + i]
  							>> (7 - yShift)) | (~buffer[yIndex+1][p0.x-8]));
  					buffer[yIndex][p0.x-8] = ~((Font8x16[fontIndex * 16 + i] << yShift)
  							| (~buffer[yIndex][p0.x-8]));
  				}
  			}
  			p0.x++;
  		}
  	}
  	else
  	{
  		for(uint8_t i=0; i<6; i++)
  		{
  			if(uiDevice->dataValid == False)
  			{
  				if(p0.x < 0) 	continue;
  				if(yIndex < 0) continue;
  				if(yIndex+1 < 0) continue;
  				buffer[yIndex+1][p0.x] |= (Font6x8[fontIndex * 6 + i] >> (7 - yShift));
  				buffer[yIndex][p0.x] |= (Font6x8[fontIndex * 6 + i] << yShift);
  			}
  			else
  			{
  				if(p0.x < 0) 	continue;
  				if(yIndex < 0) continue;
  				if(yIndex+1 < 0) continue;
  				buffer[yIndex+1][p0.x] = ~((Font6x8[fontIndex * 6 + i] >> (7 - yShift))
  						| (~buffer[yIndex+1][p0.x]));
  				buffer[yIndex][p0.x] = ~((Font6x8[fontIndex * 6 + i] << yShift)
  						| (~buffer[yIndex][p0.x]));
  			}
  			p0.x++;
  		}
  	}
  }
  ```

- 字符串的绘制：

  ```c
  /**
    *@ FunctionName: void PlotString(Point2D p0, const char* str, FontSize charSize, SSDBuffer buffer)
    *@ Author: CzrTuringB
    *@ Brief: 绘制字符串
    *@ Time: Oct 31, 2024
    *@ Requirement：
    */
  void PlotString(Point2D p0, const char* str, FontSize charSize, SSDBuffer buffer)
  {
  	size_t strSize = strlen(str);
  	for(uint8_t i=0;i<strSize;i++)
  	{
  		PlotChar(p0,str[i],charSize, buffer);
  		if(charSize == C8x16)
  		{
  			p0.x+=8;
  		}
  		else
  		{
  			p0.x+=6;
  		}
  	}
  }
  ```

### 第十三章	图标的绘制与制作

#### 第一节	图标制作

- 取模工具：[Image2Lcd 2.9](https://github.com/ChenZR0509/TuringUi/blob/main/Tools/Image2Lcd 2.9.zip)
- 图标下载网址：[Svg Vector Icons & PNG / PSD / EPS / PNM / Free Downloads - OnlineWebFonts.COM](https://www.onlinewebfonts.com/icon)【不要下载太复杂的图标】
- 首先确定待显示图标大小，这里以32×32为例，图标大小设计最好取为8的整数倍。
- 在上面的网址注册账号并下载图标，图表选为PNG格式，使用PhotoShop软件打开Png文件，设置图像大小为32×32，并导出为jpg格式。
- 打开Image2Lcd软件，导入jpg格式文件，输出数据类型选为BMP图片格式，保存即可。
- 打开PCtoLCD软件，导入bmp格式文件，使用鼠标左右键修正像素点使其美观好看，记得将修改完成后的图片另存一下。
- PCtoLCD设置：取模方式为列行式，取模走向为顺向(高位在前)，格式为C51格式，数据前缀为0x，数据后缀为“, ”。
- 图标选取与取模经验：
  1. 尽量避免选取存在小弧度的图标。
  2. 尽量避免选取过于复杂的图标。
  3. 尽量避免选取线条过于细的图标。
  4. 取模时候会伴随着分辨率的丢失，因此需要使用PCtoLCD工具对个别像素进行补全和去除，其应遵循对称原则、圆滑原则。

#### 第二节	代码

```c
/**
  *@ FunctionName: void PlotBMP(Point2D p0,uint8_t width,uint8_t height,const uint8_t* picture, SSDBuffer buffer)
  *@ Author: CzrTuringB
  *@ Brief: 绘制BMP图像
  *@ Time: Nov 1, 2024
  *@ Requirement：
  *@ 1、(x,y)即为图像左下角点坐标
  */
void PlotBMP(Point2D p0,uint8_t width,uint8_t height,const uint8_t* picture, SSDBuffer buffer)
{
	uint8_t pageStart = p0.y/8;
	uint8_t pageEnd   = (p0.y+height)/8;
	uint8_t shift = p0.y%8;

	for(uint8_t i=p0.x; i<p0.x+width; i++)
	{
		for(uint8_t j=pageStart; j<pageEnd;j++)
		{
			if(uiDevice->dataValid == False)
			{
				buffer[j+1][i] |= (picture[i-p0.x+(pageEnd-j-1)*width] >> (7 - shift));
				buffer[j][i] |= (picture[i-p0.x+(pageEnd-j-1)*width] << shift);
			}
			else
			{
				buffer[j+1][i] = ~((picture[i-p0.x+(pageEnd-j-1)*width] >> (7 - shift))
						| (~buffer[j+1][i]));
				buffer[j][i] = ~((picture[i-p0.x+(pageEnd-j-1)*width] << shift)
						| (~buffer[j][i]));
			}
		}
	}
}
```

#### 第三节	自己制作的图标

- 地址：https://github.com/ChenZR0509/TuringUi/tree/main/Resources/Icons

![Clock01](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/Clock01.BMP)

![](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/Mail01.BMP)

![setting01](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/setting01.BMP)

![Video01](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/Video01.BMP)

![Wave00](./TuringUi%E7%B3%BB%E5%88%97(%E4%BA%8C)/Wave00.bmp)

### 第十四章	数学曲线之美[^6]

#### 第一节	贝塞尔曲线的绘制

- 贝塞尔曲线绘制：

  ```c
  //贝塞尔曲线相关参数
  typedef struct
  {
  	Point2D controlPoints[4];
  	uint8_t segment;
  }BezierLine;
  /**
    *@ FunctionName: void BezierLineInit(BezierLine* line, Point2D* controlPoints, uint8_t segment)
    *@ Author: CzrTuringB
    *@ Brief: 初始化贝塞尔曲线结构体
    *@ Time: Dec 18, 2024
    *@ Requirement：
    *@ 	1、打开AutoCad软件，绘制一个从(0,0)开始到(128,64)的矩形
    *@ 	2、使用控制点策略绘制曲线形状
    *@ 	3、右键点击曲线特性，设置其控制点个数为4，并微调控制点，读取控制点坐标参数用于初始化贝塞尔结构体
    *@ 	4、段数越多，则曲线越圆滑，但计算量也就越大
    */
  void BezierLineInit(BezierLine *line, Point2D *controlPoints, uint8_t segment)
  {
  	line->segment = segment;
  	for (int i = 0; i < 4; i++)
  	{
  		line->controlPoints[i] = controlPoints[i];
  	}
  }
  /**
    *@ FunctionName: void PlotBezierLine(BezierLine* line)
    *@ Author: CzrTuringB
    *@ Brief: 绘制贝塞尔曲线
    *@ Time: Dec 18, 2024
    *@ Requirement：
    */
  void PlotBezierLine(BezierLine *line)
  {
  	Point2D prevPoint = line->controlPoints[0];
  	Point2D currentPoint;
  	//绘制贝塞尔曲线
  	for (int i = 1; i <= line->segment; i++)
  	{
  		float t = (float) i / line->segment;
  		// 贝塞尔曲线公式
  		float u = 1 - t;
  		currentPoint.x = (int) (u * u * u * line->controlPoints[0].x
  				+ 3 * u * u * t * line->controlPoints[1].x
  				+ 3 * u * t * t * line->controlPoints[2].x
  				+ t * t * t * line->controlPoints[3].x);
  		currentPoint.y = (int) (u * u * u * line->controlPoints[0].y
  				+ 3 * u * u * t * line->controlPoints[1].y
  				+ 3 * u * t * t * line->controlPoints[2].y
  				+ t * t * t * line->controlPoints[3].y);
  		// 绘制曲线点
  		PlotLine(prevPoint.x, prevPoint.y, currentPoint.x, currentPoint.y);
  		prevPoint = currentPoint;
  	}
  }
  ```

- 带箭头的贝塞尔曲线：

  ```c
  /**
    *@ FunctionName: void PlotBezierArrowLine(BezierLine* line)
    *@ Author: CzrTuringB
    *@ Brief: 绘制尾部带箭头的贝塞尔曲线
    *@ Time: Dec 18, 2024
    *@ Requirement：
    */
  void PlotBezierArrowLine(BezierLine *line)
  {
  	Point2D prevPoint = line->controlPoints[0];
  	Point2D currentPoint = line->controlPoints[0];
  	//绘制贝塞尔曲线
  	for (int i = 1; i <= line->segment; i++)
  	{
  		float t = (float) i / line->segment;
  		// 贝塞尔曲线公式
  		float u = 1 - t;
  		currentPoint.x = (int) (u * u * u * line->controlPoints[0].x
  				+ 3 * u * u * t * line->controlPoints[1].x
  				+ 3 * u * t * t * line->controlPoints[2].x
  				+ t * t * t * line->controlPoints[3].x);
  		currentPoint.y = (int) (u * u * u * line->controlPoints[0].y
  				+ 3 * u * u * t * line->controlPoints[1].y
  				+ 3 * u * t * t * line->controlPoints[2].y
  				+ t * t * t * line->controlPoints[3].y);
  		// 绘制曲线点
  		PlotLine(prevPoint.x, prevPoint.y, currentPoint.x, currentPoint.y);
  		if (i < line->segment)
  		{
  			prevPoint = currentPoint;
  		}
  	}
  	//计算箭头方向（切线方向）
  	float angle = atan2(currentPoint.y - prevPoint.y,
  			currentPoint.x - prevPoint.x);
  
  	//计算箭头的两个端点
  	uint8_t x = currentPoint.x + (int8_t) (6 * cos(angle + M_PI - 0.5));
  	uint8_t y = currentPoint.y + (int8_t) (6 * sin(angle + M_PI - 0.5));
  	PlotLine(currentPoint.x, currentPoint.y, x, y);
  	x = currentPoint.x + (int8_t) (6 * cos(angle + M_PI + 0.5));
  	y = currentPoint.y + (int8_t) (6 * sin(angle + M_PI + 0.5));
  	PlotLine(currentPoint.x, currentPoint.y, x, y);
  }
  ```

- 参数步骤：
  1. 打开AutoCad软件，绘制一个从(0,0)开始到(128,64)的矩形
  2. 使用控制点策略绘制曲线形状
  3. 右键点击曲线特性，设置其控制点个数为4，并微调控制点，读取控制点坐标参数用于初始化贝塞尔结构体
  4. 段数越多，则曲线越圆滑，但计算量也就越大

#### 第二节	玫瑰曲线的绘制

```c
/**
  *@ FunctionName: void PlotRoseCurve(Point2D pc, float a, float k, uint16_t steps, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 玫瑰曲线的绘制
  *@ Time: Dec 18, 2024
  *@ Requirement：
  *@ 	1、xc，yc即为玫瑰曲线的中心坐标
  *@ 	2、a为曲线大小
  *@ 	3、k为玫瑰花瓣数
  *@ 	4、steps为曲线平滑程度
  */
void PlotRoseCurve(Point2D pc, float a, float k, uint16_t steps, SSDBuffer buffer, FillMode mode)
{
    float theta, r;
    Point2D p;
    float stepSize = 2 * M_PI / steps; // 每一步的角度增量

    for (uint16_t i = 0; i <= steps; i++)
    {
        theta = i * stepSize;       // 当前角度
        r = a * cos(k * theta);     // 计算极径，可以改为 sin(k * theta) 看效果
        // 将极坐标转换为笛卡尔坐标
        p.x = (uint8_t)(pc.x + r * cos(theta));
        p.y = (uint8_t)(pc.y + r * sin(theta));
        // 绘制点
        PlotPoint(p, buffer, mode);
    }
}
```

#### 第三节	心型曲线的绘制

```c
/**
  *@ FunctionName: void PlotHeartCurve(Point2D pc, uint16_t steps, float scale, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 心形曲线的绘制
  *@ Time: Dec 18, 2024
  *@ Requirement：
  */
void PlotHeartCurve(Point2D pc, uint16_t steps, float scale, SSDBuffer buffer, FillMode mode)
{
    float t, x, y;
    Point2D p;
    float stepSize = 2 * M_PI / steps; // 每步角度增量

    for (uint16_t i = 0; i <= steps; i++)
    {
        t = i * stepSize;  // 当前角度
        // 计算心形曲线的极坐标（x, y）
        x = 16 * pow(sin(t), 3);
        y = 13 * cos(t) - 5 * cos(2 * t) - 2 * cos(3 * t) - cos(4 * t);
        // 应用缩放因子
        x *= scale;
        y *= scale;
        // 将心形曲线绘制到显示屏坐标系中
        p.x = (uint8_t)(pc.x + x);  // x坐标
        p.y = (uint8_t)(pc.y - y);  // y坐标需要反转以适应屏幕坐标系
        // 绘制点
        PlotPoint(p, buffer, mode);
    }
}
```

#### 第四节	蝴蝶曲线的绘制

```c
/**
  *@ FunctionName: void PlotButterflyCurve(Point2D pc, uint16_t steps, float scale, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 蝴蝶曲线的绘制
  *@ Time: Dec 18, 2024
  *@ Requirement：
  */
void PlotButterflyCurve(Point2D pc, uint16_t steps, float scale, SSDBuffer buffer, FillMode mode)
{
    float t, x, y;
    Point2D p;
    float stepSize = 2 * M_PI / steps; // 每步角度增量

    for (uint16_t i = 0; i <= steps; i++)
    {
        t = i * stepSize;  // 当前角度
        // 计算蝴蝶曲线的极坐标（x, y）
        x = sin(t) * (exp(cos(t)) - 2 * cos(4 * t) - pow(sin(t), 2));
        y = cos(t) * (exp(cos(t)) - 2 * cos(4 * t) - pow(sin(t), 2));
        // 应用缩放因子
        x *= scale;  // x坐标放大/缩小
        y *= scale;  // y坐标放大/缩小
        // 将坐标映射到屏幕坐标范围
        p.x = (uint8_t)(pc.x + x);  // x坐标
        p.y = (uint8_t)(pc.y - y);  // y坐标需要反转，以适应屏幕坐标系
        // 绘制点
        PlotPoint(p, buffer, mode);
    }
}
```

#### 第五节	星型曲线的绘制

```c
/**
  *@ FunctionName: void PlotStarCurve(Point2D pc, uint16_t steps, float a, float b, float k, float scale, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 星型曲线的绘制
  *@ Time: Dec 19, 2024
  *@ Requirement：
  *@ 	1、xc，yc即为星型曲线的中心坐标
  *@ 	2、a、b决定曲线大小和形状
  *@ 	3、k为瓣数
  *@ 	4、steps为曲线平滑程度
  */
void PlotStarCurve(Point2D pc, uint16_t steps, float a, float b, float k, float scale, SSDBuffer buffer, FillMode mode)
{
    float t, x, y;         // 参数 t 和计算得到的 x, y 坐标
    Point2D p;
    float stepSize = 2 * M_PI / steps;  // 每步角度增量

    for (uint16_t i = 0; i <= steps; i++)
    {
        t = i * stepSize;  // 当前角度

        // 计算星形线的极坐标（x, y）
        x = cos(t) * (a + b * cos(k * t));
        y = sin(t) * (a + b * cos(k * t));

        // 应用缩放因子
        x *= scale;  // x坐标放大/缩小
        y *= scale;  // y坐标放大/缩小

        // 将坐标映射到屏幕坐标范围（128x64）
        p.x = (uint8_t)(pc.x + x);  // x坐标
        p.y = (uint8_t)(pc.y - y);  // y坐标需要反转，以适应屏幕坐标系
        // 绘制点
        PlotPoint(p, buffer, mode);
    }
}
```

#### 第十五章	二维坐标系绘制

```c
/**
  *@ FunctionName: void Plot2Axes(Point2D pc, uint8_t xLength, uint8_t yLength, float angle, SSDBuffer buffer, FillMode mode)
  *@ Author: CzrTuringB
  *@ Brief: 绘制二维坐标系
  *@ Time: Dec 18, 2024
  *@ Requirement：
  */
void Plot2Axes(Point2D pc, uint8_t xLength, uint8_t yLength, float angle, SSDBuffer buffer, FillMode mode)
{
	RotateValue rot = {0,0,angle,pc.x,pc.y,0};
	uint8_t y  = pc.y + yLength;
	uint8_t x  = pc.x + xLength;
	Point2D px = (Point2D){x,pc.y};
	Point2D py = (Point2D){pc.x,y};
	RotatePoint2D(&px, rot);
	RotatePoint2D(&py, rot);
	PlotLine(pc, px, buffer, mode);
	PlotArrow(pc, px, buffer, EndArrow, mode);
	PlotLine(pc, py, buffer, mode);
	PlotArrow(pc, py, buffer, EndArrow, mode);
}
```

### 第十六章	多边形闭合区域种子填充算法

```c
/**
  *@ FunctionName: void PlotFillPolygon(Point2D pc, SSDBuffer buffer, FillMode mode, uint8_t Boundary)
  *@ Author: CzrTuringB
  *@ Brief: 种子算法填充闭合图形
  *@ Time: Dec 20, 2024
  *@ Requirement：
  *@ 	1、不要填充大面积图形(最多225个像素)
  */
void PlotFillPolygon(Point2D pc, SSDBuffer buffer, FillMode mode, uint8_t Boundary)
{
    //创建一个8*128的位图记录已填充状态，每位表示一个像素是否已填充
    uint8_t filledPoints[PageNumber][ScreenWidth] = {0};

    // 定义位操作宏
    #define IS_FILLED(p) (filledPoints[(p.y) / 8][(p.x)] & (1 << ((p.y) % 8)))
    #define MARK_FILLED(p) (filledPoints[(p.y) / 8][(p.x)] |= (1 << ((p.y) % 8)))
    #define IS_VALID(p) ((p.x) < 128 && (p.y) < 64)

    // 定义递归函数
    void Fill(Point2D p, SSDBuffer buffer)
    {
        // 边界检查，防止越界
        if (!IS_VALID(p)) return;

        // 如果点已经填充或是边界，跳过
        if (IS_FILLED(p) || GetPoint(p, buffer) == Boundary) return;

        // 填充当前点
        PlotPoint(p, buffer, mode);

        // 标记当前点为已填充
        MARK_FILLED(p);

        // 递归填充上下左右相邻点
        Fill((Point2D){p.x + 1, p.y}, buffer);
        Fill((Point2D){p.x - 1, p.y}, buffer);
        Fill((Point2D){p.x, p.y + 1}, buffer);
        Fill((Point2D){p.x, p.y - 1}, buffer);
    }
    // 从种子点开始填充
    Fill(pc, buffer);
}
```

### 第十七章	填充与蒙版

#### 第一节	填充实现

- 块的填充：

  ```c
  /**
    *@ FunctionName: void PlotFillBlock(uint8_t bx, uint8_t by, SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 填充块
    *@ Time: Dec 25, 2024
    *@ Requirement：
    */
  void PlotFillBlock(uint8_t bx, uint8_t by, SSDBuffer buffer, FillMode mode)
  {
  	for(uint8_t j=(8*bx);j<8;j++)
  	{
  		switch(mode)
  		{
  			case Fill0:
  				buffer[by][j] = 0;
  				break;
  			case Fill1:
  				buffer[by][j] = 0xFF;
  				break;
  			case Fill5:
  				if(j%2 == 0)	buffer[by][j] = 0xAA;
  				else			buffer[by][j] = 0x55;
  				break;
  			case FillA:
  				if(j%2 == 0)	buffer[by][j] = 0x55;
  				else			buffer[by][j] = 0xAA;
  				break;
  			case FillE:
  				if(j%2 != 0)	buffer[by][j] = 0xFF;
  				else			buffer[by][j] = 0xAA;
  				break;
  			case FillB:
  				if(j%2 != 0)	buffer[by][j] = 0xFF;
  				else			buffer[by][j] = 0x55;
  				break;
  			case FillD:
  				if(j%2 == 0)	buffer[by][j] = 0xFF;
  				else			buffer[by][j] = 0xAA;
  				break;
  			case Fill7:
  				if(j%2 == 0)	buffer[by][j] = 0xFF;
  				else			buffer[by][j] = 0xAA;
  				break;
  			case FillEBD7:
  				if(j%4 == 0)	buffer[by][j] = 0xEE;
  				if(j%4 == 1)	buffer[by][j] = 0xBB;
  				if(j%4 == 2)	buffer[by][j] = 0xDD;
  				if(j%4 == 3)	buffer[by][j] = 0x77;
  				break;
  		}
  	}
  }
  ```

- 屏幕填充：

  ```c
  /**
    *@ FunctionName: void PlotFillScreen(SSDBuffer buffer, FillMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 填充整个缓冲区
    *@ Time: Dec 25, 2024
    *@ Requirement：
    *@ 	1、填充完成，建议手动调用ssdwriteall函数刷屏
    */
  void PlotFillScreen(SSDBuffer buffer, FillMode mode)
  {
  	for(uint8_t i=0;i<PageNumber;i++)
  	{
  		for(uint8_t j=0;j<ScreenWidth;j++)
  		{
  			switch(mode)
  			{
  				case Fill0:
  					buffer[i][j] = 0x00;
  					break;
  				case Fill1:
  					buffer[i][j] = 0xFF;
  					break;
  				case Fill5:
  					if(j%2 == 0)	buffer[i][j] = 0xAA;
  					else			buffer[i][j] = 0x55;
  					break;
  				case FillA:
  					if(j%2 == 0)	buffer[i][j] = 0x55;
  					else			buffer[i][j] = 0xAA;
  					break;
  				case FillE:
  					if(j%2 != 0)	buffer[i][j] = 0xFF;
  					else			buffer[i][j] = 0xAA;
  					break;
  				case FillB:
  					if(j%2 != 0)	buffer[i][j] = 0xFF;
  					else			buffer[i][j] = 0x55;
  					break;
  				case FillD:
  					if(j%2 == 0)	buffer[i][j] = 0xFF;
  					else			buffer[i][j] = 0xAA;
  					break;
  				case Fill7:
  					if(j%2 == 0)	buffer[i][j] = 0xFF;
  					else			buffer[i][j] = 0xAA;
  					break;
  				case FillEBD7:
  					if(j%4 == 0)	buffer[i][j] = 0xEE;
  					if(j%4 == 1)	buffer[i][j] = 0xBB;
  					if(j%4 == 2)	buffer[i][j] = 0xDD;
  					if(j%4 == 3)	buffer[i][j] = 0x77;
  					break;
  			}
  		}
  	}
  }
  ```

- 清除块内容：

  ```c
  /**
    *@ FunctionName: void PlotCleanBlock(uint8_t bx, uint8_t by, SSDBuffer buffer)
    *@ Author: CzrTuringB
    *@ Brief: 清空块
    *@ Time: Jan 2, 2025
    *@ Requirement：
    */
  void PlotCleanBlock(uint8_t bx, uint8_t by, SSDBuffer buffer)
  {
  	if(uiDevice->dataValid == True)
  	{
  		PlotFillBlock(bx, by, buffer, Fill1);
  	}
  	else
  	{
  		PlotFillBlock(bx, by, buffer, Fill0);
  	}
  }
  ```

- 清除缓冲区：

  ```c
  /**
    *@ FunctionName: void PlotCleanBuffer(SSDBuffer buffer)
    *@ Author: CzrTuringB
    *@ Brief: 清空整个缓冲区
    *@ Time: Jan 2, 2025
    *@ Requirement：
    */
  void PlotCleanBuffer(SSDBuffer buffer)
  {
  	if(uiDevice->dataValid == True)
  	{
  		PlotFillScreen(buffer, Fill1);
  	}
  	else
  	{
  		PlotFillScreen(buffer, Fill0);
  	}
  }
  ```

- 清除屏幕：

  ```c
  /**
    *@ FunctionName: void PlotCleanScreen()
    *@ Author: CzrTuringB
    *@ Brief: 清除屏幕
    *@ Time: Jan 2, 2025
    *@ Requirement：
    */
  void PlotCleanScreen()
  {
  	SSDClean();
  	SSDUpdateScreen();
  }
  ```

#### 第二节	蒙版实现[^7]

- **蒙版（Mask）**是一种图像处理和计算机图形学中的技术，用于对图像的某些部分进行遮挡或选择性处理。蒙版通常是一个与原图像尺寸相同的二值图像或灰度图像，其中的特定区域会被标记为“透明”或“可编辑”，而其他部分则保持不变或受到限制。在OLED中开辟出一块缓冲区，在蒙版缓冲区中绘制填充相关元素并与显示缓冲区进行逻辑运算以实现蒙版功能。

- 整屏蒙版操作：

  ```c
  /**
    *@ FunctionName: void PlotScreenMaskCover(CoverMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 整屏蒙版操作
    *@ Time: Jan 10, 2025
    *@ Requirement：
    */
  void PlotScreenMaskCover(CoverMode mode)
  {
  	for(uint8_t i=0;i<PageNumber;i++)
  	{
  		for(uint8_t j=0;j<ScreenWidth;j++)
  		{
  			switch(mode)
  			{
  				case AndCover:
  					uiDevice->disBuffer[i][j] &= uiDevice->maskBuffer[i][j];
  					break;
  				case OrCover:
  					uiDevice->disBuffer[i][j] |= uiDevice->maskBuffer[i][j];
  					break;
  				case NotCover:
  					uiDevice->disBuffer[i][j] = (~uiDevice->disBuffer[i][j]);
  					break;
  				case XorCover:
  					uiDevice->disBuffer[i][j] ^= uiDevice->maskBuffer[i][j];
  					break;
  				case XnorCover:
  					uiDevice->disBuffer[i][j] ^= uiDevice->maskBuffer[i][j];
  					uiDevice->disBuffer[i][j] = (~uiDevice->disBuffer[i][j]);
  					break;
  			}
  		}
  	}
  }
  ```

- 区域蒙版操作：

  ```c
  /**
    *@ FunctionName: void PlotScreenMaskCover(CoverMode mode)
    *@ Author: CzrTuringB
    *@ Brief: 区域蒙版操作
    *@ Time: Jan 10, 2025
    *@ Requirement：
    *@ 	1、根据矩形斜对角线坐标，进行蒙版覆盖
    */
  void PlotAreaMaskCover(Point2D sp, Point2D ep, CoverMode mode)
  {
  	//检测边界
  	if(EdgeDetect(sp) == True || EdgeDetect(ep)) return;
  	if(ep.x < sp.x || ep.y < sp.y) return;
  	//计算起始页和结束页索引
  	uint8_t sPage = sp.y / 8;
  	uint8_t sShift = sp.y % 8;
  	uint8_t ePage = ep.y / 8;
  	uint8_t eShift = ep.y % 8;
  	//处理边界页
  	for(uint8_t j = sp.x; j < ep.x; j++)
  	{
  		//处理边界页
  		uint8_t sTemp,eTemp;
  		sTemp = ~(0xFF << sShift);			//计算起始页保留位
  		sTemp &= uiDevice->disBuffer[sPage][j]; //保留边界外数据
  		eTemp = ~(0xFF >> (7-eShift));			//计算结束页保留位
  		eTemp &= uiDevice->disBuffer[ePage][j]; //保留边界外数据
  		switch(mode)
  		{
  			case AndCover:
  				uiDevice->disBuffer[sPage][j] &= uiDevice->maskBuffer[sPage][j];
  				uiDevice->disBuffer[ePage][j] &= uiDevice->maskBuffer[ePage][j];
  				uiDevice->disBuffer[sPage][j] &= (0xFF << sShift);
  				uiDevice->disBuffer[sPage][j] |= sTemp;
  				uiDevice->disBuffer[ePage][j] &= (0xFF >> (7-eShift));
  				uiDevice->disBuffer[ePage][j] |= eTemp;
  				break;
  			case OrCover:
  				uiDevice->disBuffer[sPage][j] |= uiDevice->maskBuffer[sPage][j];
  				uiDevice->disBuffer[ePage][j] |= uiDevice->maskBuffer[ePage][j];
  				uiDevice->disBuffer[sPage][j] &= (0xFF << sShift);
  				uiDevice->disBuffer[sPage][j] |= sTemp;
  				uiDevice->disBuffer[ePage][j] &= (0xFF >> (7-eShift));
  				uiDevice->disBuffer[ePage][j] |= eTemp;
  				break;
  			case NotCover:
  				uiDevice->disBuffer[sPage][j] = (~uiDevice->disBuffer[sPage][j]);
  				uiDevice->disBuffer[ePage][j] = (~uiDevice->disBuffer[ePage][j]);
  				uiDevice->disBuffer[sPage][j] &= (0xFF << sShift);
  				uiDevice->disBuffer[sPage][j] |= sTemp;
  				uiDevice->disBuffer[ePage][j] &= (0xFF >> (7-eShift));
  				uiDevice->disBuffer[ePage][j] |= eTemp;
  				break;
  			case XorCover:
  				uiDevice->disBuffer[sPage][j] ^= uiDevice->maskBuffer[sPage][j];
  				uiDevice->disBuffer[ePage][j] ^= uiDevice->maskBuffer[ePage][j];
  				uiDevice->disBuffer[sPage][j] &= (0xFF << sShift);
  				uiDevice->disBuffer[sPage][j] |= sTemp;
  				uiDevice->disBuffer[ePage][j] &= (0xFF >> (7-eShift));
  				uiDevice->disBuffer[ePage][j] |= eTemp;
  				break;
  			case XnorCover:
  				uiDevice->disBuffer[sPage][j] ^= uiDevice->maskBuffer[sPage][j];
  				uiDevice->disBuffer[sPage][j] = (~uiDevice->disBuffer[sPage][j]);
  				uiDevice->disBuffer[ePage][j] ^= uiDevice->maskBuffer[ePage][j];
  				uiDevice->disBuffer[ePage][j] = (~uiDevice->disBuffer[ePage][j]);
  				uiDevice->disBuffer[sPage][j] &= (0xFF << sShift);
  				uiDevice->disBuffer[sPage][j] |= sTemp;
  				uiDevice->disBuffer[ePage][j] &= (0xFF >> (7-eShift));
  				uiDevice->disBuffer[ePage][j] |= eTemp;
  				break;
  		}
  	}
  	//处理完整页
  	for(uint8_t i = sPage + 1; i < ePage; i++)
  	{
  		for(uint8_t j = sp.x; j < ep.x; j++)
  		{
  			switch(mode)
  			{
  				case AndCover:
  					uiDevice->disBuffer[i][j] &= uiDevice->maskBuffer[i][j];
  					break;
  				case OrCover:
  					uiDevice->disBuffer[i][j] |= uiDevice->maskBuffer[i][j];
  					break;
  				case NotCover:
  					uiDevice->disBuffer[i][j] = (~uiDevice->disBuffer[i][j]);
  					break;
  				case XorCover:
  					uiDevice->disBuffer[i][j] ^= uiDevice->maskBuffer[i][j];
  					break;
  				case XnorCover:
  					uiDevice->disBuffer[i][j] ^= uiDevice->maskBuffer[i][j];
  					uiDevice->disBuffer[i][j] = (~uiDevice->disBuffer[i][j]);
  					break;
  			}
  		}
  	}
  }
  ```

- 蒙版运算类型：

  ```c
  typedef enum
  {
  	AndCover,	//按位与运算
  	OrCover,	//按位或运算
  	NotCover,	//按位取反运算
  	XorCover,	//按位异或运算
  	XnorCover,	//按位同或运算
  }CoverMode;
  ```

- 蒙版运算理解：

  - 与运算：显示缓冲区和蒙版缓冲区都为1时，则才能显示，于是可以实现**指定区域的分割、裁剪、遮挡**。
  - 或运算：显示缓冲区或蒙版缓冲区存在一个为1时，则显示，其可以实现**多个图层的合并**。
  - 非运算：实现**显示反转效果**。
  - 同或运算：通过同或运算检测两个元素是否相同，**常用于状态比对或一致性检测**。
  - 异或运算：通过异或运算在UI元素之间进行切换，**常用于实现闪烁、渐变等动画效果**。

---

## 参考

[^1]:[OLED显示屏5阶灰度显示原理介绍-抖动算法_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1vg411n7LD/?spm_id_from=333.337.search-card.all.click)
[^2]:[Bresenham直线算法 - 搜索](https://www.bing.com/search?q=Bresenham直线算法&gs_lcrp=EgRlZGdlKgYIABBFGDkyBggAEEUYOagCALACAA&FORM=ANCMS9&ucpdpc=UCPD&adppc=EDGEDBB&PC=HCTS&mkt=zh-CN)
[^3]:[计算机图形学06：中点Bresenham画圆（并填充边界，例如：边界用红色，内部用绿色填充）_中点画圆算法代码-CSDN博客](https://blog.csdn.net/myf_666/article/details/128166947)
[^4]:[中点bresenham算法画圆和椭圆]( https://www.bilibili.com/video/BV1VY41137qS/?share_source=copy_web&vd_source=cba2f4d639356f9c0e8787dfdd5ef91f )

[^5]：[扫描线种子填充算法演示_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1RA411J7wG/?spm_id_from=333.337.search-card.all.click)

[^6]: [数学的有趣图形-玫瑰线 - 知乎](https://zhuanlan.zhihu.com/p/298415455)
[^7]: [【剪辑入门】用一卷卫生纸帮你搞懂“蒙版”_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1fL41177s8/?spm_id_from=333.337.search-card.all.click&vd_source=c706c24a5bec5f2b0d006d2505e30519)
