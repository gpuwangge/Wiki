Shell 是一个用 C 语言编写的程序，它是用户使用 Linux 的桥梁。Shell 既是一种命令语言，又是一种程序设计语言。 Shell 是指一种应用程序，这个应用程序提供了一个界面，用户通过这个界面访问操作系统内核的服务。  
。Shell有很多种，比如sh, bash, csh, ksh等。sh是第一种Unix Shell。  
Shell 脚本（shell script），是一种为 shell 编写的脚本程序。 业界所说的 shell 通常都是指 shell 脚本，但读者朋友要知道，shell 和 shell script 是两个不同的概念。  

# 创建sh脚本的方法
打开文本编辑器，新建一个文件命名为name.sh就可以了。  

# 运行sh脚本的方法
> ./name.sh  

./的意思是去当前目录下找(否则Linux会去PATH下面找)  

如果没有执行权限的话是不能运行的。没有权限的话可以按照如下操作。  
> chmod +x ./name.sh  

翻译器错误：有时候新写的脚本运行出错，这时候把文件从Windows(CR LF)换成Unix(LF)就可以解决了。  
CR LF是回车换行的缩写，表示将光标移动到下一行的行首，并插入一个新行。 而LF只是换行的意思，表示将光标移动到下一行的行首，不会插入新行。
Dos和Windows采用回车+换行CR/LF表示下一行。  
而UNIX/Linux采用换行符LF表示下一行。  
苹果机(MAC OS系统)则采用回车符CR表示下一行。  
CR用符号r表示，十进制ASCII代码是13，十六进制代码为0x0D  
LF使用n符号表示，ASCII代码是10，十六制为0x0A  
所以Windows平台上换行在文本文件中是使用0d 0a两个字节表示，而UNIX和苹果平台上换行则是使用0a或0d一个字节表示。  
一般操作系统上的运行库会自动决定文本文件的换行格式。如一个程序在Windows上运行就生成CR/LF换行格式的文本文件，而在Linux上运行就生成LF格式换行的文本文件。在一个平台上使用另一种换行符的文件文件可能会带来意想不到的问题，特别是在编辑程序代码时。有时候代码在编辑器中显示正常，但在编辑时却会因为换行符问题而出错。很多文本/代码编辑器带有换行符转换功能，使用这个功能可以将文本文件中的换行符在不同格式单互换。  


# 语法
## 关键字：#!  
#!不是注释，它是告诉系统，其后路径所指定的程序是解释此脚本文件的shell程序。  
举例：  
> #!/bin/sh  
> #!/bin/bash  

如果不想写这一句，则需要运行解释器才能运行脚本，比如：  
> /bin/sh name.sh  

## 关键字：echo
echo相当于print  
> echo "Hello World!"  

## 关键字：&
默认情况下，进程是前台进程，这时就把Shell给占据了，我们无法进行其他操作，对于那些没有交互的进程，很多时候，我们希望将其在后台启动，可以在启动参数的时候加一个’&'实现这个目的。  
&放在启动参数后面表示此进程为后台进程。  
&链接的进程以多线程模式运行(如果硬件支持的话)。  
> (./program1 -i arg) & (./program2 -i arg)


## 关键字：;
;符号跟&不同。以;分隔的命令会依照次序运行。就是说第一个command执行完之后才执行下一个command。  
;符号在这里的作用其实只是让不同的命令写在同一行，并不会多线程并行。  
> command1; command2; command3  

## 关键字：for
```
for i in {2..10}
do
  echo "output: $i"
done
```
```
max=10
for i in {2..$max}
do
  echo "output: $i"
done
```
可以使用seq关键字  
```
for i in $(seq 1 10)
do
  echo "output: $i"
done
```


## 关键字：()
单小括号在这里的作用是，把括号中的命令开做一个子shell顺序执行。俗称命令组。  

()可以配合&产生并行的命令组  
```
for i in $(seq 1 10)
do
  (
    echo "output: $i"
    echo "This is a single thread run"
  ) &
done
wait
```


## 关键字: wait
使用()&产生的并行命令组分别运行在不同的线程上。  
wait指令使所有并行线程完成后再处理wait以下的指令。  


## 关键字: (())
在刚开始学习inux shell脚本编程时候，对于它的 四则运算以及逻辑运算。估计很多朋友都感觉比较难以接受。特变逻辑运算符”[]”使用时候，必须保证运算符与算数 之间有空格。 四则运算也只能借助：let,expr等命令完成。 今天讲的双括号”(())”结构语句，就是对shell中算数及赋值运算的扩展。  
1、在双括号结构中，所有表达式可以像c语言一样，如：a++,b--等。  
2、在双括号结构中，所有变量可以不加入：“$”符号前缀。  
3、双括号可以进行逻辑运算，四则运算  
4、双括号结构 扩展了for，while,if条件测试运算  
5、支持多个表达式运算，各个表达式之间用“，”分开  


## 关键字：flock
但这仍有一个问题，就使各个线程使用独立的内存空间，并且不能共享内存。  
在C语言里，可以在同一个进程中开启多个线程，并通过全局变量共享一些内存，在这里做不到。  
如果要使各个线程更新同一个变量，需要使用特殊的技巧，做出全局变量的效果。
- 方法1：使用环境变量  
- 方法2：使用外部文件  
先创建一个临时文件tmp.txt，将初始全局变量写入。  
在每个线程里，当需要更新变量的时候，建立一个tmp.txt.lock文件锁，以实现原子操作。  
```
GLOGAL_VAR_FILE="tmp.txt"
echo 0 > "$GLOBAL_VAR_FILE"
increase_global_var(){
  (
    flock -x 200
    global_var=$(cat "$GLOBAL_VAR_FILE")
    global_var=$((global_var + 1))
    echo "$global_var" > "$GLOBAL_VAR_FILE"
  ) 200>"$GLOBAL_VAR_FILE.lock"
}
for i in $(seq 1 10)
do
  (
    echo "output: $i"
    increase_global_var
  ) &
done
wait
final_value=${cat "$GLOBAL_VAR_FILE"}
echo ${final_value}
rm "$GLOBAL_VAR_FILE"
rm "$GLOBAL_VAR_FILE.lock"
```
flock -x参数的意思是产生独占(exclusive)锁。如果不加参数，默认使用独占锁。  
200参数的意思是添加数字描述符(file descriptor number)。当把该描述符写入锁文件中的时候会释放(关闭)锁。  
除此之外，另一个释放锁的办法是使用如下指令：  
> flock -u 200


## 关键字：${}
此关键字内带一个变量名，表示变量的值。在不引起歧义的情况下可以省略大括号。  
> ${a}  

# Reference
https://www.cnblogs.com/chengmo/archive/2010/10/19/1855577.html  


