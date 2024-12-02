# 游戏引擎摄像机设计目的
创建可以用键盘wasd和鼠标移动改变视角的摄像机  
图形变换中，物体坐标是在世界坐标系下的，它并不是屏幕上应该显示的位置坐标  
观察矩阵(View)：把世界坐标换成观察坐标(也就是摄像机坐标系)  
定义摄像机坐标系：摄像机自己为原点，看向-z方向，并且以y轴为向上方向(因为我们的目的就是把任意向量转换到摄像机坐标系下，这样定义之后的LookAt矩阵的推导最简单)  
实际上摄像机在世界坐标系里有它自己的位置，观察的方向，right方向和up方向  
当把物体坐标换到摄像机坐标系之后，新的物体坐标就是屏幕上应该观察到的位置了，可以直接绘制(一般还会乘一个P矩阵)  

假设摄像机初始在世界坐标系的(0,0,3)。z轴从屏幕指向人类。如果摄像机向z轴正方向移动，实际上的效果是摄像机在后退。  
定义摄像机指向的对象为(0,0,0)。摄像机的方向就可以求出来了。  
```c++
glm::vec3 cameraPos = glm::vec3(0.0f,0.0f,3.0f);  
glm::vec3 cameraTarget = glm::vec3(0.0f, 0.0f, 0.0f);  
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);  
```
注意，这里的cameraDirection指向从摄像机到目标的相反方向。把它作为第一个轴变量  

另外两个轴变量：  
```c++
glm::vec3 up = glm::vec3(0.0f,1.0f,0.0f);
glm::vec3 cameraRight = glm::normalize(glm::cross(up, cameraDirection);
```
up一定跟cameraRight垂直，但不一定跟cameraDirection垂直，所以还要做一次叉乘，得到的cameraUp同时跟cameraDirection, cameraRight垂直。  
```c++
glm::vec3 cameraUp = glm::cross(cameraDirection, cameraRight);  
```
这样，就得到了互相垂直的三个camera轴变量。  

下面要创建LookAt矩阵
这个矩阵的好处是使用了如下原理：一个空间坐标，如果已知三个互相垂直的轴(摄像机轴)和一个平移向量(摄像机位置)，则可以用它们创建一个矩阵，可以使用这个矩阵乘以任何向量来变换到相应的空间。  
LookAt = [Rx Ry Rz 0; Ux Uy Uz 0; Dx Dy Dz 0; 0 0 0 1] * [1 0 0 -Px; 0 1 0 -Py; 0 0 1 -Pz; 0 0 0 1]  
注意到这里的P前面有负号，因为希望把世界平移到与自身移动的相反方向。  

以上几个公式在GLM里面有现成的实现  
```c++
glm::mat4 view;
view = glm::lookAt(
    glm::vec3(0.0f, 0.0f, 3.0f), //Camera Position 
    glm::vec3(0.0f, 0.0f, 0.0f), //Camera Target
    glm::vec3(0.0f, 1.0f, 0.0f)); //up
```

LookAt矩阵的推导方法：实际上就是，第一步把摄像机移动到原点位置，第二步旋转到Front指向z负方向，并且Up指向y正方向的变换矩阵。  
第一步很简单。正常来讲，第二步很难。但第二步的逆矩阵很好写，因为z负方向和y方向的叉乘是x方向，所以这三个方向其实就是摄像机矩阵的基变换，非常容易推导。之后把这个矩阵取逆就可以了。  
又因为旋转矩阵是正交矩阵，所以取逆就等于取转置，也很简单。  


# 绕着某个点运动的摄像机
```c++
GLfloat radius = 10.0f;
GLfloat camX = sin(glfwGetTime()) * radius;
GLfloat camZ = cos(glfwGetTime()) * radius;
glm::mat4 view;
view = glm::lookAt(
    glm::vec3(camX, 0.0f, camZ),
    glm::vec3(0.0f, 0.0f, 0.0f), 
    glm::vec3(0.0f, 1.0f, 0.0f)); 
```

# 自由移动摄像机
```c++
glm::vec3 cameraPos = glm:：vec3(0.0f, 0.0f, 3.0f);
glm::vec3 cameraFront = glm::vec3(0.0f, 0.0f, -1.0f);
glm::vec3 cameraUp = glm::vec3(0.0f, 1.0f, 0.0f);
view = glm::lookAt(cameraPos, cameraPos + cameraFront, cameraUp);
```
更新cameraPos的方式举例  
KEY_W: cameraPos += cameraSpeed * cameraFront;  
KEY_S:  cameraPos -= cameraSpeed * cameraFront;  
KEY_A: cameraPos -= glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;  
KEY_D: cameraPos += glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;  

# 视角移动
我们已经可以使用剪片改变摄像机的位置了，下面使用鼠标改变cameraFront以实现方向改变。  
WIP  

# Reference
使用OpenGL底层实现游戏引擎中的摄像机  
https://blog.csdn.net/qq_42428486/article/details/120420352  
https://www.bilibili.com/video/BV1X7411F744?spm_id_from=333.788.videopod.episodes&vd_source=e9d9bc8892014008f20c4e4027b98036&p=4  
