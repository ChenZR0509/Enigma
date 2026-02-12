---
title: TuringUi系列(三)
categories:
  - 项目
  - TuringUi
tags:
  - 计算机图形学
abbrlink: 10692
date: 2024-12-24 09:41:00

---

# TuringUi系列(三)

---

>**CzrTuringB：**在前一个博客我们实现了相关二维图像的绘制，在本博客我实现了三维空间的运动变换，并引入了半边数据结构实现了对圆柱、圆锥、六面体的绘制及其消隐算法，**其理论上可以拓展至其他多面体**。之后介绍了基于贝塞尔缓动动画的实现方法，并设计了几款动画在博客尾部进行展示。

---

## 第一部分	运动变换

### 第一章	坐标系定义

- 二维坐标系：
  - 原点为屏幕左下角第一个像素
  - x轴横向
  - y轴纵向
- 三维坐标系：
  - x轴与二维坐标系x轴平行
  - y轴与二维坐标系y轴平行
  - z轴垂直屏幕向外
  - 原点任意
  
- 三维点数据结构：

  ```c
  //三维坐标点
  typedef struct
  {
      float x, y, z;
  }Point3D;
  ```

### 第二章	正交投影

- 直接忽略深度信息，将三维坐标转换为二维坐标。
  $$
  \left\{ \begin{array}{l}
  	x'=x_c+x\cdot scale\\
  	y'=y_c+y\cdot scale\\
  \end{array} \right.
  $$

- 代码：

  ```c
  /**
    *@ FunctionName: void OrtProject(Point3D* p3, Point2D* p2, float scale, Point2D pc)
    *@ Author: CzrTuringB
    *@ Brief: 正交投影变换
    *@ Time: Dec 18, 2024
    *@ Requirement：
    *@ 	1、xc和yc是正交投影的中心
    */
  void OrtProject(Point3D p3, Point2D* p2, float scale, Point2D pc)
  {
  	p2->x = (uint8_t)(pc.x + p3.x * scale);
  	p2->y = (uint8_t)(pc.y - p3.y * scale);
  }
  ```

### 第三章	透视投影

- 可以实现进大远小的投影
  $$
  \left\{ \begin{array}{l}
  	x'=x_c+\frac{x\cdot scale}{z+depth}\\
  	y'=y_c+\frac{y\cdot scale}{z+depth}\\
  \end{array} \right.
  $$

- 代码：

  ```c
  /**
    *@ FunctionName: void PreProject(Point3D* p3, Point2D* p2, float scale, uint8_t depth, Point2D pc)
    *@ Author: CzrTuringB
    *@ Brief: 透视投影变换
    *@ Time: Dec 18, 2024
    *@ Requirement：
    *@ 	1、xc和yc是透视投影的中心
    */
  void PreProject(Point3D p3, Point2D* p2, float scale, uint8_t depth, Point2D pc)
  {
  	if(p3.z + depth != 0)
  	{
  		p2->x = (uint8_t)(pc.x + p3.x * scale / (p3.z + depth));
  		p2->y = (uint8_t)(pc.y - p3.y * scale / (p3.z + depth));
  	}
  	else
  	{
  		p2->x = (uint8_t)(pc.x);
  		p2->y = (uint8_t)(pc.y);
  	}
  }
  ```

### 第四章	平移

- 二维平移公式：
  $$
  2D\text{：}\left\{ \begin{array}{l}
  	x=x_0+x_d\\
  	y=y_0+y_d\\
  \end{array} \right.
  $$

- 三维平移公式：
  $$
  3D\text{：}\left\{ \begin{array}{l}
  	x=x_0+x_d\\
  	y=y_0+y_d\\
  	z=z_0+z_d\\
  \end{array} \right.
  $$

- 代码：

  ```c
  typedef struct
  {
      int16_t xTra;
      int16_t yTra;
      int16_t zTra;
  }TranslateValue;
  /**
    *@ FunctionName: void TranslatePoint2D(Point2D* p2, TranslateValue tra)
    *@ Author: CzrTuringB
    *@ Brief: 二维点的平移
    *@ Time: Dec 18, 2024
    *@ Requirement：
    *@ 	1、xTra和yTra表示平移距离
    */
  void TranslatePoint2D(Point2D* p2, TranslateValue tra)
  {
  	p2->x += tra.xTra;
  	p2->y += tra.yTra;
  }
  /**
    *@ FunctionName: void TranslatePoint3D(Point3D* p3, TranslateValue tra);
    *@ Author: CzrTuringB
    *@ Brief: 三维点的平移
    *@ Time: Dec 18, 2024
    *@ Requirement：
    */
  void TranslatePoint3D(Point3D* p3, TranslateValue tra)
  {
  	p3->x += tra.xTra;
  	p3->y += tra.yTra;
  	p3->z += tra.zTra;
  }
  ```

### 第五章	旋转

- 二维旋转：旋转轴是固定的为z轴，只需要确定旋转中心即可
  $$
  \left[ \begin{array}{c}
  	x'\\
  	y'\\
  \end{array} \right] =\left[ \begin{matrix}
  	\cos \theta&		-\sin \theta\\
  	\sin \theta&		\cos \theta\\
  \end{matrix} \right] \left[ \begin{array}{c}
  	x\\
  	y\\
  \end{array} \right]
  $$

- 三维旋转：旋转轴即为x轴、y轴、z轴，只需要确定绕三个轴旋转的角度，即可实现任意旋转。
  $$
  x\text{轴旋转：}\left[ \begin{array}{c}
  	x'\\
  	y'\\
  	z'\\
  \end{array} \right] =\left[ \begin{matrix}
  	1&		0&		0\\
  	0&		\cos \theta&		-\sin \theta\\
  	0&		\sin \theta&		\cos \theta\\
  \end{matrix} \right] \left[ \begin{array}{c}
  	x\\
  	y\\
  	z\\
  \end{array} \right] 
  $$
  $$
  y\text{轴旋转：}\left[ \begin{array}{c}
  	x'\\
  	y'\\
  	z'\\
  \end{array} \right] =\left[ \begin{matrix}
  	\cos \theta&		0&		\sin \theta\\
  	0&		1&		0\\
  	-\sin \theta&		0&		\cos \theta\\
  \end{matrix} \right] \left[ \begin{array}{c}
  	x\\
  	y\\
  	z\\
  \end{array} \right] 
  $$
  $$
  z\text{轴旋转：}\left[ \begin{array}{c}
  	x'\\
  	y'\\
  	z'\\
  \end{array} \right] =\left[ \begin{matrix}
  	\cos \theta&		-\sin \theta&		0\\
  	\sin \theta&		\cos \theta&		0\\
  	0&		0&		1\\
  \end{matrix} \right] \left[ \begin{array}{c}
  	x\\
  	y\\
  	z\\
  \end{array} \right]
  $$

- 代码：

  ```c
  typedef struct
  {
      float xAngle;
      float yAngle;
      float zAngle;
      int8_t xc;
      int8_t yc;
      int8_t zc;
  }RotateValue;
  void RotatePoint2D(Point2D* p2, RotateValue rot)
  {
  	//计算三角函数
      float rad = rot.zAngle * M_PI / 180.0f;
      float cosA = cos(rad);
      float sinA = sin(rad);
  
      //将点平移到旋转中心
      uint8_t tx = p2->x - rot.xc;
      uint8_t ty = p2->y - rot.yc;
  
      //旋转计算
      float rx = tx * cosA - ty * sinA;
      float ry = tx * sinA + ty * cosA;
  
      //平移回原位置
      p2->x = rot.xc + (int8_t)rx;
      p2->y = rot.yc + (int8_t)ry;
  }
  void RotatePoint3D(Point3D* p3,RotateValue rot)
  {
      // 将角度转换为弧度
      float xRad = rot.xAngle * 3.14159265 / 180.0f;
      float yRad = rot.yAngle * 3.14159265 / 180.0f;
      float zRad = rot.zAngle * 3.14159265 / 180.0f;
      // 平移到旋转中心
      float px = p3->x - rot.xc;
      float py = p3->y - rot.yc;
      float pz = p3->z - rot.zc;
      // 旋转绕 X 轴
      float cosX = cos(xRad);
      float sinX = sin(xRad);
      float tempY = py * cosX - pz * sinX;
      float tempZ = py * sinX + pz * cosX;
      py = tempY;
      pz = tempZ;
      // 旋转绕 Y 轴
      float cosY = cos(yRad);
      float sinY = sin(yRad);
      float tempX = px * cosY + pz * sinY;
      tempZ = -px * sinY + pz * cosY;
      px = tempX;
      pz = tempZ;
      // 旋转绕 Z 轴
      float cosZ = cos(zRad);
      float sinZ = sin(zRad);
      tempX = px * cosZ - py * sinZ;
      tempY = px * sinZ + py * cosZ;
      px = tempX;
      py = tempY;
      // 平移回原位置
      p3->x = px + rot.xc;
      p3->y = py + rot.yc;
      p3->z = pz + rot.zc;
  }
  ```

## 第二部分	三维元素绘制

### 第一章	半边数据结构[^1]

- **半边数据结构**：通常用于计算机图形学中的多边形建模。它的核心思想是通过对边界（包括边和顶点）进行有效的组织，使得可以高效地执行几何处理，如布尔运算、邻接查询等。

- **基本元素：**

  - **顶点（Vertex）**： 顶点表示多边形的角或边的端点。
  - **边（Edge）**： 边是连接两个顶点的线段。
  - **半边(Half Edge)**：半边是带有方向的边，一条半边和另一条顶点相同方向相反的半边组成一条边。半边的方向依据右手定则确定。
  - **面（Face）**： 面是由边界边界围成的区域，可以包含多个边。每个面由首尾相连的多个半边组成，其法向量指向三维多面体的外部，与半边方向遵循右手定则。

- **数据结构设计：**

  - 顶点：每一个顶点都包含这一个与此顶点相关的半边指针，方便后期查询

    ```c
    struct Vertex
    {
        Point3D p;       		 // 顶点坐标
        Point3D p3;			     // 顶点运动后坐标
        Point2D p2;				 // 顶点投影后坐标
        HalfEdge* halfEdge;  	 // 存储任意一个与顶点关联的半边
    };
    ```

  - 半边：

    ```c
    struct HalfEdge
    {
        Vertex* vStart;    // 半边开始的顶点
        Vertex* vEnd;      // 半边指向的顶点
        HalfEdge* pair;    // 相对的半边
        HalfEdge* next;    // 同一面上的下一个半边
        HalfEdge* prev;    // 同一面上的上一个半边
        Face* face;        // 半边所属的面
    };
    ```

  - 面：每一个面都存储这一个在这个平面上的半边和这个面的法向量，方便后期查询

    ```c
    struct Face
    {
        Point3D dirVector;		  // 平面的法向量
        HalfEdge* halfEdge;       // 面的某一个半边
    };
    ```

- **三维向量计算：**

  - 向量减法：依据两个三维顶点或两个三维向量即可计算出其和向量

    ```c
    void Subtract(Point3D* a, Point3D* b, Point3D* result)
    {
        result->x = a->x - b->x;
        result->y = a->y - b->y;
        result->z = a->z - b->z;
    }
    ```

  - 向量点积：点积的结果是一个标量，其为向量模的积在乘两个向量夹角的COS值

    ```c
    void DotProduct(Point3D *a, Point3D *b,float* result)
    {
    	*result=a->x * b->x + a->y * b->y + a->z * b->z;
    }
    ```

  - 向量叉乘：叉乘的结果是一个向量，此向量垂直于原来两个向量所构成的平面，其方向符合右手定则

    ```c
    void CrossProduct(Point3D *a, Point3D *b, Point3D *result)
    {
    	result->x = a->y * b->z - a->z * b->y;
    	result->y = a->z * b->x - a->x * b->z;
    	result->z = a->x * b->y - a->y * b->x;
    }
    ```

- **背向面消隐算法：**

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%89)/007.png)

  ​	依据空间三维坐标系和屏幕二维坐标系的关系，我们可以在Z轴上，多面体外部取一点将其作为视点，以视点和面上的一点构成一个视角向量，通过判断视角向量个平面法向量点积的正负即可确定平面是否朝向视点，依据绘制逻辑可以对背向视角的平面进行消隐操作。

  ​	如上图所示面向视角的平面法向量F2与视角向量V其点积结果小于零，背向视角的平面法向量F1与视角向量V其点积结果大于零。

  ```c
  Bool IsFrontFace(Face* face, Point3D viewPoint)
  {
      //获取面上的两个半边
      HalfEdge* e1 = face->halfEdge;
      HalfEdge* e2 = e1->next;
      //计算半边向量
      Point3D AB, AC;
      Subtract(&e1->vEnd->p3, &e1->vStart->p3, &AB);  // 向量 AB
      Subtract(&e2->vEnd->p3, &e2->vStart->p3, &AC);  // 向量 AC
      //计算面法向量（叉积）
      Point3D N;
      CrossProduct(&AB, &AC, &N);
      //计算视点到面上一个点的向量
      Point3D AV;
      Subtract(&viewPoint, &e1->vStart->p, &AV);
      //计算法向量与视点向量的点积
      float dotProduct;
  	DotProduct(&N, &AV, &dotProduct);
      //如果点积小于0，则视点在面前
      return (dotProduct < 0) ? True : False;
  }
  ```

### 第二章	任意多面体数据结构设计

- 存储结构：

  - 多面体顶点：**圆柱、圆锥顶点数为8**
  - 多面体半边：一半为多面体边数的二倍，**圆柱、圆锥的半边数分别为8和5**
  - 多面体面：**圆柱面为2，圆锥面为1**

  ```c
  typedef struct
  {
  	Vertex vertices[8];		//顶点数组
  	HalfEdge edges[8];		//半边数组
  	Face	faces[2];		//面数组
  	float maxDistence;		//圆柱空间最长距离
  	MoveValue move;			//运动变换参数
  }多面体;
  ```

- 半边数据结果初始化流程：

  1. 首先要在纸上或三维建模软件上绘制出三维物体，然后确定顶点的编号【顶点编号即对应顶点数组的索引】，其编号顺序不做特别要求。

  2. 初始化运动变换相关参数。

  3. 创建半边，半边的创建需要提供两个参数即半边起点和半边顶点，创建半边时其余参数设置为空。

     ```c
     void CreateHalfEdge(HalfEdge *edge, Vertex *vStart, Vertex *vEnd)
     {
     	edge->vStart = vStart;
     	edge->vEnd = vEnd;
     	edge->pair = NULL;
     	edge->next = NULL;
     	edge->prev = NULL;
     	edge->face = NULL;
     }
     ```

  4. 创建顶点：依据顶点所在位置输入顶点的坐标，在创建过程中查询与顶点相关的半边，将其中一个半边指针存入顶点结构体中。

     ```c
     void CreateVertex(Vertex* ver, Point3D p, HalfEdge *edgeArray, uint8_t numEdges, float scale, Point2D pc)
     {
     	//1、设置顶点的三维坐标
     	ver->p = p;
     	ver->p3 = p;
     	//2、计算顶点的二维坐标
     	OrtProject(p, &ver->p2, scale, pc);
     	//3、遍历半边数组，查找与顶点相关的半边
     	for (uint8_t i = 0; i < numEdges; i++)
     	{
     		HalfEdge *edge = &edgeArray[i];
     		// 检查半边的起始顶点或结束顶点是否是当前顶点
     		if (edge->vStart == ver || edge->vEnd == ver)
     		{
     			// 一旦找到一个与顶点相关的半边，设置顶点的halfEdge
     			ver->halfEdge = edge;
     			break;  // 找到一个相关半边后就可以停止遍历
     		}
     	}
     }
     ```

  5. 创建面：面的创建需要输入面的法向量，和面中的任意一个半边。对于简单的三维几何体，平面法向量可以直接口算输入【保证指向三维体外部】，复杂的多面体法向量可以通过建模软件获取，也可以根据平面上两半边向量的叉乘计算得出。**在创建面时，使用递归算法将属于这个平面的半边向量的面属性设置为这个平面。**

     >**注意：**em...实现是想不到什么好的初始化方法了，首先查找与平面已知半边相连接的半边，然后依据平面法向量、半边向量的点积是否为零，来判断这个半边是否位与这个平面平行的平面上。这里就存在以下几个问题，所以还是不能存储过于复杂的多面体：
     >
     >​	1、如果前面初始化存在错误，这里遍历就会挂掉。
     >
     >​	2、递归深度太深直接寄。
     >
     >​	3、浮点型进行了比较，倒是可以引入精度因子来解决。

     ```c
     void SetFaceRecursive(HalfEdge *edge, Face *face, Point3D *planeNormal,
     		HalfEdge *edgeArray, uint8_t numEdges)
     {
     	// 设置当前半边的面属性
     	if (edge->face != NULL)
     		return;
     	edge->face = face;
     	// 遍历所有半边，查找与当前半边相邻且符合条件的半边
     	for (uint8_t i = 0; i < numEdges; i++)
     	{
     		HalfEdge *e1 = &edgeArray[i];
     		// 如果半边的起点与当前半边的终点一致，并且在平面上，且未设置面属性
     		if (e1->vStart == edge->vEnd && e1->vEnd != edge->vStart && IsEdgeOnPlane(e1, planeNormal) == True)
     		{
     			//递归设置相邻半边的面属性
     			SetFaceRecursive(e1, face, planeNormal, edgeArray, numEdges);
     		}
     	}
     }
     void CreateFace(Face *face, HalfEdge *edge, Point3D vec, HalfEdge *edgeArray,
     		uint8_t numEdges)
     {
     	// 设置面和第一个半边的面属性
     	face->halfEdge = edge;
     	face->dirVector = vec;
     	// 使用递归设置其他相关半边的面属性
     	SetFaceRecursive(edge, face, &vec, edgeArray, numEdges);
     }
     ```

  6. 设置半边结构的其他属性：

     1. 遍历查询所有半边，依据起点和终点对应来设置pair属性。
     2. 遍历查询所有同平面的半边，设置半边的next和prev属性。

     ```c
     void SetHalfEdgePairs(HalfEdge *edgeArray, uint8_t numEdges)
     {
         for (uint8_t i = 0; i < numEdges; i++)
         {
             HalfEdge *e1 = &edgeArray[i];
             if (e1->pair != NULL) continue; // 已设置配对关系，跳过
     
             for (uint8_t j = i + 1; j < numEdges; j++) // 避免重复判断
             {
                 HalfEdge *e2 = &edgeArray[j];
                 if (e2->pair == NULL &&
                     e1->vStart == e2->vEnd &&
                     e1->vEnd == e2->vStart)
                 {
                     e1->pair = e2;
                     e2->pair = e1;
                     break; // 配对完成，跳出内循环
                 }
             }
         }
     }
     void SetHalfEdgeNextPrev(HalfEdge *edgeArray, uint8_t numEdges)
     {
         for (uint8_t i = 0; i < numEdges; i++)
         {
             HalfEdge *e1 = &edgeArray[i];
             HalfEdge *startEdge = e1;
             HalfEdge *currentEdge = e1;
     
             do {
                 // 找到与 currentEdge 的 vEnd 匹配的半边
                 for (uint8_t j = 0; j < numEdges; j++)
                 {
                     HalfEdge *candidate = &edgeArray[j];
                     if (candidate == currentEdge || candidate->face != currentEdge->face)
                         continue; // 跳过非同面半边
     
                     if (currentEdge->vEnd == candidate->vStart && currentEdge->next == NULL)
                     {
                         currentEdge->next = candidate;
                         candidate->prev = currentEdge;
                         break; // 找到后跳出内循环
                     }
                 }
                 currentEdge = currentEdge->next;
             } while (currentEdge != NULL && currentEdge != startEdge);
         }
     }
     void SetHalfEdgeRelations(HalfEdge *edgeArray, uint8_t numEdges)
     {
         SetHalfEdgePairs(edgeArray, numEdges);
         SetHalfEdgeNextPrev(edgeArray, numEdges);
     }
     ```

  7. 计算多面体空间中的最长距离，以保证视点位于体外部。

### 第三章	圆柱及其绘制

>单色Oled上肯定绘制不了曲面，简单的绘制圆柱的线框倒是可以，圆柱旋转时圆面会变成椭圆，因此为了只要在圆上确定四个点(其连线相互垂直)以这四个点绘制椭圆，然后再连接上下圆面长轴的端点，就可以绘制出旋转过程的圆柱

- 示意图：

  ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%89)/008.png)

  1. 提前规定了0、2、4、6即为长轴端点，其余点为短轴端点，如果其绕Y轴方向旋转造成长短轴互换该怎么解决？

     **CzrTuring：**只能限制圆柱不能绕Y轴旋转。。。。。。。

  2. Bresenham算法如何绘制任意角度的椭圆弧？

     **CzrTuring：**这里也暂时没想到好的方法，为了达到消隐操作，就引入了蒙版操作，先绘制两个椭圆，然后以0、2、4、6为端点在这个四边形区域里面填充0，根据背向面法判断可视面，前向面在画一遍椭圆，背向面可以不进行绘制或绘制一个虚线椭圆，之后在连接圆柱的两条直线边。【PS：这个方法很蠢，哎】

