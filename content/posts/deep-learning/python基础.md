---
title: "Python基础"
subtitle: ""
date: 2023-04-23T20:00:05+08:00
lastmod: 2023-04-23T20:00:05+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["python"]
tags: ["python"]
featuredImage: ""
---

研一到现在一直没有一个研究方向，被老板push了。。。花了三天把python学了，会Java学python确实是轻松。以前被编程语言给限制了，其实有基础的话，学门语言的成本是很低的。学python的时候在网上复制了份笔记，稍微改了改，放博客上了么好了。

## python基础语法

#### 注释

```python
# 单行注释

"""
多行注释
"""
```

#### 变量有没有类型?

没有，字符串变量表示变量存储了字符串而不是表示变量就是字符串。

#### 类型转换

int(x)  # x转整数

float(x)  # x转浮点数

str(x) # x转字符串

#### 标识符（python大小写敏感）

是用户在编程的时候所使用的一系列名字，用于给变量、类、方法等命名。

关键字不能做标识符：

False、True、None、and、as、assert、break、class、continue、def、del、elif、else、except、、finally、for、from、global、if、import、in、is、lambda、nonlocal、not、or、pass、raise、return、try、while、with、yield

#### 运算符

- /  ：除
- // ：整除
- % ：取余
- ** ：指数



#### 字符串

- 定义
  - 单引号、双引号、三引号
  - 引号嵌套：\转义 或 用不同引号

- 字符串格式化1（str1）

精度控制语法是：m.n的形式控制，如%5d、 %5.2f、%.2f，其中m和.n均可省略

- 字符串格式化2（如str2，注意那个f）

这种方式：不理会类型、不做精度控制。适合对精度没有要求的时候快速使用。

```
name = "张三"
age = 25
salary = 300000.0
str1 = "姓名:%s,年龄:%d,工资:%.2f" % (name, age, salary)
str2 = f"姓名:{name},年龄:{age},工资:{salary}"
print(str1 +'\n'+ str2)
```

结果

```
姓名:张三,年龄:25,工资:300000.00
姓名:张三,年龄:25,工资:300000.0
```



#### 数据输入输出

input()，注意获取的是字符串；print()，输出。

```
name = input("name:")
age = input("age:")  # input获取的是字符串
age = int(age)  # 类型转换
salary = 300000.0
aveSalary = salary/12
print(f"姓名:{name},年龄:{age},工资:{salary}")
print("%s的月工资:%.2f" % (name, aveSalary))
```

结果

```
name:李四
age:28
姓名:李四,年龄:28,工资:300000.0
李四的月工资:25000.00
```

#### 条件语句

if elif ... else

```
# 三次机会猜数字例子
import random
num = random.randint(1, 10)

for i in range(0, 3):
    inputNum = input("输入你猜的数字：")
    inputNum = int(inputNum)
    if inputNum == num:
        print("恭喜你，猜对了！")
        exit()
    elif inputNum > num:
        print(f"{inputNum}大了")
    else:
        print(f"{inputNum}小了")
print("很遗憾，没有机会了，答案是：", num)
```

#### 循环语句

**while循环语句，for循环语句。**

两者能完成的功能基本差不多,但仍有一些区别:

while循环的循环条件是自定义的，自行控制循环条件；

for循环是一种”轮询”机制，是对一批内容（严格说是序列类型的内容：字符串、列表、元组等）进行逐个处理，只能被动取出数据处理。

```python
'''
输出乘法表（while的例子）
'''

i = 1
while i <= 9:
    j = 1
    while j <= i:
        # \t:制表符；end=''的作用：输出不换行
        print(f"{j}*{i}={j*i}\t", end='')
        j += 1
    print()  # 输出一个换行
    i += 1
```

```
name = "zhangsan is a boy"
count = 0
for a in name:
    if a == 'a': 
        count += 1
print(f"有{count}个a")
```

结果：有3个a

**continue 和 break**

continue关键字：中断本次循环，直接进入下一次循环

break：结束循环

#### 函数

- 定义

```
def 函数名(传入参数):
    函数体
    return 返回值

# 注意：1. 先定义函数，后调用函数。
       2. 参数不需要，可以省略；
          返回值不需要，可以省略。
```



