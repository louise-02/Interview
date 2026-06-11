# Shell 基础

## 创建脚本

```sh
# 创建shell
touch test.sh

// #! 是一个约定的标记 它告诉系统这个脚本需要什么解释器来执行
#!/bin/bash
echo "Hello World !"

# 使得脚本具有可执行权限
chmod +x ./test.sh
# 执行脚本
./test.sh
```

## 变量

```sh
# 使用变量
your_name="louise"
echo $your_name
echo ${your_name}

# 只读变量
myUrl="https://www.google.com"
readonly myUrl

# 删除变量
unset variable_name

# 命令替换 使用$()或`
echo $(docker ps -a)
echo `docker ps -a`
```

## 字符串

```sh
# 字符串拼接
your_name="louise"
greeting="hello, "$your_name" !"

# 获取字符串长度
string="abcd"
echo ${#string}   # 输出 4

# 提取子字符串 第一个下标是0
string="runoob is a great site"
echo ${string:1:4} # 输出 unoo

# 查找子字符串  i o哪个字母先出现就计算哪个
string="runoob is a great site"
echo `expr index "$string" io`  # 输出 4
```

## 数组

Shell 数组用括号来表示，元素用"空格"符号分割开。

```sh
# 定义数组
数组名=(值1 值2 ... 值n)
array_name=(value0 value1 value2 value3)

# 读取数组
${数组名[下标]}
echo ${array_name[@]}  -- @获取数组中的所有元素

# 获取数组的长度
length=${#array_name[@]}
length=${#array_name[*]}

# 遍历数组
a=("Fdf" "df" "fd")
for str in ${a[@]};do
echo $str
done
```

## 传递参数

```sh
./test.sh  1 2 3

# 获取传递参数个数
$#

# 获取第n个参数
$n

# 以"$1 $2 … $n"的形式输出所有参数
$*

# 以"$1" "$2" … "$n" 的形式输出所有参数
$@

# 脚本运行的当前进程ID号
$$

# 脚本自己本身名字
$0

# 遍历参数
for arg in "$@"
do
    echo "Argument: $arg"
done
```

## 运算符

```sh
# expr 是一款表达式计算工具，使用它能完成表达式的求值操作
# 算数运算符
+ - * / %    `expr $a + $b`
= 赋值  a=$b
== !=  [ $a == $b ]  前后必须要有空格
val=`expr 2 + 2`

# 关系运算符  只支持数字，不支持字符串，除非字符串的值是数字
-eq -ne -gt -lt -ge -le  [ $a -eq $b ]

# 布尔运算符
!  非运算  [ ! false ]
-o 或运算  [ $a -lt 20 -o $b -gt 100 ]
-a 与运算  [ $a -lt 20 -a $b -gt 100 ]

# 逻辑运算符
&& ||   [[ $a -lt 100 && $b -gt 100 ]] 

# 字符串运算符
= != 
-z 检测字符串长度是否为0，为0返回 true         [ -z $a ] 
-n 检测字符串长度是否不为 0，不为 0 返回 true  [ -n "$a" ] 
$  检测字符串是否为空，不为空返回 true         [ $a ]

# 文件测试运算符
-b file	检测文件是否是块设备文件，如果是，则返回 true。	[ -b $file ] 返回 false。
-c file	检测文件是否是字符设备文件，如果是，则返回 true。	[ -c $file ] 返回 false。
-d file	检测文件是否是目录，如果是，则返回 true。	[ -d $file ] 返回 false。
-f file	检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。	[ -f $file ] 返回 true。
-g file	检测文件是否设置了 SGID 位，如果是，则返回 true。	[ -g $file ] 返回 false。
-k file	检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。	[ -k $file ] 返回 false。
-p file	检测文件是否是有名管道，如果是，则返回 true。	[ -p $file ] 返回 false。
-u file	检测文件是否设置了 SUID 位，如果是，则返回 true。	[ -u $file ] 返回 false。
-r file	检测文件是否可读，如果是，则返回 true。	[ -r $file ] 返回 true。
-w file	检测文件是否可写，如果是，则返回 true。	[ -w $file ] 返回 true。
-x file	检测文件是否可执行，如果是，则返回 true。	[ -x $file ] 返回 true。
-s file	检测文件是否为空（文件大小是否大于0），不为空返回 true。	[ -s $file ] 返回 true。
-e file	检测文件（包括目录）是否存在，如果是，则返回 true。	[ -e $file ] 返回 true。
```

## echo

```sh
# 显示字符串
echo "It is a test"
echo It is a test

# 显示转义字符
echo "\"It is a test\""  "It is a test"

# 显示变量
read name  --read 命令从标准输入中读取一行,并把输入行的每个字段的值指定给 shell 变量
echo "$name It is a test"

# 显示换行
echo -e "OK! \n"   -e 开启转义

# 显示不换行
echo -e "OK! \c"

# 显示结果指定到文件
echo "It is a test" > myfile

# 显示命令执行结果 反引号
echo `date`
```

## test 判断

```sh
#!/bin/bash

num1=100
num2=200

# 方式 1：使用 test 关键字
if test $num1 -eq $num2
then
    echo "两个数字相等"
else
    echo "两个数字不相等"
fi

# 方式 2：使用 [ ] 符号（这是最推荐的，也是最常见的）
if [ $num1 -lt $num2 ]; then
    echo "num1 小于 num2"
fi

# 方式 3：结合文件判断
if test -e "/etc/passwd"; then
    echo "系统配置文件存在"
fi
```

## if

```sh
-- if else
if condition
then
    command1 
    ...
else
    command
fi

-- if else-if else
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi

#!/bin/bash
num1=$[2*3]
num2=$[1+5]
if test $[num1] -eq $[num2]
then
    echo '两个数字相等!'
else
    echo '两个数字不相等!'
fi
```

## for

```sh
-- for 循环
for var in item1 item2 ... itemN
do
    command1
    ...
done

#!/bin/bash
for loop in 1 2 3 4 5
do
    echo "The value is: $loop"
done
```

## while

```sh
while condition
do
    command
done

#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done

-- 无限循环
while true
do
    command
done

for (( ; ; ))
```

## case

```sh
# 语法
case 值 in
模式1)
    command1
    commandN
    ;;
模式2)
    command1
    commandN
    ;;
esac

#!/bin/bash
echo '输入 1 到 4 之间的数字:'
echo '你输入的数字为:'
read aNum
case $aNum in
    1)  echo '你选择了 1'
    ;;
    2)  echo '你选择了 2'
    ;;
    3)  echo '你选择了 3'
    ;;
    4)  echo '你选择了 4'
    ;;
    *)  echo '你没有输入 1 到 4 之间的数字'
    ;;
