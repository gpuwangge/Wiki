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

## 关键字：>
>可以把stdout的输出direct到一个文件里  
2>类似，但是是stderr的输出，也是到一个文件里  
&>可以把stdout和stderr都导到同一个文件里  
以上三个关键字都是新开文件。如果需要append，则可以使用>, 2>>和&>>  
另外，>和2>可以出现在同一个命令中，表示把stdout导入一个文件里，stderr导入另一个文件里  

举例：  
> echo "This is standard output" > output.txt  

> ls existing_file non_existent_file >> output.txt 2>> error.txt  




## 关键字：if
```
if [ condition ]
then
  commands
fi
```

## 关键字：test 和 []
est 是 Shell 内置命令，用来检测某个条件是否成立。test 通常和 if 语句一起使用，并且大部分 if 语句都依赖 test。  
test 命令允许你做各种测试并在测试成功或失败时返回它的退出状态码（为0表示为真，为1表示为假）  
[]等同于test  
[]和test内部能用的比较运算符只有==和!=，两者都是用于字符串比较的，不可用于整数比较。  
整数比较只能使用-eq，-gt这种形式。  
无论是字符串比较还是整数比较都不支持大于号小于号。  
shell中的比较运算符：  
-eq       等于  
-ne       不等于  
-gt       大于 （greater）  
-lt       小于 （less）  
-ge       大于等于  
-le       小于等于  


## 关键字：[[]]
[]关键字虽然可以判断条件，但不能使用表达式。  
&&、||、<和> 操作符能够正常存在于[[]]条件判断结构中  
比如如下语法错误：  
```
if [ $a != 1 && $a != 2 ] #错误用法
```
这时候可以使用[[]]关键字。    
如下语法正确：  
```
if [[ $a != 1 && $a != 2 ]]
```
并且，上述等效于下面的表达式：  
```
if [ $a -ne 1 ] && [ $a != 2 ]
```

## 关键字：${}
此关键字内带一个变量名，表示变量的值。在不引起歧义的情况下可以省略大括号。  
> ${a} 

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

## 关键字: (())
在刚开始学习inux shell脚本编程时候，对于它的 四则运算以及逻辑运算。  
估计很多朋友都感觉比较难以接受。特变逻辑运算符”[]”使用时候，必须保证运算符与算数 之间有空格。   
四则运算也只能借助：let,expr等命令完成。 今天讲的双括号”(())”结构语句，就是对shell中算数及赋值运算的扩展。  
1、在双括号结构中，所有表达式可以像c语言一样，如：a++,b--等。  
2、在双括号结构中，所有变量可以不加入：“$”符号前缀。  
3、双括号可以进行逻辑运算，四则运算  
4、双括号结构 扩展了for，while,if条件测试运算  
5、支持多个表达式运算，各个表达式之间用“，”分开 

## 关键字: wait
使用()&产生的并行命令组分别运行在不同的线程上。  
wait指令使所有并行线程完成后再处理wait以下的指令。   

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

## 关键字: tee
tee命令作用可以用字母T来形象地表示。它把输出的一个副本输送到标准输出，另一个副本拷贝到相应的文件中。  
如果希望在看到输出的同时，也将其存入一个文件，那么这个命令再合适不过了。  
它的一般形式为：   
> tee files  

-a参数的意思是添加到文件末尾。  
```
filename="file.txt"
echo "hello world" | tee -a "$filename" 
```

## 关键字: date
用于显示时间信息  
```
dt=$(date '+%m/%d/%Y %H:%M:%S')
echo "$dt"
```
输出结果为：  
10/01/2024 09:25:18  

如果要计算经过了多少秒，可以使用如下指令  
```
start_time=$(date +%s)
end_time=$(date +%s)
duration=$((end_time-start_time))
echo "Duration in second: $duration"
```

## 关键字：read
读取文件信息    
比如读取inputfile.txt并加入到数组中：  
```
test_array=()
while IFS= read -r line; do
  test_array+=(${line}
done < inputfile.txt
total_count=${#test_array[@]}
```
@表示全部元素  

## 关键字：$?
用于显示最后命令的退出状态，一般用0表示没有错误，1或其他值表示有错误

## 关键字：cmp
比较两个文件，如果相同，返回0。  
如果不同，返回1。  
如果文件不存在，返回2。  
```
file1="file1.txt"
file2="file2.txt"
cmp -s "$file1" "file2"
r=$?
echo ${r}
```
-s的意思是silent, 就是只返回状态，不返回其他信息。  
如果不加参数，那么除了状态外，会打印出两个文件在哪里不一样。  
因为cmp是逐字节比对，因此会打印出哪一个文件的那个一个字节不同。  
比如file1的内容是abc, file2的内容是abc1，则输出如下：  
```
file1.txt file2.txt differ: byte 4, line 1
```
表示从文件开头算起，第四个字节不同，该处位于文件第一行。  
如果文件有不止一行，byte的值也是从文件最开头算起的。  
注意windows下换行是两个字节，unix是一个字节。  

## 关键字：diff
cmp只能发现哪一个字节不同。如果想生成一份全面的对比结果，可以使用diff命令。  
diff能逐行比较两个文件，并输出结果  
diff的使用方法跟cmp类似。区别是生成的信息中包含如下信息：
```
1,3c1,3
```
这份信息表示两个文件的第1至第3行不同。  
```
2a3
```
这份信息表示第一个文件的第二行后面缺失了第二个文件的第三行。  




# Reference
https://www.cnblogs.com/chengmo/archive/2010/10/19/1855577.html  


