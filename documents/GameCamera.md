# 游戏引擎摄像机设计目的
创建可以用键盘wasd和鼠标移动改变视角的摄像机  
图形变换中，物体坐标是在世界坐标系下的，它并不是屏幕上应该显示的位置坐标  
观察矩阵(View)：把世界坐标换成观察坐标(也就是摄像机坐标系)  
定义摄像机坐标系：摄像机自己为原点，看向-z方向，并且以y轴为向上方向(因为我们的目的就是把任意向量转换到摄像机坐标系下，这样定义之后的LookAt矩阵(也就是view矩阵)的推导最简单)  
(换句话说，摄像机前进的时候，总是沿着自己坐标系的z轴方向前进)  
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


# 绕着某个点运动的摄像机(LookAt模式)
使用glm::lookAt函数生成view。第一行是Camera Position，第二行是Look Target Position，第三行是上方向，一般就是(0,1,0)  
```c++
glm::vec3 cameraPos = Position;
glm::vec3 cameraUp = DirectionUp;
matrices.view = glm::lookAt(cameraPos, TargetPosition, cameraUp);
```

# 自由移动摄像机(FreeMove模式)
这个模式其实是在LookAt模式上修改了一下，Look Target Position永远设置在Camera Position的前方  
```c++
glm::vec3 cameraPos = Position;
glm::vec3 cameraFront = DirectionFront;
glm::vec3 cameraUp = DirectionUp;
matrices.view = glm::lookAt(cameraPos, cameraPos + cameraFront, cameraUp);
```

# Direction的计算方法
上面该公式中提到了Direction相关变量，这个应该如何计算呢？  
对于普通的物体，一般做法是通过pitch/yaw/roll获得四元素向量，然后转化成RotationMatrix，然后用这么矩阵乘以坐标轴基向量获得DirectionFront, DirectionUp和DirectionLeft三个向量。  
```
DirectionLeft = glm::normalize(RotationMatrix * glm::vec4(1,0,0,0));
DirectionUp = glm::normalize(RotationMatrix * glm::vec4(0,1,0,0));
DirectionFront = glm::normalize(RotationMatrix * glm::vec4(0,0,1,0));
```
但是对于摄像机比较特殊，因为如上文所述，摄像机的DirectionUp定死了为(0,1,0)(或者(0,-1,0)), 相当于没有roll这个分量了  
对于第一人称视角的游戏，一般用如下特殊方法设计摄像机的方向计算  
```
if(Rotation.x > 89.0f) Rotation.x = 89.0f;
if(Rotation.x < -89.0f) Rotation.x = -89.0f;
DirectionFront.x = cos(glm::radians(Rotation.x)) * cos(glm::radians(90-Rotation.y));
DirectionFront.y = sin(glm::radians(Rotation.x));
DirectionFront.z = sin(glm::radians(90-Rotation.y)) * cos(glm::radians(Rotation.x));
DirectionFront = glm::normalize(DirectionFront);
DirectionUp = glm::vec3(0,-1,0); //vulkan use different NDC compare to opengl
DirectionLeft = glm::normalize(glm::cross(DirectionUp, DirectionFront));
```
这样的好处是pitch up和pitch down非常准确；yaw的量也非常自然，符合第一人称对视角转动的体验  
需要注意的是pitch的量要限制在-89~89度之间，不然视角会跳转(这样也是更符合第一人称的体验)  


# 鼠标推动视角移动
我们已经可以使用键盘改变摄像机的位置和朝向了。为了做出一款更像是第一人称游戏的摄像机视角，下面介绍用鼠标改变Camera Direction的方法以实现鼠标的方向改变（以SDL3为例）。  
思路是按下鼠标左键的时候，开启第一人称视角。这时候可以用鼠标位移更改摄像机的pitch和yaw值。再按一下又可以锁定视角。  
```
case SDL_EVENT_MOUSE_BUTTON_UP:
    CApplication::mainCamera.AngularVelocity.x = 0;
    CApplication::mainCamera.AngularVelocity.y = 0;
    bFirstPersonMouseRotate = !bFirstPersonMouseRotate;
    break;
case SDL_EVENT_MOUSE_MOTION:
    if(bFirstPersonMouseRotate){
        CApplication::mainCamera.AngularVelocity.x = mouse_sensitive*(m_windowCenterX - event.pmotion.y);
        CApplication::mainCamera.AngularVelocity.y = mouse_sensitive*(-m_windowCenterY + event.pmotion.x);
    }
    break;
```

# Camera in Vulkan
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/gameCamera.png" alt="alt text" width="600" height="450">  
</p>  

# Reference
使用OpenGL底层实现游戏引擎中的摄像机  
https://blog.csdn.net/qq_42428486/article/details/120420352  
https://www.bilibili.com/video/BV1X7411F744?spm_id_from=333.788.videopod.episodes&vd_source=e9d9bc8892014008f20c4e4027b98036&p=4  