- 数据结构定义：

  ```c
  typedef struct
  {
  	Vertex vertices[8];		//顶点数组
  	HalfEdge edges[8];		//半边数组
  	Face	faces[2];		//面数组
  	float maxDistence;		//圆柱空间最长距离
  	float r;				//圆柱半径
  	float h;				//圆柱高
  	MoveValue move;			//运动变换参数
  }Cylinder;
  ```

- 初始化：

  ```c
  void CylinderInit(Cylinder *cyl, float r, float h, float scale, Point2D pc)
  {
  	//1、初始化运动转换参数
  	cyl->move.scale = scale;
  	cyl->move.pc = pc;
  	cyl->move.rot = (RotateValue){0,0,0,0,0,0};
  	//2、创建半边
  	CreateHalfEdge(&cyl->edges[0], &cyl->vertices[0], &cyl->vertices[1]);
  	CreateHalfEdge(&cyl->edges[1], &cyl->vertices[1], &cyl->vertices[2]);
  	CreateHalfEdge(&cyl->edges[2], &cyl->vertices[2], &cyl->vertices[3]);
  	CreateHalfEdge(&cyl->edges[3], &cyl->vertices[3], &cyl->vertices[0]);
  	CreateHalfEdge(&cyl->edges[4], &cyl->vertices[4], &cyl->vertices[5]);
  	CreateHalfEdge(&cyl->edges[5], &cyl->vertices[5], &cyl->vertices[6]);
  	CreateHalfEdge(&cyl->edges[6], &cyl->vertices[6], &cyl->vertices[7]);
  	CreateHalfEdge(&cyl->edges[7], &cyl->vertices[7], &cyl->vertices[4]);
  	//3、创建顶点
  	CreateVertex(&cyl->vertices[0], (Point3D){ -r, 0.5 * h, 0 }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[1], (Point3D){ 0, 0.5 * h, r }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[2], (Point3D){ r, 0.5 * h, 0 }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[3], (Point3D){ 0, 0.5 * h, -r }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[4], (Point3D){ -r, -0.5 * h, 0 }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[5], (Point3D){ 0, -0.5 * h, -r }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[6], (Point3D){ r, -0.5 * h, 0 }, cyl->edges, 8, scale, pc);
  	CreateVertex(&cyl->vertices[7], (Point3D){ 0, -0.5 * h, r }, cyl->edges, 8, scale, pc);
  	//4、创建面，半边形参方向遵循右手定则向外
  	CreateFace(&cyl->faces[0], &cyl->edges[0], (Point3D){ 0, 1, 0 }, cyl->edges, 8);
  	CreateFace(&cyl->faces[1], &cyl->edges[7], (Point3D){ 0, -1, 0 }, cyl->edges,8);
  	//5、初始化半边的其余参数
  	SetHalfEdgeRelations(&cyl->edges[0], 8);
  	//6、计算圆柱空间最长距离
  	cyl->r = r;
  	cyl->h = h;
  	float xD = 2*r;
  	float yD = h;
  	cyl->maxDistence = sqrt(xD*xD+yD*yD);
  }
  ```