esac
```

## 函数

```sh
# 语法
[ function ] funname [()]

{

    action;

    [return int;]

}

#!/bin/bash
demoFun(){
    echo "这是我的第一个 shell 函数!"
}
echo "-----函数开始执行-----"
demoFun
echo "-----函数执行完毕-----"
```

## 输入输出重定向

```sh
# 将输出重定向到 file
command > file
# 将输入重定向到 file
command < file
# 将输出以追加的方式重定向到 file
command >> file
# 将文件描述符为 n 的文件重定向到 file
n > file
# 将文件描述符为 n 的文件以追加的方式重定向到 file
n >> file
# 将输出文件 m 和 n 合并
n >& m
# 将输入文件 m 和 n 合并
n <& m
# 将开始标记 tag 和结束标记 tag 之间的内容作为输入
<< tag

# 重定向深入讲解
# 标准输入文件(stdin)：stdin的文件描述符为0，Unix程序默认从stdin读取数据。
# 标准输出文件(stdout)：stdout 的文件描述符为1，Unix程序默认向stdout输出数据。
# 标准错误文件(stderr)：stderr的文件描述符为2，Unix程序会向stderr流中写入错误信息。
command >> file 2>&1  

# 如果希望执行某个命令，但又不希望在屏幕上显示输出结果，那么可以将输出重定向到 /dev/null
command > /dev/null 
```

## ${},``,$(),$(())四种语法

${ } 变量、截取、替换

```sh
path=/etc/sysconfig/network

