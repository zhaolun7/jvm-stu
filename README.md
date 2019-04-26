# jvm-stu
#### 1.基础
##### 对象的结构
* Header
  + 自身运行时数据（Mark Word）
    - 哈希值 GC分代年龄 锁状态标志 线程持有的锁 偏向线程ID 偏向时间戳
  + 类型指针 -> 指向类的元数据
* InstanceData
  + Hotspot实现将长度相同的字段分配到一块
* Padding -> 对齐填充，占位符
  + hotspot要求对象的起始地址必须是8个字节的整数倍

##### 对象的访问定位
* 使用句柄
指向句柄池，通过句柄池找到对象的真正地址
* 直接指针

#### 2. 垃圾回收
* 如何判定对象为垃圾对象
  - 引用计数法
  - 可达性分析法
* 如何回收
  - 回收策略
    + 标记-清除算法
    + 复制算法
    + 标记-整理算法
    + 分代收集算法
  - 常见的垃圾回收器
    + serial
    + Parnew
    + Cms
    + G1
* 何时回收
##### 2.1 如何判定对象为垃圾对象
2.1.1 引用计数法
在对象中添加一个引用计数器，当有地方引用这个对象的时候，引用计数的值加1，当引用失效的时候，计数器的值就-1
java收集器基本不使用这种方式(循环引用问题）
**打印垃圾回收器的回收日志参数**：
&#8195;-verbose:gc
&#8195;-XX:PrintGCDetails
```java
package com.rcp.test7;

public class Test {
	Test mem;
	public Test() {
		byte[] m = new byte[20<<20];
	}
	
	public static void main(String[] args) throws Exception {
		Test t1 = new Test();
		Test t2 = new Test();
		//循环引用
		t1.mem = t2;
		t2.mem = t1;
		//断开引用
		t1 = null;
		t2 = null;
		
		System.gc();
	}
}
```
>[GC (System.gc()) [PSYoungGen: 22476K->648K(38400K)] 42956K->21136K(125952K), 0.0008089 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
>[Full GC (System.gc()) [PSYoungGen: 648K->0K(38400K)] [ParOldGen: 20488K->545K(87552K)] 21136K->545K(125952K), [Metaspace: 2744K->2744K(1056768K)], 0.0038244 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
>Heap
> PSYoungGen      total 38400K, used 333K [0x00000000d5e00000, 0x00000000d8880000, 0x0000000100000000)
>  eden space 33280K, 1% used [0x00000000d5e00000,0x00000000d5e534a8,0x00000000d7e80000)
>  from space 5120K, 0% used [0x00000000d7e80000,0x00000000d7e80000,0x00000000d8380000)
>  to   space 5120K, 0% used [0x00000000d8380000,0x00000000d8380000,0x00000000d8880000)
> ParOldGen       total 87552K, used 545K [0x0000000081a00000, 0x0000000086f80000, 0x00000000d5e00000)
>  object space 87552K, 0% used [0x0000000081a00000,0x0000000081a88638,0x0000000086f80000)
> Metaspace       used 2751K, capacity 4486K, committed 4864K, reserved 1056768K
>  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K

2.1.2 可达性分析算法
定义垃圾回收的根节点，从栈帧中的临时变量表开始查找
从GCRoot向下搜索，如果gc对象没有引用链相连接，则需要回收
**作为GCRoot的对象：**
- 虚拟机栈
- 方法区的类属性所引用的对象
- 方法区中常量所引用的对象
- 本地方法栈中引用的对象

#### 2.2 回收算法
##### 2.1 标记-清除算法
效率问题
空间问题（碎片）
![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2ak6d867wj30d60a4q57.jpg)
##### 2.2 复制算法
解决标记-清除算法的效率问题
* 线程共享区域：堆，方法区
    - 堆：
      + 新生代
        * Eden 伊甸园
        * Suvivor 存活区
        * Tenured Gen 年龄较老到这个区域
      + 老年代
      
 内存浪费50%
 ![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2akgbx4yzj30l00a0dgw.jpg)
 解决方案：（内存浪费10%）
 对象分配在Eden区，如果不够分配到一个Suvivor区，
 进行垃圾收集时存活放到另一个suvivor区，Eden和此Suvivor清空
 ![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2akhngi0nj30mf0a4gly.jpg)

##### 2.3 标记-整理算法
复制算法适用场景：存活率10%，新生代
标记整理算法：存活率高，老年代
**标记-整理-清除算法**：先标记，然后需要回收的对象向回收区域复制，不需要回收的对象复制到不需回收的内存区域，然后直接清除掉待回收内存区域。执行效率高
![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2akwi675vj30n4093gni.jpg)
##### 2.4 分代算法
标记-整理算法 与 复制算法 结合
新生代：使用 复制算法
老年代：使用 标记-整理算法

#### 3 垃圾收集器
##### 3.1 Serial收集器
java1.3之前主要的收集器
- 最基本，发展最悠久
- 单线程
- 适合小型桌面应用
停止所有用户线程，进行垃圾收集后，继续用户线程
![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2albxovp9j30b204saa6.jpg)

##### 3.2 ParNew收集器
与Serial类似，但是在垃圾收集时是多线程同时进行
适合服务器，不适合桌面应用，单线程性能不如Serial

##### 3.3 parallel收集器
Parallel Scavenge收集器 与 ParNew 相同点
- 复制算法（新生代收集器）
- 都是多线程收集器
不同点：
- ParNew关注点是缩短垃圾回收时间
- Parallel目标是达到一个可控制的吞吐量
吞吐量：CPU用于运行用户代码的时间与CPU消耗的总时间的比值
吞吐量 = 执行用户代码的时间 / （执行用户代码时间 + 垃圾回收所占用的时间）
`-XX:MaxGCPauseMillis`  :垃圾收集器停顿时间设置合理的最大停顿时间，并不是越小越好
`-XX:GCTimeRatio `: 吞吐量大小（0，100）值越大表示吞吐量越大

##### 3.4 CMS收集器
Concurrent Mark Sweep
![](http://ww1.sinaimg.cn/large/006y5ilOgy1g2b6hnbv8tj30kw08iwfp.jpg)
**工作过程**:
  - 初始标记
  - 并发标记
  - 重新标记
  - 并发清理

优点
  - 并发收集
  - 低停顿

缺点
  - 占用大量cpu资源
  - 无法处理浮动垃圾（需要下次打扫）
  - 出现Concurrent Mode Failure
  - 空间碎片
##### 3.5 G1收集器
Garbage First
优势：
  - 并行和并发（利用多核优势）
  - 分代收集
  - 空间整合
  - 可预测的停顿（借助remember table定位region,避免扫描整个堆）

步骤：
- 初始标记（并行）
- 并发标记
- 最终标记
- 筛选回收 

与CMS比较：
- 吞吐量不如CMS

G1适用于内存比较大（几十G）的应用

#### 4.内存分配策略
- 优先分配到Eden区域
- 大对象直接分配到老年代
- 长期存活的对象分配到老年代
- 空间分配担保（借老年代空间）
- 动态对象年龄判断

##### 4.1 优先分配到Eden
java检测内存大于2g 就是server版 使用parallel收集器
```java
package com.rcp.test8;

public class Main {
	public static void main(String[] args) {
		byte []b1 = new byte[2*1024*1024];
		byte []b2 = new byte[2*1024*1024];
		byte []b3 = new byte[2*1024*1024];
		byte []b4 = new byte[7*1024*1024];
		System.gc();
	}
}
```
启动参数：`-verbose:gc -XX:+PrintGCDetails -Xms20M -Xmx20M -XX:+UseSerialGC -Xmn10M -XX:SurvivorRatio=8`
结果可以看出 7m分配到Eden, 3个2m到了old区（内存分配担保）
![](https://ws1.sinaimg.cn/large/007c6W5Igy1g2g5ondncxj30w008k3za.jpg)

##### 4.2 大对象直接分配到老年代
-XX:PretenureSizeThreshold=6M
分配大对象时Eden放不下直接分配到老年代
```java
public class Main9 {
	public static void main(String[] args) {
		byte []b1 = new byte[7*1024*1024];
	}
}
```
设置参数`-verbose:gc -XX:+PrintGCDetails -Xms20M -Xmx20M -XX:+UseSerialGC -Xmn10M -XX:SurvivorRatio=8`