- 运动参数更新：

  ```c
  void UpdateCylinder(Cylinder* cyl)
  {
      for (uint8_t i = 0; i < 8; i++)
      {
          RotatePoint3D(&cyl->vertices[i].p3, cyl->move.rot);
          OrtProject(cyl->vertices[i].p3, &cyl->vertices[i].p2, cyl->move.scale, cyl->move.pc);
      }
  }
  ```

- 绘制：

  ```c
  void PlotCylinder(Cylinder* cyl, SSDBuffer buffer, FillMode mode, Plot3Mode pMode)
  {
  	//特殊情况处理：只看到一个圆形
  	if(cyl->vertices[0].p2.x == cyl->vertices[4].p2.x)
  	{
  		if(cyl->vertices[0].p2.y == cyl->vertices[4].p2.y)
  		{
  			PlotCircle(cyl->move.pc,cyl->move.scale*cyl->r, buffer, mode);
  			return;
  		}
  	}
  	//特殊情况处理：只看到一个矩形
  	if(cyl->vertices[1].p2.x == cyl->vertices[3].p2.x)
  	{
  		if(cyl->vertices[1].p2.y == cyl->vertices[3].p2.y)
  		{
  			Point2D p[4] = {cyl->vertices[0].p2,cyl->vertices[2].p2,cyl->vertices[6].p2,cyl->vertices[4].p2};
  			PlotQuadrilateral(p, buffer, Fill1);
  			return;
  		}
  	}
      //1、设置视点
      Point3D viewPoint = {0, 0, cyl->maxDistence};
      //2、绘制椭圆轮廓
  	PlotEllipse(cyl->vertices[0].p2, cyl->vertices[1].p2, cyl->vertices[2].p2, cyl->vertices[3].p2, buffer, mode, False);
  	PlotEllipse(cyl->vertices[4].p2, cyl->vertices[5].p2, cyl->vertices[6].p2, cyl->vertices[7].p2, buffer, mode, False);
  	if(pMode != NoHidden)
  	{
  		//3、清除上下椭圆可能被遮挡的部分
  		Point2D p[4] = {cyl->vertices[0].p2,cyl->vertices[2].p2,cyl->vertices[6].p2,cyl->vertices[4].p2};
  		PlotFillQuadrilateral(p, buffer, Fill0);
  		//遍历面判断其是否被遮挡
  		for(uint8_t i=0;i<2;i++)
  		{
  			if(IsFrontFace(&cyl->faces[i], viewPoint) == True)
  			{
  				PlotEllipse(cyl->vertices[i*4].p2, cyl->vertices[i*4+1].p2, cyl->vertices[i*4+2].p2, cyl->vertices[i*4+3].p2, buffer, mode, False);
  			}
  			else if(pMode == DashedHidden)
  			{
  				PlotEllipse(cyl->vertices[i*4].p2, cyl->vertices[i*4+1].p2, cyl->vertices[i*4+2].p2, cyl->vertices[i*4+3].p2, buffer, mode, True);
  			}
  		}
  	}
  	//绘制侧边直线
  	PlotLine(cyl->vertices[0].p2, cyl->vertices[4].p2, buffer, mode);
  	PlotLine(cyl->vertices[2].p2, cyl->vertices[6].p2, buffer, mode);
  }
  ```

