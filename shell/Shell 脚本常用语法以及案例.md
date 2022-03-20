
### 0 、关于shell 
	shell是一个程序，它从键盘读取命令然后交由操作系统来执行。几乎所有 Linux 发行版都提供了一个名为 GNU Project 的 shell 程序。
	“bash”是“Bourne Again ，SHell”首字母缩写，bash 是 sh 的增强版本，sh 史蒂夫·伯恩（Steve Bourne）编写的原始 Unix shell 程序。
	用户界面和命令行就是这个另外开发的程序，就是这层“代理”。
	在Linux下，这个命令行程序叫做 Shell。
	

### 1、Shell概述
Shell是一个命令行解释器，它接收应用程序/用户命令，然后调用操作系统内核。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522224919581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)

Shell还是一个功能相当强大的编程语言，易编写、易调试、灵活性强。

### 2、Shell解析器
#### 2.1、查看Linux提供的Shell解析器

```bash
cat /etc/shells
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522225139574.png)
一般常用的是前两个  sh bash 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522234427752.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522234452649.png)

#### 2.2、sh  和 bash的关系
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522234613689.png)
也是就说 执行sh命令的时候  最终执行的还是bash 命令

#### 2.3、查看默认的解析器是

```bash
echo $SHELL
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522234811502.png)
**centos7默认的解析器是/bin/bash**

### 3、Shell脚本入门
#### 3.1、脚本以#!/bin/bash开头（指定解析器）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235017496.png)
#### 3.2、脚本的常用执行方式
##### 3.2.1、 第一种：采用bash或sh+脚本的相对路径或绝对路径（不用赋予脚本+x权限）
- sh+脚本的相对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235331311.png)
- sh+脚本的绝对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235359772.png)

- bash+脚本的相对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235416646.png)

- bash+脚本的绝对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021052223543613.png)

##### 3.2.1、第二种：采用输入脚本的绝对路径或相对路径执行脚本（必须具有可执行权限+x）
- 首先要赋予hello.sh 脚本的+x权限
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235609273.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235620141.png)


- 执行脚本
	1. 相对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235711475.png)
	2. 绝对路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210522235738705.png)

#### 3.3、小案例
**需求**：在/test目录下创建一个test.txt,在test.txt文件中增加“I love you”。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000216768.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000344785.png)

代码：

```shell
#!/bin/bash
cd /test
touch test.txt
echo "I love you " >> test.txt
```

### 4、Shell中的变量
#### 4.1、系统变量
**常用系统变量**
- $HOME ：HOME目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000614182.png)

- $PWD ：当前目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000635976.png)
- $SHELL：默认的解析器
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000712533.png)

- $USER：当前用户
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000738259.png)
- 显示当前Shell中所有变量 set
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523000935281.png)

#### 4.2、自定义变量
##### 4.2.1、基本语法
1. 定义变量：变量=值 
2. 撤销变量：unset 变量
3. 声明静态变量：readonly变量，注意：不能unset

##### 4.2.2、变量定义规则
1. 变量名称可以由字母、数字和下划线组成，但是不能以数字开头，环境变量名建议大写。
2. 等号两侧不能有空格
3. 在bash中，变量默认类型都是字符串类型，无法直接进行数值运算。
4. 变量的值如果有空格，需要使用双引号或单引号括起来。

##### 4.2.3、简单案列
1. 定义变量A
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523001453312.png)

2. 给变量A重新赋值
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523001517780.png)

3. 撤销变量A
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523001602862.png)

4. 声明静态的变量B=2，不能unset
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523001800948.png)

5. 在bash中，变量默认类型都是字符串类型，无法直接进行数值运算
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523002259376.png)

6. 变量的值如果有空格，需要使用双引号或单引号括起来
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523002348916.png)

7. 可把变量提升为全局环境变量，可供其他Shell程序使用
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523002524215.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523002559378.png)
**输出刚才定义的D变量，发现刚才定义的变量并没有数据输出。原因是需要将变量提升为全局变量，然后才能被其他的程序使用。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523002752786.png)

#### 4.3、特殊变量：$n
##### 4.3.1、基本语法
		$n（功能描述：n为数字，$0代表该脚本名称，$1-$9代表第一到第九个参数，十以上的参数，十以上的参数需要用大括号包含，如${10}）
##### 4.3.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021052300325610.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523003241714.png)


#### 4.4、特殊变量：$#
##### 4.4.1、基本语法
		$#	（功能描述：获取所有输入参数个数，常用于循环）。
##### 4.4.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523003448577.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523003435687.png)