- None返回值

1. None是类型'NoneType'的字面量，用于表示:空的、无意义的；
2. 函数如何返回None：不使用return语句即返回None；主动return None
3. 使用场景：函数返回值；if判断相当于False；变量定义

- 函数说明文档（多行注释解释函数）

```
def count_a(name):
    """
    :param name:待测字符串
    :return: 字符串中a的个数
    """

    count = 0
    for a in name:
        if a == 'a':
            count += 1
    return count

str = "ZhangSan is a boy"
num = count_a(str)
print(f"有{num}个a")
```

- 变量在函数的作用域

​       使用global关键字，可以在函数体内申明全局变量



函数有4中常见参数使用方式:(参数还可以是一个函数)

\1. 位置参数：

根据参数位置来传递参数

\2. 关键字参数

通过“键=值”形式传递参数，可以不限参数顺序。可以和位置参数混用，位置参数需在前。

\3. 缺省参数

不传递参数值时会使用默认的参数值

默认值的参数必须定义在最后

\4. 不定长参数（可变参数）

位置不定长传递以*****号标记一个形式参数，以**元组**的形式接受参数，一般命名为args。

关键字不定长传递以******号标记一个形式参数，以**字典**的形式接受参数，一般命名为kwargs。

```
"""
函数进阶之传参
"""
def getBeauty(name, residence, age=14):
    print(f"{name}住在大观园的{residence}，今年{age}岁，亭亭玉立。")
# 1. 按照位置一一传参
getBeauty('黛玉', '潇湘馆', 12)
# 2. 关键字参数
getBeauty(name='宝钗', age=13, residence='怡红院')
# 3. 缺省参数（函数定义时，参数有默认值，只能在最后）
getBeauty('探春', '秋爽斋')
# 4. 不定长参数

def test1(*args):
    print(args)

def test2(**kwargs):
    print(kwargs)

test1('黛玉', '潇湘馆', 12)
test2(name='宝钗', age=13, residence='怡红院')
```

结果:

黛玉住在大观园的潇湘馆，今年12岁，亭亭玉立。

宝钗住在大观园的怡红院，今年13岁，亭亭玉立。

探春住在大观园的秋爽斋，今年14岁，亭亭玉立。

('黛玉', '潇湘馆', 12)

{'name': '宝钗', 'age': 13, 'residence': '怡红院'}

- 匿名函数

  ```python
  lambda 参数1, 参数2: 函数体（只能一行代码）
  ```

  应用可参考： https://blog.csdn.net/PY0312/article/details/88956795 

  和java的stream、lambda差不多。

#### 数据容器

**1.概要**

1.什么是数据容器?

一种可以存储多个元素的Python数据类型

\2. Python有哪些数据容器?

list(列表)、tuple(元组)、 str(字符串)、 set(集合)、 dict(字典)。

它们各有特点，但都满足可容纳多个元素的特性。

**2.list(列表)**

1.定义

```
myList = ["贾宝玉", "林黛玉", "薛宝钗", 1, 2, 3]
print(myList)
myList[-1] = "王熙凤"
print(myList[-1])
print(myList[2])
# 遍历
# for name in myList:
#    print("姓名：", name)
```

结果

['贾宝玉', '林黛玉', '薛宝钗', 1, 2, 3]

王熙凤

薛宝钗

**3.tuple(元组)**

1.定义：

```python
a_tuple = ("贾宝玉", "林黛玉", "薛宝钗", 1, 2)

b_tuple = ("hh", ) # 元组中，单个元素的话，后面一定要加个，

print(a_tuple)
```

结果

('贾宝玉', '林黛玉', '薛宝钗', 1, 2)

2.元组的特点：

- 可以容纳多个数据；
- 可以容纳不同类型的数据( 混装)
- 数据是有序存储的(下标索引)
- 允许重复数据存在
- **不可以修改**(增加或删除元素等)
- 支持for循环

多数特性和list一致,不同点在于不可修改的特性。

**4.string(字符串)**

1.作为数据容器，字符串有如下特点:

- 只可以存储字符
- 长度任意(取决于内存大小)
- 支持下标索引
- 允许重复字符串存在
- 不可以修改(增加或删除元素等)
- 支持for循环