### 第四章	圆锥及其绘制

>与圆柱的绘制思路一样，不在此做过多介绍

- 数据结构：

  ```c
  typedef struct
  {
  	Vertex vertices[5];		//顶点数组
  	HalfEdge edges[4];		//半边数组
  	Face	faces[1];		//面数组
  	float maxDistence;		//圆锥空间最长距离
  	float r;				//圆锥半径
  	float h;				//圆锥高
  	MoveValue move;			//运动变换参数
  }Cone;
  ```

- 初始化：

  ```c
  void ConeInit(Cone *con, float r, float h, float scale, Point2D pc)
  {
  	//1、初始化运动转换参数
  	con->move.scale = scale;
  	con->move.pc = pc;
  	con->move.rot = (RotateValue){0,0,0,0,0,0};
  	//2、创建半边
  	CreateHalfEdge(&con->edges[0], &con->vertices[0], &con->vertices[1]);
  	CreateHalfEdge(&con->edges[1], &con->vertices[1], &con->vertices[2]);
  	CreateHalfEdge(&con->edges[2], &con->vertices[2], &con->vertices[3]);
  	CreateHalfEdge(&con->edges[3], &con->vertices[3], &con->vertices[0]);
  	//3、创建顶点
  	CreateVertex(&con->vertices[0], (Point3D){ -r, 0, 0 }, con->edges, 4, scale, pc);
  	CreateVertex(&con->vertices[1], (Point3D){ 0, 0, -r }, con->edges, 4, scale, pc);
  	CreateVertex(&con->vertices[2], (Point3D){ r, 0, 0 }, con->edges, 4, scale, pc);
  	CreateVertex(&con->vertices[3], (Point3D){ 0, 0, r }, con->edges, 4, scale, pc);
  	CreateVertex(&con->vertices[4], (Point3D){ 0, -h, 0 }, NULL, 0, scale, pc);
  	//4、创建面，半边形参方向遵循右手定则向外
  	CreateFace(&con->faces[0], &con->edges[0], (Point3D){ 0, -1, 0 }, con->edges, 4);
  	//5、初始化半边的其余参数
  	SetHalfEdgeRelations(&con->edges[0], 4);
  	//6、计算圆锥空间最长距离
  	float xD = 2*r;
  	float yD = h;
  	con->r = r;
  	con->h = h;
  	con->maxDistence = sqrt(xD*xD+yD*yD);
  }
  ```

- 更新运动参数：

  ```c
  void UpdateCone(Cone* con)
  {
      for (uint8_t i = 0; i < 5; i++)
      {
          RotatePoint3D(&con->vertices[i].p3, con->move.rot);
          OrtProject(con->vertices[i].p3, &con->vertices[i].p2, con->move.scale, con->move.pc);
      }
  }
  ```

- 绘制：

  ```c
  void PlotCone(Cone* con, SSDBuffer buffer, FillMode mode, Plot3Mode pMode)
  {
  	//特殊情况处理：只看到一个圆形
  	if(con->vertices[4].p2.x == con->move.pc.x)
  	{
  		if(con->vertices[4].p2.y == con->move.pc.y)
  		{
  			PlotCircle(con->move.pc,con->move.scale*con->r, buffer, mode);
  			return;
  		}
  	}
  	//特殊情况处理：只看到一个三角形
  	if(con->vertices[1].p2.x == con->vertices[3].p2.x)
  	{
  		if(con->vertices[1].p2.y == con->vertices[3].p2.y)
  		{
  			Point2D p[3] = {con->vertices[0].p2,con->vertices[2].p2,con->vertices[4].p2};
  			PlotTriangle(p, buffer, Fill1);
  			return;
  		}
  	}
      //1、设置视点
      Point3D viewPoint = {0, 0, con->maxDistence};
      //2、绘制椭圆轮廓
  	PlotEllipse(con->vertices[0].p2, con->vertices[1].p2, con->vertices[2].p2, con->vertices[3].p2, buffer, mode, False);
  	if(pMode != NoHidden)
  	{
  		//3、清除上下椭圆可能被遮挡的部分
  		Point2D p[3] = {con->vertices[0].p2,con->vertices[2].p2,con->vertices[4].p2};
  		PlotFillTriangle(p, buffer, Fill0);
  		//遍历面判断其是否被遮挡
  		if(IsFrontFace(&con->faces[0], viewPoint) == True)
  		{
  			PlotEllipse(con->vertices[0].p2, con->vertices[1].p2, con->vertices[2].p2, con->vertices[3].p2, buffer, mode, False);
  		}
  		else if(pMode == DashedHidden)
  		{
  			PlotEllipse(con->vertices[0].p2, con->vertices[1].p2, con->vertices[2].p2, con->vertices[3].p2, buffer, mode, True);
  		}
  	}
  	//绘制侧边直线
  	PlotLine(con->vertices[4].p2, con->vertices[0].p2, buffer, mode);
  	PlotLine(con->vertices[4].p2, con->vertices[2].p2, buffer, mode);
  }
  ```

### 第五章	六面体及其绘制

- 数据结构：

  ```c
  typedef struct
  {
  	Vertex vertices[8];		//顶点数组
  	HalfEdge edges[24];		//半边数组
  	Face	faces[6];		//面数组
  	float maxDistence;		//六面体空间最长距离
  	MoveValue move;			//运动变换参数
  }Hexahedron;
  ```

