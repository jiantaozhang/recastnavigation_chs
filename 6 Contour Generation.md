
## 生成 Contour
这一篇讲述建立NavMesh的第三个阶段，构建一系列简单多边形（凸多边形、凹多边形）来表示原始几何体的可行走表面

这里的Contour，也就是轮廓，依然是基于体素空间，这是从 体素空间 转向 矢量空间 的第一步

---
### Searching for Region Edges
从 heightfield 到 contour，最大的概念上的转变就是 关注点从 span surface 转移到了 span edge

对于 Contour，关注的是 span edge。这里会有两种类型的edge
- region edge 是指两个不同region之间的边界
- internal edge 是指同一个region内部的两个子区域的边界

下面的各种例子将会基于2D来解释，看起来方便些，后面会回到3D空间
<img src="img/6/cont_01_surfedgecomp.png" />

这一步，会将edge的类型进行区分，region edges / internal edges

这个比较容易处理，遍历所有的span，检查 axis-neighbors，如果neighbor不是在同一个区域的，那么这两个span之间的边就是 region edge
<img src="img/6/cont_02_edges.png" />

---
### Finding Region Contours
经过上一步的处理之后，现在有了每个span edge是不是region edge的信息，接下来构建 Contour 就可以基于这个信息

遍历所有 spans，如果span有一个 region edge，就进行如下操作：
1. 面向已知的 region edge，添加到 Contour 的数据结构中
<img src="img/6/cont_03_walkedge_01.png" />

2. 顺时针旋转90°，如果指向的还是region edge，那么添加到 Contour 中，如果指向的是 internal edge，朝着指向前进一格
<img src="img/6/cont_03_walkedge_02.png" />

3. 上一步前进之后，逆时针旋转90°，如果指向是的 region edge，那么添加到Contour中，同时重复第2步的操作。 如果指向的是 internal edge，朝指向前进一格
<img src="img/6/cont_03_walkedge_03.png" />

其实就是反复上面第2、3步，指向region edge就顺时针转90，如果指向internal edge，就前进一格后逆时针转90，继续判断edge类型，反复如此，一直回到最初的span 和 最初的朝向 为止

---
### Moving from Edges to Vertices
事实上，为了将数据从 体素空间 带回到 矢量空间，我们需要的是顶点，而不只是edge

这里(x,z)的坐标很容易获取，对每个edge，直接取corner的数据就好了 （因为corner的数据也是会预先存在span里面的）
<img src="img/6/cont_04_vertpicxz.png" />

取y值的话就会有个小细节，如下图所示，y值最多会有四个潜在的值
<img src="img/6/cont_05_edgevertysel.png" />

这里会直接采用最大的y值，原因如下
- 保证了这个位置最终的顶点(x,y,z) 一定是高于 原始物体在这个位置的顶点
- 统一标准
<img src="img/6/cont_05_edgevertysel_02.png" />

---
### Simplifying the Contours
此时，所有region的contour已经构建好了。这些轮廓都是由上一步中，从span的corner中采集到的顶点构成。下面是一张宏观的图

注意，有两种类型的 contour
- 两个相邻 region 之间的 contour
- 和 无效空间 之间的 contour （其实就是一块可行走区域的边界）
<img src="img/6/cont_06_detailoverview.png" />

我们真的需要这么多顶点吗？即使是一条直线轮廓，也会导致一连串的顶点出现。所以答案一定是否定

唯一真正需要顶点的地方就是 多个region的交界处
<img src="img/6/cont_07_mandatoryverts.png" />

这里做一个非常简单的简化，除了必要的顶点，丢掉剩下没有用的顶点
<img src="img/6/cont_08_portalsimplification.png" />

