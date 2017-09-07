# Python常用运维脚本实例

### 1.  用Python写一个列举当前目录以及所有子目录下的文件，并打印出绝对路径

```python
#!/usr/bin/env python
import os
for root, dirs, files in os.walk('/tmp'):
    for name in files:
        print (os.path.join (root, name))
```

os.walk()原型为os.walk(top, topdown = True, onerror = None, followlinks = False)。我们一般只使用第一个参数（topdown指明遍历的顺序），该方法对于每个目录返回一个三元组(dirpath, dirnames,  filenames)，第一个是路径，第二个是路径下面的目录，第三个是路径下面的非目录（对于windows来说也就是文件）。而os.listdir(path)，其参数path表示要获得内容目录的路径。

### 2. 写程序打印三角形

```python
#!/usr/bin/env python
input = int(raw_input('input number:'))
for i in range(input):
    for j in range(i):
        print '*',
    print '\n'
```

### 3. 猜数器。程序随机生成一个个位数字，然后等待用户输入，输入数字和生成数字相同则视为成功。成功则打印三角形。失败则重新输入（提示：随机数函数random）

```python
#!/usr/bin/env python
import random
while True:
    input = int(raw_input('input number:'))
    random_num = random.randint(1, 10)
    print input, random_num
    if input == random_num:
        for i in range(input):
            for j in range(i):
                print '*',
            print '\n'
    else:
        print 'please input number again'
```

### 4. 请按照这样的日期格式（xxxx-xx-xx）每日生成一个文件，例如今天生成的文件为2013-09-23.log，并且把磁盘的使用情况写到到这个文件中

```python
#!/usr/bin/env python
#!coding = utf-8
import time
import os
new_time = time.strftime('%Y-%m-%d')
disk_status = os.popen('df -h').readlines()
str1 = ''.join(disk_status)
f = file(new_time + '.log','w')
f.write('%s' %str1)
f.flush()
f.close()
```

### 5. 统计出每个IP的访问量有多少？（从日志文件中查找）

```python
#!/usr/bin/env python
#!coding=utf-8
list = []
f = file('/tmp/1.log')
str1 = f.readlines() 
f.close() 
for i in str1:
    ip = i.split()[0]
    list.append(ip) 
list_num = set(list)
for j in list_num: 
    num = list.count(j) 
    print '%s : %s' %(j,num)
```

### 6. 写个程序，接受用户输入数字，并进行校验，非数字给出错误提示，然后重新等待用户输入。根据用户输入数字，输出从0到该数字之间所有的素数（只能被1和自身整除的数为素数）

```python
#!/usr/bin/env python
#coding=utf-8
import tab
import sys
while True:
    try:
        n = int(raw_input('请输入数字:').strip())
        for i in range(2, n + 1):
            for x in range(2, i):
                if i % x == 0:
                    break
            else:
                print i
    except ValueError:
        print('你输入的不是数字，请重新输入:')
    except KeyboardInterrupt:
        sys.exit('\n')
```

### 7. 抓取web页面

```python
#!/usr/bin/env python
#coding=utf-8
from urllib import urlretrieve
def firstNonBlank(lines): 
    for eachLine in lines: 
        if not eachLine.strip(): 
            continue 
    else: 
        return eachLine
    
def firstLast(webpage): 
    f=open(webpage) 
    lines=f.readlines() 
    f.close 
    print firstNonBlank(lines) #调用函数
    lines.reverse() 
    print firstNonBlank(lines)
    
def download(url= 'http://search.51job.com/jobsearch/advance_search.PHP', process=firstLast): 
    try: 
        retval = urlretrieve(url) [0] 
    except IOError: 
        retval = None 
    if retval: 
        process(retval)
        
if __name__ == '__main__': 
    download()
```

### 8. Python中的sys.argv[]用法练习

```python
#!/usr/bin/python
#coding = utf-8
import sys

def readFile(filename):
    f = file(filename)
    while True:
        fileContext = f.readline()
        if len(fileContext) ==0:
            break;
        print fileContext
    f.close()
    
if len(sys.argv) < 2:
    print "No function be setted."
    sys.exit()

if sys.argv[1].startswith("-"):

    option = sys.argv[1][1:]

    if option == 'version':
        print "Version1.2"
    elif option == 'help':
        print "enter an filename to see the context of it!"
    else:
        print "Unknown function!"
        sys.exit()
else:
    for filename in sys.argv[1:]:
        readFile(filename)
```

### 9. python迭代查找目录下文件

两种方法：

```python
#!/usr/bin/env python
import os 
dir='/root/sh'

def fr(dir):
  filelist=os.listdir(dir)
  for i in filelist:
    fullfile=os.path.join(dir,i)
    if not os.path.isdir(fullfile):
      if i == "1.txt":
        #print fullfile
    os.remove(fullfile)
    else:
      fr(fullfile)
```

```python
#!/usr/bin/env python
import os 
dir='/root/sh'

def fw()dir:
  for root,dirs,files in os.walk(dir):
    for f in files:
      if f == "1.txt":
        #os.remove(os.path.join(root,f))
        print os.path.join(root,f)
```

### 10. ps 可以查看进程的内存占用大小，写一个脚本计算一下所有进程所占用内存大小的和（提示，使用ps aux 列出所有进程，过滤出RSS那列，然后求和）

```python
#!/usr/bin/env python
#!coding=utf-8
import os
list = []
sum = 0   
str1 = os.popen('ps aux','r').readlines()
for i in str1:
    str2 = i.split()
    new_rss = str2[5]
    list.append(new_rss)
for i in  list[1:-1]: 
    num = int(i)
    sum = sum + num 
print '%s:%s' %(list[0],sum)
```