- 通用六面体初始化：

  ```c
  void HexahedronInit(Hexahedron *cube, Point3D* p, float scale, Point2D pc)
  {
  	//1、初始化运动转换参数
  	cube->move.scale = scale;
  	cube->move.pc = pc;
  	cube->move.rot = (RotateValue){0,0,0,0,0,0};
  	//2、创建半边
  	CreateHalfEdge(&cube->edges[0], &cube->vertices[0], &cube->vertices[1]);
  	CreateHalfEdge(&cube->edges[1], &cube->vertices[1], &cube->vertices[2]);
  	CreateHalfEdge(&cube->edges[2], &cube->vertices[2], &cube->vertices[3]);
  	CreateHalfEdge(&cube->edges[3], &cube->vertices[3], &cube->vertices[0]);
  
  	CreateHalfEdge(&cube->edges[4], &cube->vertices[1], &cube->vertices[5]);
  	CreateHalfEdge(&cube->edges[5], &cube->vertices[5], &cube->vertices[6]);
  	CreateHalfEdge(&cube->edges[6], &cube->vertices[6], &cube->vertices[2]);
  	CreateHalfEdge(&cube->edges[7], &cube->vertices[2], &cube->vertices[1]);
  
  	CreateHalfEdge(&cube->edges[8], &cube->vertices[4], &cube->vertices[7]);
  	CreateHalfEdge(&cube->edges[9], &cube->vertices[7], &cube->vertices[6]);
  	CreateHalfEdge(&cube->edges[10], &cube->vertices[6], &cube->vertices[5]);
  	CreateHalfEdge(&cube->edges[11], &cube->vertices[5], &cube->vertices[4]);
  
  	CreateHalfEdge(&cube->edges[12], &cube->vertices[0], &cube->vertices[3]);
  	CreateHalfEdge(&cube->edges[13], &cube->vertices[3], &cube->vertices[7]);
  	CreateHalfEdge(&cube->edges[14], &cube->vertices[7], &cube->vertices[4]);
  	CreateHalfEdge(&cube->edges[15], &cube->vertices[4], &cube->vertices[0]);
  
  	CreateHalfEdge(&cube->edges[16], &cube->vertices[3], &cube->vertices[2]);
  	CreateHalfEdge(&cube->edges[17], &cube->vertices[2], &cube->vertices[6]);
  	CreateHalfEdge(&cube->edges[18], &cube->vertices[6], &cube->vertices[7]);
  	CreateHalfEdge(&cube->edges[19], &cube->vertices[7], &cube->vertices[3]);
  
  	CreateHalfEdge(&cube->edges[20], &cube->vertices[1], &cube->vertices[0]);
  	CreateHalfEdge(&cube->edges[21], &cube->vertices[0], &cube->vertices[4]);
  	CreateHalfEdge(&cube->edges[22], &cube->vertices[4], &cube->vertices[5]);
  	CreateHalfEdge(&cube->edges[23], &cube->vertices[5], &cube->vertices[1]);
  	//3、创建顶点
  	for(uint8_t i=0; i<8;i++)
  	{
  		CreateVertex(&cube->vertices[i], p[i], cube->edges, 24, scale, pc);
  	}
  	//4、创建面，半边形参方向遵循右手定则向外
  	CreateFace(&cube->faces[0], &cube->edges[0], (Point3D){ 0, 1, 0 }, cube->edges, 24);
  	CreateFace(&cube->faces[1], &cube->edges[4], (Point3D){ 0, 0, 1 }, cube->edges,24);
  	CreateFace(&cube->faces[2], &cube->edges[8], (Point3D){ 0, -1, 0 }, cube->edges, 24);
  	CreateFace(&cube->faces[3], &cube->edges[12], (Point3D){ 0, 0, -1 }, cube->edges,24);
  	CreateFace(&cube->faces[4], &cube->edges[16], (Point3D){ 1, 0, 0 }, cube->edges, 24);
  	CreateFace(&cube->faces[5], &cube->edges[20], (Point3D){ -1, 0, 0 }, cube->edges,24);
  	//5、初始化半边的其余参数
  	SetHalfEdgeRelations(cube->edges, 24);
  	//6、计算六面体空间最长距离
  	float xD = cube->vertices[0].p3.x - cube->vertices[6].p3.x;
  	float yD = cube->vertices[0].p3.y - cube->vertices[6].p3.y;
  	float zD = cube->vertices[0].p3.z - cube->vertices[6].p3.z;
  	cube->maxDistence = sqrt(xD*xD+yD*yD+zD*zD);
  }
  ```

- 正方体初始化：

  ```c
  void CubeInit(Hexahedron *cube, float scale, Point2D pc, float length)
  {
  	Point3D p[8] =
  	{
  			(Point3D){-0.5*length, 0.5*length, -0.5*length},
  			(Point3D){-0.5*length, 0.5*length, 0.5*length},
  			(Point3D){0.5*length, 0.5*length, 0.5*length},
  			(Point3D){0.5*length, 0.5*length, -0.5*length},
  			(Point3D){-0.5*length, -0.5*length, -0.5*length},
  			(Point3D){-0.5*length, -0.5*length, 0.5*length},
  			(Point3D){0.5*length, -0.5*length, 0.5*length},
  			(Point3D){0.5*length, -0.5*length, -0.5*length},
  	};
  	HexahedronInit(cube, p, scale, pc);
  }
  ```

- 长方体初始化：

  ```c
  void CuboidInit(Hexahedron *cube, float scale, Point2D pc, float length, float width, float Height)
  {
  	Point3D p[8] =
  	{
  			(Point3D){-0.5*length, 0.5*Height, -0.5*width},
  			(Point3D){-0.5*length, 0.5*Height, 0.5*width},
  			(Point3D){0.5*length, 0.5*Height, 0.5*width},
  			(Point3D){0.5*length, 0.5*Height, -0.5*width},
  			(Point3D){-0.5*length, -0.5*Height, -0.5*width},
  			(Point3D){-0.5*length, -0.5*Height, 0.5*width},
  			(Point3D){0.5*length, -0.5*Height, 0.5*width},
  			(Point3D){0.5*length, -0.5*Height, -0.5*width},
  	};
  	HexahedronInit(cube, p, scale, pc);
  }
  ```

- 更新运动参数：

  ```c
  void UpdateHexahedron(Hexahedron* cube)
  {
      for (uint8_t i = 0; i < 8; i++)
      {
          RotatePoint3D(&cube->vertices[i].p3, cube->move.rot);
          OrtProject(cube->vertices[i].p3, &cube->vertices[i].p2, cube->move.scale, cube->move.pc);
      }
  }
  ```

- 绘制六面体：

  ```c
  void PlotHexahedron(Hexahedron* cube, SSDBuffer buffer, FillMode mode, Plot3Mode pMode)
  {
      //1、设置视点
      Point3D viewPoint = {0, 0, cube->maxDistence};
      Bool isDashed = False;
      //2、判断不可视面
      for (uint8_t i = 0; i < 6; i++)
      {
      	if(pMode != NoHidden)
      	{
      		//判断平面是否在视角的后面
      		if(IsFrontFace(&cube->faces[i], viewPoint) == False)
      		{
      			if(pMode == Hidden)	continue;
      			else	isDashed = True;
      		}
      	}
      	//遍历当前面的半边然后绘制线条
      	HalfEdge* temp;
      	temp = cube->faces[i].halfEdge;
      	do
      	{
      	    // 获取起始点和结束点的投影坐标
      	    Point2D start = temp->vStart->p2;  // 投影后的 2D 坐标
      	    Point2D end = temp->vEnd->p2;
      	    if(isDashed == True)
      	    {
      	    	PlotDashedLine(start, end, buffer, 1, 3, mode);
      	    }
      	    else
      	    {
      	    	PlotLine(start, end, buffer, mode);
      	    }
      	    temp = temp->next;
      	}while (temp != cube->faces[i].halfEdge);
      	isDashed = False;
      }
  }
  ```

### 第六章	三维坐标轴及其绘制