#### 4.5、特殊变量：$*、$@
##### 4.5.1、基本语法
	$*	（功能描述：这个变量代表命令行中所有的参数，$*把所有的参数看成一个整体）
	$@	（功能描述：这个变量也代表命令行中所有的参数，不过$@把每个参数区分对待）

##### 4.5.2、简单案例 
	打印输入的所有参数
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523003714414.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523003737582.png)

#### 4.6、特殊变量：$？
##### 4.6.1、基本语法
	$？	（功能描述：最后一次执行的命令的返回状态。如果这个变量的值为0，证明上一个命令正确执行；如果这个变量的值为非0
	（具体是哪个数，由命令自己来	决定），则证明上一个命令执行不正确了。）

##### 4.6.2、简单案例 
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021052300385261.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)

### 5、运算符		
#### 5.1、基本语法
1. “$((运算式))”或“$[运算式]”
2. expr  + , - , \*,  /,  %    加，减，乘，除，取余
注意：expr运算符间要有空格

#### 5.2、简单案例 
##### 5.2.1、计算3+2的值
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523004420883.png)

##### 5.2.2、计算3-2的值
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021052300443991.png)

##### 5.2.3、计算（2+3）X4的值   expr一步完成计算**加粗样式**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523004639568.png)
##### 5.2.4、计算（2+3）X4的值  采用$[运算式]方式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523004750389.png)

### 6、条件判断
#### 6.1、基本语法
[ condition ]（注意condition前后要有空格）
注意：条件非空即为true，[ test ]返回true，[] 返回false。
#### 6.2、 常用判断条件
##### 6.2.1、两个整数之间比较
= 字符串比较
-lt 小于（less than）			-le 小于等于（less equal）
-eq 等于（equal）				-gt 大于（greater than）
-ge 大于等于（greater equal）	-ne 不等于（Not equal）
##### 6.2.2、按照文件权限进行判断
-r 有读的权限（read）			-w 有写的权限（write）
-x 有执行的权限（execute）
##### 6.2.3、按照文件类型进行判断
-f 文件存在并且是一个常规的文件（file）
-e 文件存在（existence）		-d 文件存在并是一个目录（directory）

#### 6.3、简单案例 
##### 6.3.1、23是否大于等于22
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523005508858.png)

##### 6.3.2、hello.sh是否具有写权限
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523005616804.png)

##### 6.3.3、/test/cls.txt目录中的文件是否存在![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523005644345.png)

##### 6.3.4、多条件判断（&& 表示前一条命令执行成功时，才执行后一条命令，|| 表示上一条命令执行失败后，才执行下一条命令）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523005916863.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523010045501.png)

### 7、流程控制		
#### 7.1、if判断
##### 7.1.1、基本语法
	if [ 条件判断式 ];then 
	  程序 
	fi 
	或者 
	if [ 条件判断式 ] 
	  then 
	    程序 
	fi
		注意事项：
	（1）[ 条件判断式 ]，中括号和条件判断式之间必须有空格
	（2）if后要有空格

##### 7.1.2、简单案例

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523011717664.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523011746640.png)
或者
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523011909537.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523011940105.png)

#### 7.2、 case 语句
##### 7.2.1、基本语法
	case $变量名 in 
	  "值1"） 
	    如果变量的值等于值1，则执行程序1 
	    ;; 
	  "值2"） 
	    如果变量的值等于值2，则执行程序2 
	    ;; 
	  …省略其他分支… 
	  *） 
	    如果变量的值都不是以上的值，则执行此程序 
	    ;; 
	esac
	注意事项：
	1)	case行尾必须为单词“in”，每一个模式匹配必须以右括号“）”结束。
	2)	双分号“;;”表示命令序列结束，相当于java中的break。
	3)	最后的“*）”表示默认模式，相当于java中的default。

##### 7.2.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523012342566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523012412585.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)


#### 7.3、for 循环
##### 7.3.1、基本语法
	for (( 初始值;循环控制条件;变量变化 )) 
	  do 
	    程序 
	  done
	或者
	for 变量 in 值1 值2 值3… 
	  do 
	    程序 
	  done


##### 7.3.2、简单案例  从1加到100
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523012948300.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523013001754.png)
或者
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021052301342435.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523013439628.png)


#### 7.4、while 循环
##### 7.4.1、基本语法
	while [ 条件判断式 ] 
	  do 
	    程序
	  done

##### 7.4.2、简单案例 从1加到100
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523013839246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523013846720.png)



### 8、read读取控制台输入	
#### 8.1、基本语法
	read(选项)(参数)
		选项：
	-p：指定读取值时的提示符；
	-t：指定读取值时等待的时间（秒）。
	参数
		变量：指定读取值的变量名

