---
title: Python按层级解析网页中的ul和li
url: 1982.html
id: 1982
categories:
  - Python
date: 2019-02-15 22:41:38
tags:
---

最近有需要按层级解析如下格式的内容：
```
<ul>
   <li>A</li>
   <ul>
       <li>A.1</li>
       <li>A.2</li>
   </ul>
   <li>B</li>
      <ul>
       <li>B.1</li>
       <li>B.2</li>
      </ul>
   <li>C</li>
       <ul>
       <li>C.1</li>
            <ul>
              <li>C.1.2</li>
            </ul>
       <li>C.2</li>
      </ul>
</ul>
```
做了下搜索，有人建议是使用BeautifulSoup库，但是并没有显示出层级关系。这里写了简单示例，采用嵌套函数解决上述问题。直接看代码：

#导入Beautiful库，如果没有安装就使用pip install bs4安装吧
```python
from bs4 import BeautifulSoup

myxml = """
<ul>
   <li>A</li>
   <ul>
       <li>A.1</li>
       <li>A.2</li>
   </ul>
   <li>B</li>
      <ul>
       <li>B.1</li>
       <li>B.2</li>
      </ul>
   <li>C</li>
       <ul>
       <li>C.1</li>
            <ul>
              <li>C.1.2</li>
            </ul>
       <li>C.2</li>
      </ul>
</ul>
"""
```
#ul为Soup要解析的UL，level为列表层级
```python
def findSoup(ul, level):
    print("level:"+str(level))
    for i in ul.contents:
        if '\\n' != i:
            #如果找不到li表示，这个层级已经没有下一层级了，如果有的话，嵌套该函数继续查找
            if None!=i.find('li'):
                findSoup(i, level+1) 
            else:
                print(str(i.get_text()))

#打印soup变量看看，调用BeautifulSoup后会给加上***html***和***body***标签
soup = BeautifulSoup(myxml,'lxml')
#soup.ul，也可以用soup.find('ul')
findSoup(soup.ul, 0)
```
以上执行结果如下：

level:0
A
level:1
A.1
A.2
B
level:1
B.1
B.2
C
level:1
C.1
level:2
C.1.2
C.2

可以看出列表的层级已经分析出来了。以上仅为提供一个思路，如果要做进一步分析，可自行修改。