- 数据结构：

  ```c
  typedef struct
  {
  	Point3D vertices3[3];		//原始三维坐标点
  	Point2D vertices2[3];		//二维坐标点
  	Point2D pc;					//投影中心点
  }Axes;
  ```

- 绘制：

  ```c
  void Plot3Axes(Point2D pc,uint8_t length,RotateValue rot,SSDBuffer buffer,FillMode mode)
  {
  	Axes worldAxes;
  	worldAxes.pc = pc;
  	worldAxes.vertices3[0] = (Point3D){0,0,length};
  	worldAxes.vertices3[1] = (Point3D){0,length,0};
  	worldAxes.vertices3[2] = (Point3D){length,0,0};
  	for(uint8_t i = 0; i<3; i++)
  	{
  		RotatePoint3D(&worldAxes.vertices3[i], rot);
  		OrtProject(worldAxes.vertices3[i], &worldAxes.vertices2[i], 1.0, pc);
  	}
  	PlotLine(pc,worldAxes.vertices2[0],buffer, mode);
  	PlotLine(pc,worldAxes.vertices2[1],buffer, mode);
  	PlotLine(pc,worldAxes.vertices2[2],buffer, mode);
  	PlotArrow(pc,worldAxes.vertices2[0], buffer, mode, EndArrow);
  	PlotArrow(pc,worldAxes.vertices2[1], buffer, mode, EndArrow);
  	PlotArrow(pc,worldAxes.vertices2[2], buffer, mode, EndArrow);
  }
  ```


## 第三部分	缓动动画的实现

### 第一章	 贝塞尔曲线生成[^2]

