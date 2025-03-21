# const
const是C++98/03标准引入的关键字。  
首先介绍C语言中“只读变量”和“常量”的区别。  
- 编译时常量(compile-time constant)：编译时常量被存储在内存中的只读区，任何方式都无法修改。在编译的时候，这个值就确定了。用处是有些场合比如数组大小、模板参数必须在编译阶段确定，所以要用这个。  
- 运行时常量(run-time constant): 跟上述差不多，但是不能在编译阶段确定数值，需要程序开始运行了才能计算出来。  
- 编译时确定值的只读变量(compile-time read-only variable)：变量在内存变量区中开辟一个地方来存值。编译器不允许直接使用只读变量修改这个值。但是可以通过一些手段间接修改(比如创建一个指向同样地址的常规指针变量)。有些只读变量在编译的阶段就可以确定。一旦确定就不能直接改了。  
- 运行时确定值的只读变量(run-time read-only variable)：跟上述差不多，但是在编译阶段因为没有具体值而不能确定。它可以运行后被初始化，一旦初始化之后就不能直接改了。  
```c++
const int compileTimeConst = 10; //编译时常量
```
```c++
int someValue = 30;
const int runTimeConst = someValue; //运行时常量
```
```c++
class MyClass{
  MyClass(){
    value = 10;
  }
  const int value; //编译时确定值的只读变量
};
```
```c++
class MyClass{
  MyClass(){}
  const int value; //运行时确定值的只读变量
};

```
总结：当使用const修饰量的时候，可以出现四种不同情况。其中只有编译时能确定值的两种情况可以用来初始化数组大小和模板参数。  
正是因为const的使用容易造成混淆，C++11引入新关键字constexpr，用来声明在编译时就能确定的量。编译器在遇见constexpr修饰的量(或constexpr修饰的函数)的时候，会尝试在编译阶段计算出确定的值。  
所以正确的使用习惯是如果是在编译阶段就能确定的值，一律用constexpr而不是const。  

const也可以用于修饰类的成员函数(const关键字被放置在成员函数的末尾)，表示这个成员函数只能读取类内的非静态成员变量，但不能修改(无论这个非静态成员变量是不是只读)。  
const成员函数也不能调用其他非const成员函数。  
如果类有一个成员函数仅提供查询而不改变类的内容，设置成const成员函数可以起到保护类的作用。  
但是const成员函数是可以修改静态成员变量的(因为静态成员变量属于类，而不属于具体的对象)  
(静态成员变量是使用static修饰的成员变量)  


# reinterpret_cast
reinterpret_cast是类型转换运算符。它可以转换任意类型的指针。  
- 将一种指针类型转换成另一种指针类型  
- 把指针转换成整数，或把整数转换成指针  
- 将一种函数指针类型转换为另一种函数指针类型  
```c++
struct A { int x; };
struct B { int y; };
int main(){
  A a;
  a.x = 10;
  B* b = reinterpret_cast<B*>(&a); //&a是A类型的指针地址，这里转换为B*也就是B类型的指针地址，并赋值给b
  std::cout << "b->y: " << b->y << std::endl; //输出是10，因为a.x和b.y是同一个内存地址
  //正常来讲B类型的指针是不能调用A的成员函数的，这里打破了保护，是有一定风险的
}
```
```c++
int a = 42;
int * intPtr = &a;
char* charPtr = reinterpret_cast<char*>(intPtr); //把整数指针转换成字符指针
uintptr_t intAddress = reinterpret_cast<uintptr_t>(intPtr); //把整数指针转换成整数(其实是一样的，指针一般是十六进制，整数一般是十进制)
int *newIntPtr = reinterpret_cast<int*>(intAddress); //又把整数转成了整数指针
std::cout<<*newIntPtr; //这里打印的还是42，表示转换没有错误
```
```c++
typedef void (*FuncType1)(int);
typedef void (*FuncType2)(double);
void func1(int x){std::cout<<x<<std::endl;}
int main(){
  FuncType1 f1 = func1;
  FuncType2 f2 = reinterpret_cast<FuncType2>(f1); //这里把f1从FuncType2转换成FuncType1类型的函数指针
  f2(3.14); //输出结果为3
}
//结论：我们在这里把double类型的数据传到了本应该支持int的func1(int)函数里(当然3.14截断成了3)
```


