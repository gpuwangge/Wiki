# 判断 triangle 的 frontface/backface：几何上判断是否朝向相机  
设三角形三个顶点为p0​,p1​,p2​，相机位置为C
```
N=(p1−p0)×(p2−p0)
V=C−p0
```
再计算：
```
d=N⋅V
```