例子：strip()和split()演示

```
a = "  \n张三\n李四  \n"
b = a.strip()  # 删除字符串头尾的空格和换行,返回字符串，
# b:'张三\n李四', a还是a
c = a.split()  # 以空格和\n为分隔符，分割字符串,返回list
# c:['张三', '李四']
```

例子：分割字符串

```
my_str = "itheima itcast boxuegu"
numIT = my_str.count("it")  # 数字符串中"it"的数量
new_my_str = my_str.replace(" ", "|")  # 替换空格为|
str_list = new_my_str.split("|")  # 分割字符串

print(f"字符串{my_str}中有{numIT}个it字符")
print(f"字符串{my_str}中空格替换|后的新字符串：{new_my_str}")
print(f"新字符串{new_my_str}中按照|分割后的列表是：{str_list}")

"""
结果：
字符串itheima itcast boxuegu中有2个it字符
字符串itheima itcast boxuegu中空格替换|后的新字符串：itheima|itcast|boxuegu
新字符串itheima|itcast|boxuegu中按照|分割后的列表是：['itheima', 'itcast', 'boxuegu']
"""
```

**5.序列**

1.什么是序列?

内容连续、有序，支持下标索引的一类数据容器，（列表、元组、字符串）

2.序列如何做切片

```
heima = "你好好学习，你天天向上，加油！"
new_heima = heima[12:2:-1] # '加，上向天天你，习学'
```

序列[起始:结束:步长]

- 起始可以省略，省略从头开始
- 结束可以省略，省略到尾结束
- 步长可以省略，省略步长为1 (可以为负数，表示倒序执行)

```
"""
序列切片，得到”上向天天“
"""

heima = "你好好学习，你天天向上，加油！"
# 分割出 你天天向上
heima = heima.split("，")[1]  # 得到中间的字符串
# 倒序字符串
new_heima = heima[::-1]  # 倒叙切片：上向天天你
new_heima = new_heima.replace("你", "")  # 替换你为空：上向天天
print(new_heima)  # 输出 上向天天
```

**6.集合（set）**

1.集合有如下特点:

- 可以容纳多个数据
- 可以容纳不同类型的数据(混装)
- 数据是无序存储的(不支持下标索引)
- 不允许重复数据存在
- 可以修改(增加或删除元素等)
- 支持for循环，不支持while

2.定义

```
"""
集合练习：信息去重
"""

my_list = ["贾宝玉", "林黛玉", "王熙凤", "贾宝玉", "林黛玉", "王熙凤", "XiFeng", "DaiYu", "XiFeng", "DaiYu"]
my_set = set()
# my_set = {}  # 定义的是字典，不是集合
for beauty in my_list:
    my_set.add(beauty)
print(f"列表{my_list}存入集合后的结果{my_set}")

"""
结果
列表['贾宝玉', '林黛玉', '王熙凤', '贾宝玉', '林黛玉', '王熙凤', 'XiFeng', 'DaiYu', 'XiFeng', 'DaiYu']存入集合后的结果{'王熙凤', 'DaiYu', '贾宝玉', 'XiFeng', '林黛玉'}
"""
```

**7.字典（dict）**

- 相当于Java中的Map
- 格式：{key: value, key:value}
- key和value可以是任意类型，value不能是字典
- key不能重复，因为dict中的key使用集合实现的

例子

```
"""
字典例子：涨工资，升职
"""
daGuanYuan = {'贾宝玉': {'住处': '怡红院', '月钱': 8000, '级别': 5},
              '林黛玉': {'住处': '潇湘馆', '月钱': 6000, '级别': 5},
              '薛宝钗': {'住处': '蘅芜院', '月钱': 5000, '级别': 4},
              '袭人': {'住处': '怡红院', '月钱': 900, '级别': 2},
              '晴雯': {'住处': '怡红院', '月钱': 900, '级别': 2}}
for beauty in daGuanYuan:
    # 找出大丫头们，给她们涨工资，升级
    if daGuanYuan[beauty]['级别'] == 2:
        daGuanYuan[beauty]['级别'] += 1
        daGuanYuan[beauty]['月钱'] += 1000

print(daGuanYuan)
```