# 截取
# #是去掉左边（键盘上#在 $ 的左边）
# %是去掉右边（键盘上% 在$ 的右边）
# 单一符号#，%是最小匹配；两个符号##，%%是最大匹配
echo ${path#*/}     # etc/sysconfig/network
echo ${path#tc}     # /etc/sysconfig/network
echo ${path##*/}    # network
echo ${path%/*}     # /etc/sysconfig
echo ${path%%/*}    # 空
echo ${path:3:7}    # c/sysco

# 字符替换
# 第一个出现的et替换成op
echo ${path/et/op}
# 所有出现的et替换成op
echo ${path//et/op}
```

｀｀与$() 命令替换

```sh
# 直接当成字符串输出
echo date
# 相当于函数调用，先执行date命令 Tue Sep 3 16:10:43 CST 2019
echo `date`
```

$(())与expr 整数运算

```sh
echo $((3+2))         # 5
echo $((3*2))         # 6
echo `expr 2+3`       # 运算符前后需要空格  -- 2+3
echo `expr 2 + 3`     # 5
echo `expr 2 * 3`     # 会提示语法错误，需要转义
echo `expr 2 \* 3`    # 6
```

## date

```sh
echo $(date +'%Y-%m-%d %H:%M:%S')
```

## 文件取并集 交集 差集

```sh
# 并集
cat $file1 $file2 | sort | uniq
sort a.txt b.txt | uniq

# 交集
sort a.txt b.txt | uniq -d

# 差集
sort a.txt b.txt  | uniq -u   (去掉重复行)
```

# 常用脚本

## 部署 java 项目

```sh
#!/bin/bash

# kill 进程
ps -ef | grep scqms.jar | grep -v grep  | awk '{print $2;}' | xargs kill -9
# 归档日志
mv /home/scqms/logs/scqms.log /home/scqms/logs/scqms$(date +'%Y-%m-%d_%H-%M-%S').log
# 启动后端项目
nohup java -jar -Xmx20480m -Xms5120m  scqms.jar > ./logs/scqms.log 2>&1 &
```

## 部署前端项目

```sh
#!/bin/bash

# 检查是否传递了压缩包文件名作为参数
if [ $# -ne 1 ]; then
    echo "Usage: $0 <archive_file>"
    exit 1
fi

# 备份
mv html html$(date +'%Y-%m-%d_%H-%M-%S')

# 使用传递的参数作为压缩包文件名
archive_file=$1

# 解压缩指定的压缩包到 html 文件夹
unzip $archive_file -d html
```

## 集群分发脚本

```sh
# 在环境变量下创建脚本
vim xsync

# 编写脚本内容
#!/bin/bash
#1. 判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi
#2. 遍历集群所有机器
for host in hadoop102 hadoop103 hadoop104
do
    echo ====================  $host  ====================
    #3. 遍历所有目录，挨个发送
    for file in $@
    do
        #4. 判断文件是否存在
        if [ -e $file ]
            then
                #5. 获取父目录
                pdir=$(cd -P $(dirname $file); pwd)
 
                #6. 获取当前文件的名称
                fname=$(basename $file)
                ssh $host "mkdir -p $pdir"
                rsync -av $pdir/$fname $host:$pdir
            else
                echo $file does not exists!
        fi
    done
done

# 修改执行权限
chmod +x xsync
# 执行脚本
sudo ./bin/xsync
```