#### 8.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523014219279.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523014226700.png)


### 9、 函数		
#### 9.1、系统函数
##### 9.1.1、基本语法
**basename**

	basename [string / pathname] [suffix]  	（功能描述：basename命令会删掉所有的前缀包括最后一个（‘/’）字符，然后将字符串显示出来。
	选项：
	suffix为后缀，如果suffix被指定了，basename会将pathname或string中的suffix去掉。
	
**dirname**

	dirname 文件绝对路径		（功能描述：从给定的包含绝对路径的文件名中去除文件名（非目录的部分），然后返回剩下的路径（目录的部分））

##### 9.1.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523014619270.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523014633690.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523014706583.png)


#### 9.2、自定义函数
##### 9.2.1、基本语法
	[ function ] funname[()]
	{
		Action;
			[return int;]
	}
	funname
	（1）必须在调用函数地方之前，先声明函数，shell脚本是逐行运行。不会像其它语言一样先编译。
	（2）函数返回值，只能通过$?系统变量获得，可以显示加：return返回，如果不加，将以最后一条命令运行结果，作为返回值。
	return后跟数值n(0-255)

##### 9.2.2、简单案例
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523015122760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ByZWZlY3Rfc3RhcnQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523015142153.png)


### 10、Shell工具（重点）
#### 10.1、cut
	cut的工作就是“剪”，具体的说就是在文件中负责剪切数据用的。cut 命令从文件的每一行剪切字节、字符和字段并将这些字节、字符和字段输出。
##### 10.1.1、基本语法
	cut [选项参数]  filename   
	说明：默认分隔符是制表符
##### 10.1.2、选项参数说明
选项参数	功能
-f	列号，提取第几列
-d	分隔符，按照指定分隔符分割列
##### 10.1.3、简单案例 切割ifconfig 后打印的IP地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210523015625783.png)


#### 10.2、sed
	sed是一种流编辑器，它一次处理一行内容。处理时，把当前处理的行存储在临时缓冲区中，称为“模式空间”，接着用sed命令处理缓冲区中的内容，处理完成后，把缓冲区的内容送往屏幕。接着处理下一行，这样不断重复，直到文件末尾。文件内容并没有改变，除非你使用重定向存储输出。

##### 10.2.1、基本语法
	sed [选项参数]  ‘command’  filename
##### 10.2.2、选项参数说明
	选项参数	功能
	-e	直接在指令列模式上进行sed的动作编辑。

##### 10.2.3、命令功能描述
	命令	功能描述
	a 	新增，a的后面可以接字串，在下一行出现
	d	删除
	s	查找并替换 
##### 10.2.4、简单案例

#### 10.3、awk
	一个强大的文本分析工具，把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行分析处理。
##### 10.3.1、基本语法
	awk [选项参数] ‘pattern1{action1}  pattern2{action2}...’ filename
	pattern：表示AWK在数据中查找的内容，就是匹配模式
	action：在找到匹配内容时所执行的一系列命令

##### 10.3.2、选项参数说明
	选项参数	功能
	-F	指定输入文件折分隔符
	-v	赋值一个用户定义变量

##### 10.3.3、简单案例

#### 10.4、sort
	sort命令是在Linux里非常有用，它将文件进行排序，并将排序结果标准输出。
##### 10.4.1、基本语法
	sort(选项)(参数)
	选项	说明
	-n	依照数值的大小排序
	-r	以相反的顺序来排序
	-t	设置排序时所用的分隔字符
	-k	指定需要排序的列
	参数：指定待排序的文件列表
##### 10.4.2、简单案例
### 11、Shell面试题
#### 10.1、使用Linux命令查询file1中空行所在的行号

```bash
awk '/^$/{print NR}' sed.txt 
```

#### 10.2、Shell脚本里如何检查一个文件是否存在？如果不存在该如何处理？

```bash
#!/bin/bash

if [ -f file.txt ]; then
   echo "文件存在!"
else
   echo "文件不存在!"
fi

```

#### 10.3、用shell写一个脚本，对文本中无序的一列数字排序，并将数字进行累加输出

```bash
sort -n test.txt|awk '{a+=$0;print $0}END{print "SUM="a}'
```

#### 10.4、请用shell脚本写出查找当前文件夹（/home）下所有的文本文件内容中包含有字符”shen”的文件名称

```bash
rep -r "shen" /home | cut -d ":" -f 1
```

#### 10.4、有文件chengji.txt内容如下:
	张三 40
	李四 50
	王五 60
	使用Linux命令计算第二列的和并输出
```bash
cat chengji.txt | awk -F " " '{sum+=$2} END{print sum}'
```