**8.总结**

基于各类数据容器的特点，它们的应用场景如下:

- 列表: 一批数据，可修改、可重复、有序的存储场景
- 元组: 一批数据，不可修改、可重复、有序的存储场景
- 字符串:一串字符串的存储场景
- 集合:一批数据，去重、无序存储场景
- 字典: 一批数据,可用Key检索Value的存储场景



#### python文件操作

**1.文件的编码**

是一种规则集合，记录了内容与二进制间互换的逻辑，常用的时UTF-8.

**2.文件操作**

- **文件读取**

![img](https://i0.hdslb.com/bfs/note/e7c86a6b2050c08a453bb37a0314621ea703fcd8.png@680w_!web-note.webp)



```python
"""
open(file,mode,encoding)例子：数文件中某字符串个数
"""

# coding=utf-8

f = open("红楼梦.txt", 'r', encoding='UTF-8')
"""
for line in f:
	# line就是文件的每一行，可以把文件理解为列表
"""

# hongLou = f.read()  # 读取文件全部信息,返回字符串 
					# f.read(10) 读取10字节数据，注意光标也会移动

lines = f.readlines()  # 获取文件全部内容，返回list，每行是list的一个元素
count2 = 0
for line in lines:
    # print(line.strip())  # strip删除字符串中头尾的空格和换行
    names = line.split()  # 以空格和\n为分隔符，分割字符串,返回list
    for name in names:
        if name == "怡红院":
            count2 += 1

print(f"方法二：文件《红楼梦.txt》中有{count2}个怡红院")

f.close()  # 对应open
```

- **文件写入和追加**

​	1.写入文件使用open函数的"w"模式；

​	追加写入文件使用open函数的"a"模式。

​	既要读又要写："w+"、"a+"。

​	2.写入和追加的方法一样，有:

- write(), 写入内容
- flush(), 刷新内容到硬盘中

​	3.注意事项:

- w/a模式，文件不存在，会创建新文件；
- w模式，文件存在，会清空原有内容；a模式则会在原有内容后追加写入；
- close()方法,带有flush()方法的功能

```
f1 = open("红楼梦1.txt", "w", encoding="UTF-8")  # utf8防乱码
f1.write("石头记")
f1.close()

f1 = open("红楼梦1.txt", "a", encoding="UTF-8")
f1.write("\n木石前盟")
f1.close()
```

文件：红楼梦1.txt

```
石头记
木石前盟
```

备份文件的例子

```python
fileName = "金陵十二钗.txt"
# 备份文件名
new_fileName = "金陵十二钗.txt.bak"
# 读取文件 
#with语法，可以自动close文件
#可参考 https://www.liaoxuefeng.com/wiki/1016959663602400/1115615597164000
with open(fileName, 'r', encoding='utf-8') as f:
    jinChai = f.readlines()
out_file = open(new_fileName, 'w', encoding='utf-8')
# 写入备份文件
for line in jinChai:
    # 又副册的数据丢弃
    if line.split()[1] == "又副册":
        break
    out_file.write(line)

out_file.close()
```



#### 异常（bug）

**1.语法

```python
try:
    可能有异常的代码
except [异常 as 别名]:
    出现异常执行的代码
else:
    没出现异常执行
finally:
    总会执行
```



```
"""
异常捕获
"""

file_name = "红楼梦1.txt"
# 打开一个不存在的文件
# 1. 出现异常，执行except中的语句,捕获所有异常
try:
    f = open(file_name, 'r', encoding='utf-8')
except Exception as e:
    # 如果出现异常，则执行以下语句，而不是停止程序
    print(e)
    f = open(file_name, 'w', encoding='utf-8')

# 2. 捕获指定异常
try:
    1/0
except ZeroDivisionError as e  	
    print("除数不能是0")
    print(e)


# 3. 捕获指定多个异常
# else和finally可选
try:
    print(1/1)
except (ZeroDivisionError, NameError) as e:
    print("除数不能是0")
    print(e)
else:
    print("没有异常执行else语句内容")
finally:
    print("有没有异常都执行finally的内容")
    f.close()
```

异常捕获2,---异常的传递性

```
"""
异常捕获2,---异常的传递性
"""

def readTxt(file_name):
    f = open(file_name, 'r', encoding='utf-8')
    for name in f:
        print(name)

file_name = "红楼梦1.txt"
# 打开一个不存在的文件
try:
    readTxt(file_name)
except Exception as e:
    # 如果出现异常，则执行以下语句，而不是停止程序
    print(e)
```

#### python模块

1.什么是模块?

模块就是一个Python代码文件，内含类、函数、变量等，我们可以导入进行使用。

2.如何导入模块

[from 模块名] import [模块|类|变量| 函数|*] [as 别名]

3.注意事项:

- from可以省略，直接import即可
- as别名可以省略
- 通过”.”来确定层级关系
- 模块的导入一般写在代码文件的开头位置

4.自定义模块

![img](https://i0.hdslb.com/bfs/note/3d063a58c54142afdb84eb5fab56c40a43a6ee5a.jpg@680w_!web-note.webp)

```
__all__变量可以控制import * 时哪些功能可以被导入，是个列表 只能限制*，按函数名还是能导入的
用法：__all__ = ['fun1','fun2']

不同模块，同名的功能，被导入时，后导入的会覆盖先导入的
```

例子：__main__

```
"""
自定义模块 day4_bug.py
"""

def readTxt(file_name):
    f = open(file_name, 'r', encoding='utf-8')
    for name in f:
        print(name, end='')

# 没有main，在调用时执行了
file_name = "红楼梦.txt"
readTxt(file_name)


# __main__保证在该模块被引用的时候，不执行main中的部分。
# pycharm快捷键：输入 main 可直接跳出 if __name__ == '__main__':
if __name__ == '__main__':
    file_name = "红楼梦1.txt"
    readTxt(file_name)
```

引用该模块

```
"""
自定义模块的调用
"""

from day4_bug import readTxt
readTxt("../day03/红楼梦.txt")
```

运行结果：调用了文件中没有被__main__包围的内容，所以输出了红楼梦.txt的内容

```
红楼梦.txt文件的内容如下：
王熙凤
贾宝玉
贾琏
贾政
贾敏
../day03/红楼梦.txt文件的内容如下：
贾宝玉 住处 怡红院 月钱 8000
林黛玉 住处 潇湘馆 月钱 6000
薛宝钗 住处 蘅芜院 月钱 5000
袭人 住处 怡红院 月钱 900
晴雯 住处 怡红院 月钱 900
```

#### python包

- 包就是一个文件夹，可以存放多个python模块，在逻辑上将一批模块归为一类，方便使用。
- `__init__.py`文件用于表示该文件夹为python包，而非普通文件夹（高版本python也可以不用这个文件了）。导入包时，该文件里的内容会被先执行。



#### json

json是用于网络传输的字符串，格式和python中的字典一模一样。

- json.dumps(data)：可以将字典转化为json字符串
- json.loads(data)：将json字符串转为python字典或列表（列表中的元素都是字典）
- 含有中文是，函数中可以加上：ensure_ascii = False



#### 面向对象

**1.类的定义**

```python
class 类名:
    成员变量
    
   	def 方法名(self):
        pass
```



**2.面向对象编程**

面向对象编程就是，使用对象进行编程。

即，设计类，基于类创建对象，并使用对象来完成具体的工作。

**3.一些内置方法**

- ``__init__``：构造函数
- ``__str__``：输出时的字符串
- ``__lt__``：比较大小
- ``__le__``：比较大小
- ``__eq__``：比较大小



**4.私有成员、封装**

- 以” __ “开头的变量就是私有变量

**5.继承--**单继承、多继承

```python
class 子类(父类[, 父类2, 父类3]):
    pass
```

继承多个类，父类中有同名属性或方法时，先继承的优先级高（从左到右）

- **复写父类成员方法、属性**



**6.类型注解**

类型注解只是提示性的，数据类型和注解类型无法对应也不会出错

```python
语法1：
data: list[int] = [1,2,3]

语法2：
data # type: int

方法注解
def 方法名(data1: int, data2: float) -> list    #返回值类型 ->

Union联合类型注解
from typing import Union
lis: list[Union[str,int]] = [1, "fff",3, "ddd"]
```



**7.多态**

。。。