- PC端在线生成工具：[贝塞尔曲线|cubic-bezier - css工具](https://cubic-bezier.tupulin.com/)
- Android端工具：https://github.com/venshine/BezierMaker.git

### 第二章	贝塞尔曲线轨迹动画[^3][^4][^5]

#### 第一节	理解

- 三次贝塞尔曲线：

  - 公式：
    $$
    P\left( t \right) =P_0\left( 1-t \right) ^3+P_1t\left( 1-t \right) ^2+P_2t^2\left( 1-t \right) +P_3t^3
    $$

    $$
    \left\{ \begin{array}{l}
    	x\left( t \right) =x_0\left( 1-t \right) ^3+x_1t\left( 1-t \right) ^2+x_2t^2\left( 1-t \right) +x_3t^3\\
    	y\left( t \right) =y_0\left( 1-t \right) ^3+y_1t\left( 1-t \right) ^2+y_2t^2\left( 1-t \right) +y_3t^3\\
    \end{array} \right.
    $$
    
    ```
    @t：表示差值参数
    @P0,P1,P2,P3：表示控制点,P0为曲线起点，P3为曲线结束点，P1和P2为控制点，可以通过贝塞尔曲线工具获取
    ```
    
  - 使用PC端工具，在这个二维坐标系中，其横轴表示时间$$t$$，纵轴表示动画运动的位置$$P$$，并限制了起点坐标为$$(0,0)$$，终点坐标为$$(1.0,0)$$
  
    ![](./TuringUi%E7%B3%BB%E5%88%97(%E4%B8%89)/003.png)
  
- 理解：现在要运动的元素的起始位置对应的就是上图曲线的起点的y值，目标位置对应的就是上图曲线的结束点的y值，上图横坐标为0~1.0的时间t值，因此在设计动画函数时，需要先归一化到上图的坐标系中，在更新位置时也要转换回来。

- 为了避免因每次循环导致动画时间不同的现象，特此引入动态时间因子调整算法，用户可在函数形参中输入动画时间参数以控制动画播放时间。

- 贝塞尔曲线计算函数：

  ```c
  float BezierCurve(float t, float v0, float v1, float v2, float v3)
  {
      float u = 1 - t;
      return (u * u * u * v0 + 3 * u * u * t * v1 + 3 * u * t * t * v2 + t * t * t * v3);
  }
  ```

- 贝塞尔曲线参数因子t计算函数：

  ```c
  float SolveBezierT(float x, float x1, float x2, float epsilon)
  {
      // 使用二分法求解贝塞尔曲线的参数 t
      float x0 = 0.0f, x3 = 1.0f, t = 0.5f;
  
      while (x3 - x0 > epsilon)
      {
          float xt = BezierCurve(t, 0.0f, x1, x2, 1.0f); // 计算当前 t 对应的 x 值
          if (fabs(xt - x) < epsilon)
          {
              return t; // 找到符合条件的 t
          }
          if (xt < x)
          {
              x0 = t; // 调整下限
          }
          else
          {
              x3 = t; // 调整上限
          }
          t = (x0 + x3) * 0.5f; // 更新 t 为中间点
      }
      return t; // 返回最终逼近的 t
  }
  ```


#### 第二节	代码

- 理解：

  1. 将一个数值的变换视为一个动画的运动属性，这个数值既可以是一个方向的位移、也可以是旋转角度、同样也可所以是缩放因子，函数参数等。

  2. 一个动画类型和一个运动属性组成了一个动画元素，在一个动画播放期间，一个运动属性有且只能以一个动画类型进行播放。以小球沿x轴平移为例：x轴位移距离即为运动属性，加速、匀速或减速即为动画类型，同一时刻小球有且只能进行一种运动模式，他不可能既加速，又减速，又匀速的运动。

  3. 一个运动属性，在一个动画中有且只能有唯一的起始位置和结束位置。

  4. 一个动画集合以一个动画开始时间、一个动画持续时间、一个贝塞尔时间参数和多个动画元素组成。在一个动画集合播放期间，某个对象的多个运动属性可以做多种不同的运动方式。例如：一个风车在一段时间内，其既可以沿着x轴做加速运动，也可以沿着y轴做减速运动，沿着z轴做匀速运动，且自身还具有旋转运动。

- 数据结构设计：

  ```c
  typedef struct
  {
  	Point3D p1;				    //贝塞尔控制点1
  	Point3D p2;				    //贝塞尔控制点2
  }AnimType;
  typedef struct
  {
  	float  sPosition;			//起始位置
  	float  ePosition;			//结束位置
  }MoveScale;
  typedef struct
  {
      AnimType* anit;				//动画类型
      MoveScale* scale;			//运动范围
   	float*  Attribute;			//运动属性
  }AnimElement;
  typedef struct
  {
  	uint32_t startTime;     	  	//动画开始时间: 若其值为零则表示动画尚未开始或已经结束
  	uint32_t  duration;			 	//动画持续时间
  	Bool isFinish;					//动画是否播放完成
  	float time;						//贝塞尔时间参数 = 动画已经播放时间/动画持续时间
      uint8_t num;				 	//动画元素的数量
      AnimElement* anie;			     //动画元素数组
  }AnimSet;
  ```

- 动画函数流程：

  1. 根据startTime是否为0，判断是否刚开始运行动画，获取动画开始时的系统滴答计数。

  2. 获取此次调用动画函数的系统滴答计数，依据动画持续时间，计算已经经过的时间。

  3. 依据已经经过的时间与动画运动时间大小，判断动画是否已经完成，若完成则：
    1. 当前位置属性等于目标位置属性；

    2. 将贝塞尔的时间因子参数time置为0.0；

    3. 将动画是否完成标志置为True；

  4. 若动画未完成则：
    1. 计算时间因子参数time：已经经过的时间/动画持续周期；

    2. 计算贝塞尔曲线参数因子t；

    3. 计算贝塞尔曲线的y值；

    4. 计算当前动画位置=动画起始位置+（动画目标位置-动画起始位置）× y；

- 常用动画参数：

  ```c
  #define aLinear (Point3D){1.00, 1.00, 0.00}, (Point3D){0.00, 0.00, 0.00}
  #define aEasing (Point3D){0.25, 0.10, 0.00}, (Point3D){0.25, 1.00, 0.00}
  #define aEasingIn (Point3D){0.42, 0.00, 0.00}, (Point3D){1.00, 1.00, 0.00}
  #define aEasingOut (Point3D){0.00, 0.00, 0.00}, (Point3D){0.58, 1.00, 0.00}
  #define aEasingInOut (Point3D){0.42, 0.00, 0.00}, (Point3D){0.58, 1.00, 0.00}
  ```

- 初始化函数：

  ```c
  void AnimTypeInit(AnimType* anit, Point3D p1, Point3D p2)
  {
  	anit->p1 = p1;
  	anit->p2 = p2;
  }
  void MoveScaleInit(MoveScale* scale, float sp, float ep)
  {
  	scale->ePosition = ep;
  	scale->sPosition = sp;
  }
  void AnimElementInit(AnimElement* anie,AnimType* anit,MoveScale* scale,float* attribute)
  {
  	anie->Attribute = attribute;
  	anie->anit = anit;
  	anie->scale = scale;
  }
  void AnimSetInit(AnimSet* anis,AnimElement* anie,uint8_t num,uint32_t  duration)
  {
  	anis->duration =duration;
  	anis->num = num;
  	anis->startTime = 0;
  	anis->time = 0.0;
  	anis->anie = anie;
  	anis->isFinish = False;
  }
  ```

- 动画集合播放函数：

  ```c
  void Animation(AnimSet* anis)
  {
  	//1、获取动画开始时的系统滴答计数
  	if(anis->startTime == 0 && anis->isFinish == False)
  	{
  		anis->startTime = xTaskGetTickCount();
  		for(uint8_t i=0;i<anis->num;i++)
  		{
  			*(anis->anie[i].Attribute) = anis->anie[i].scale->sPosition;
  		}
  		return;
  	}
      //2、获取当前时间、计算已过时间
      uint32_t currentTime = xTaskGetTickCount();
      uint32_t elapsedTime = currentTime - anis->startTime;
      //3、动画处理
      if (elapsedTime >= anis->duration)
      {
          //动画完成
      	anis->isFinish = True;
      	anis->startTime = 0;
      	anis->time = 0.0f;
  		for(uint8_t i=0;i<anis->num;i++)
  		{
  			*(anis->anie[i].Attribute) = anis->anie[i].scale->ePosition;
  		}
      }
      else
      {
          //动画还未完成
      	anis->time = (float)elapsedTime / (float)anis->duration;
  		for(uint8_t i=0;i<anis->num;i++)
  		{
  	        //使用贝塞尔曲线控制动画的进度
  	        float t = SolveBezierT(anis->time, anis->anie[i].anit->p1.x, anis->anie[i].anit->p2.x, 0.001f);
  	        float y = BezierCurve(t, 0.0f, anis->anie[i].anit->p1.y, anis->anie[i].anit->p2.y, 1.0f);
  	        //根据贝塞尔曲线进度计算当前动画位置
  	        *(anis->anie[i].Attribute) = anis->anie[i].scale->sPosition + (anis->anie[i].scale->ePosition - anis->anie[i].scale->sPosition) * y;
  		}
      }
  }
  ```
  
- 往返循环动画：

  ```c
  /**
    *@ FunctionName: void AnimationRoundTrip(AnimSet* anis,uint8_t num)
    *@ Author: CzrTuringB
    *@ Brief: 将动画设置为往返循环模式
    *@ Time: Jan 10, 2025
    *@ Requirement：
    */
  void AnimBidirectionalLoop(AnimSet* anis,uint8_t num)
  {
  	if(anis->isFinish == True)
  	{
  		for(uint8_t i=0; i<num; i++)
  		{
  			float temp = anis->anie[i].scale->sPosition;
  			anis->anie[i].scale->sPosition = anis->anie[i].scale->ePosition;
  			anis->anie[i].scale->ePosition = temp;
  		}
  		anis->isFinish = False;
  	}
  }
  ```

- 单向循环动画：

  ```c
  /**
    *@ FunctionName: void AnimOneWayLoop(AnimSet* anis)
    *@ Author: CzrTuringB
    *@ Brief: 将动画设置为单项循环模式
    *@ Time: Jan 10, 2025
    *@ Requirement：
    */
  void AnimOneWayLoop(AnimSet* anis)
  {
  	anis->isFinish = False;
  }
  ```

### 第三章	动画案例

#### 第一节	小球的平移[^6]

- 代码：

  ```c
  void OledApp(void)
  {
  	//进行Oled以及TuringUi的初始化
  	vTaskDelay(pdMS_TO_TICKS(50));
  	SSDInit();
  	//设置动画参数
  	MoveScale scale = {47,123};
  	AnimType anit[5];
  	AnimTypeInit(&anit[0], aLinear);
  	AnimTypeInit(&anit[1], aEasing);
  	AnimTypeInit(&anit[2], aEasingIn);
  	AnimTypeInit(&anit[3], aEasingInOut);
  	AnimTypeInit(&anit[4], aEasingOut);
  	AnimElement anie[5];
  	float x1,x2,x3,x4,x5;
  	AnimElementInit(&anie[0], &anit[0], &scale, &x1);
  	AnimElementInit(&anie[1], &anit[1], &scale, &x2);
  	AnimElementInit(&anie[2], &anit[2], &scale, &x3);
  	AnimElementInit(&anie[3], &anit[3], &scale, &x4);
  	AnimElementInit(&anie[4], &anit[4], &scale, &x5);
  	AnimSet anis;
  	AnimSetInit(&anis, anie, 5, pdMS_TO_TICKS(1000));
  	while(1)
  	{
  		//清除缓存内容
  		SSDClean();
  		//绘制边框
  		PlotArcRectangle((Point2D){0,0}, 37, 63, 4, uiDevice.disBuffer, Fill1);
  		PlotString((Point2D){1,53}, "Linear", C6x8, uiDevice.disBuffer);
  		PlotString((Point2D){1,40}, "Easing", C6x8, uiDevice.disBuffer);
  		PlotString((Point2D){1,27}, "EaseIn", C6x8, uiDevice.disBuffer);
  		PlotString((Point2D){1,14}, "EasOut", C6x8, uiDevice.disBuffer);
  		PlotString((Point2D){1,1},  "EInOut", C6x8, uiDevice.disBuffer);
  		PlotCircle((Point2D){x1,57}, 4, uiDevice.disBuffer, Fill1);
  		PlotCircle((Point2D){x2,43}, 4, uiDevice.disBuffer, Fill1);
  		PlotCircle((Point2D){x3,31}, 4, uiDevice.disBuffer, Fill1);
  		PlotCircle((Point2D){x4,18}, 4, uiDevice.disBuffer, Fill1);
  		PlotCircle((Point2D){x5,5}, 4, uiDevice.disBuffer, Fill1);
  		if(anis.isFinish == True)
  		{
  			float temp = scale.ePosition;
  			scale.ePosition = scale.sPosition;
  			scale.sPosition = temp;
  			anis.isFinish = False;
  		}
  		Animation(&anis);
  		//刷新屏幕
  		SSDUpdateScreen();
  	}
  }
  ```


#### 第二节	旋转的立方体[^7]

- 代码：

  ```c
  void OledApp(void)
  {
  	//进行Oled以及TuringUi的初始化
  	vTaskDelay(pdMS_TO_TICKS(50));
  	SSDInit();
  	//初始化立方体
  	Hexahedron cube;
  	CubeInit(&cube, 2, (Point2D){64,32}, 10);
  	//初始化动画参数
  	MoveScale scale1 = {0,360};	//旋转角度变化
  	MoveScale scale2 = {30,390};	//旋转角度变化
  	MoveScale scale3 = {45,405};	//旋转角度变化
  	AnimType anit = {aEasingInOut};
  	AnimElement anie[3];
  	AnimElementInit(&anie[0], &anit, &scale1, &cube.move.rot.xAngle);
  	AnimElementInit(&anie[1], &anit, &scale2, &cube.move.rot.yAngle);
  	AnimElementInit(&anie[2], &anit, &scale3, &cube.move.rot.zAngle);
  	AnimSet anis;
  	AnimSetInit(&anis, anie, 3, pdMS_TO_TICKS(2000));
  	while(1)
  	{
  		//清除缓存内容
  		SSDClean();
  		//立方体
  		UpdateHexahedron(&cube);
  		PlotHexahedron(&cube, uiDevice.disBuffer, Fill1, DashedHidden);
  		if(anis.isFinish == True)
  		{
  			anis.isFinish = False;
  			DebugPrintf("01\n");
  		}
  		Animation(&anis);
  		//刷新屏幕
  		SSDUpdateScreen();
  	}
  }
  ```


#### 第三节	旋转的圆柱[^8]

- 代码：

  ```c
  void OledApp(void)
  {
  	//进行Oled以及TuringUi的初始化
  	vTaskDelay(pdMS_TO_TICKS(50));
  	SSDInit();
  	//初始化圆柱
  	Cylinder cyl;
  	CylinderInit(&cyl, 8, 15, 2.0, (Point2D){64,32});
  	//初始化动画参数
  	MoveScale scale1 = {0,360};	//旋转角度变化
  	MoveScale scale2 = {0,360};	//旋转角度变化
  	AnimType anit = {aEasingInOut};
  	AnimElement anie[2];
  	AnimElementInit(&anie[0], &anit, &scale1, &cyl.move.rot.xAngle);
  	AnimElementInit(&anie[1], &anit, &scale2, &cyl.move.rot.zAngle);
  	AnimSet anis;
  	AnimSetInit(&anis, anie, 2, pdMS_TO_TICKS(2000));
  	while(1)
  	{
  		//清除缓存内容
  		SSDClean();
  		//圆柱
  		UpdateCylinder(&cyl);
  		PlotCylinder(&cyl, uiDevice.disBuffer, Fill1, DashedHidden);
  		if(anis.isFinish == True)
  		{
  			anis.isFinish = False;
  		}
  		Animation(&anis);
  		//刷新屏幕
  		SSDUpdateScreen();
  	}
  }
  ```

#### 第四节	旋转的圆锥[^9]

- 代码：

  ```c
  void OledApp(void)
  {
  	//进行Oled以及TuringUi的初始化
  	vTaskDelay(pdMS_TO_TICKS(50));
  	SSDInit();
  	//初始化圆柱
  	Cone con;
  	ConeInit(&con, 8, 12, 2.0, (Point2D){64,32});
  	//初始化动画参数
  	MoveScale scale1 = {0,360};	//旋转角度变化
  	MoveScale scale2 = {0,360};	//旋转角度变化
  	AnimType anit = {aEasingInOut};
  	AnimElement anie[2];
  	AnimElementInit(&anie[0], &anit, &scale1, &con.move.rot.xAngle);
  	AnimElementInit(&anie[1], &anit, &scale2, &con.move.rot.zAngle);
  	AnimSet anis;
  	AnimSetInit(&anis, anie, 2, pdMS_TO_TICKS(2000));
  	while(1)
  	{
  		//清除缓存内容
  		SSDClean();
  		//圆柱
  		UpdateCone(&con);
  		PlotCone(&con, uiDevice.disBuffer, Fill1, DashedHidden);
  		if(anis.isFinish == True)
  		{
  			anis.isFinish = False;
  		}
  		Animation(&anis);
  		//刷新屏幕
  		SSDUpdateScreen();
  	}
  }
  ```


#### 第五节	不断变化的玫瑰曲线[^10]

- 代码：

  ```c
  void OledApp(void)
  {
  	//进行Oled以及TuringUi的初始化
  	vTaskDelay(pdMS_TO_TICKS(50));
  	SSDInit();
  	//初始化动画参数
  	MoveScale scale = {3,360};	//旋转角度变化
  	AnimType anit = {aEasingInOut};
  	AnimElement anie;
  	float k;
  	AnimElementInit(&anie, &anit, &scale, &k);
  	AnimSet anis;
  	AnimSetInit(&anis, &anie, 1, pdMS_TO_TICKS(5000));
  	while(1)
  	{
  		//清除缓存内容
  		SSDClean();
  		PlotStarCurve((Point2D){64,32}, 2000, 1.5, 2.5, k, 6.0, uiDevice.disBuffer, Fill1);
  		if(anis.isFinish == True)
  		{
  			float temp = scale.ePosition;
  			scale.ePosition = scale.sPosition;
  			scale.sPosition = temp;
  			anis.isFinish = False;
  		}
  		Animation(&anis);
  		//刷新屏幕
  		SSDUpdateScreen();
  	}
  }
  ```

### 第四章	动画函数编写注意事项

- 首先明确屏幕中的运动对象是什么，之后确定运动的类型、范围与持续时间。

- 将持续时间相同、起始时间相同的动画归结为同一个动画集合中。

- 动画参数静态初始化流程：

  ```c
  //1、创建MoveScale变量，可以是他的一个数组
  //2、创建AnimType变量，其表示动画的播放类型，有贝塞尔控制点组成
  //3、创建运动因子，其存储动画的计算结果
  //4、创建动画元素，并对其进行初始化，一个动画类型对应一个运动范围变量、一个动画类型变量、一个运动因子变量，若有动画元素某个参数相同，只需通过改变指针指向即可实现。
  //5、创建动画集合，将动画元素传入
  //第一步
  float attribute[3];								//创建动作因子
  uint8_t fillMode[11] = {8,3,5,2,6,3,4,2,7,3,1};	//外边框填充模式的变化规律
  //第二步
  AnimType anit;									//创建动画类型
  AnimTypeInit(&anit,aEasingInOut);				//动画类型设置为缓入换出
  //第三步
  MoveScale scale[3] = {{0,60},{0,160},{0,10}};	//设置动画的起始位置和结束位置：内圆弧旋转角度、外圆弧旋转角度，边框动作模式
  //第四步
  AnimElement anie[3];
  for(uint8_t i=0;i<3;i++)
  {
  	//初始化动画元素
  	AnimElementInit(&anie[i], &anit, &scale[i], &attribute[i]);
  }
  //第五步
  AnimSet anis;
  AnimSetInit(&anis, &anie[0], 3, pdMS_TO_TICKS(1500));//设置动画时间为1.5s
  ```

- 动画参数动态初始化流程：

  ```c
  //1、创建MoveScale变量，可以是他的一个数组
  //2、创建AnimType变量，其表示动画的播放类型，有贝塞尔控制点组成
  //3、创建运动因子，其存储动画的计算结果
  //4、创建动画元素，并对其进行初始化，一个动画类型对应一个运动范围变量、一个动画类型变量、一个运动因子变量，若有动画元素某个参数相同，只需通过改变指针指向即可实现。
  //5、创建动画集合，将动画元素传入
  float* attribute = pvPortMalloc(sizeof(float));			//创建运动因子
  if(attribute == NULL)	return;
  AnimType *anit = pvPortMalloc(sizeof(AnimType));		//创建动画类型
  if(anit == NULL)	return;
  AnimTypeInit(anit,aEasingInOut);						//动画类型设置为缓入换出
  MoveScale *scale = pvPortMalloc(sizeof(MoveScale));		//创建运动范围
  if(scale == NULL)	return;
  scale->sPosition = 0;
  scale->ePosition = label->disLength;
  AnimElement *anie = pvPortMalloc(sizeof(AnimElement));	//创建动画元素
  if(anie == NULL)	return;
  AnimElementInit(anie, anit, scale, attribute);			//初始化动画元素
  label->anis = pvPortMalloc(sizeof(AnimSet));			//创建动画集合
  if(label->anis == NULL)	return;
  AnimSetInit(label->anis, anie, 1, pdMS_TO_TICKS(label->numChar * 200));
  ```

---

## 参考

[^1]:[半边数据结构解析-CSDN博客](https://blog.csdn.net/outtt/article/details/78544053)
[^2]:[贝塞尔曲线|cubic-bezier - css工具](https://cubic-bezier.tupulin.com/)
[^3]:[缓动——丝滑动效的奥秘_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1n44y1J7wt/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^4]:[如何用数学让动画变得极致丝滑？【中英双字】Giving Personality to Procedural Animations using Math_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1wN4y1578b/?vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^5]:[从 A 到 B 的最完美运动。 《完美动画》_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1DB4y1T7kc/?spm_id_from=333.999.0.0)
[^6]:[OLED缓动动画（一）移动的小球_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1muCbYrEwZ/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^7]:[OLED缓动动画（二）旋转的立方体_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1HTCaY6Ee4/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^8]:[OLED缓动动画（三）旋转的圆柱_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1scCaYyE4i/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^9]:[OLED缓动动画（四）旋转的圆锥_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1WFCaYLE9b/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
[^10]:[OLED缓动动画（五）玫瑰曲线的变化_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1CpChY6EpU/?spm_id_from=333.999.0.0&vd_source=c706c24a5bec5f2b0d006d2505e30519)
