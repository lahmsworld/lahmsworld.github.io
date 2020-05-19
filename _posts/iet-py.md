---
layout:     post
title:      "iet.py"
subtitle:   "实现iet与xml的互相转换"
date:       2020-05-19 11:18:00
author:     "Umo"
header-img: "img/post/inbox.jpg"
catalog: true
tags:
    - Python
    - 技术
---



刚刚才学到，python官方的命名规范是下划线命名法（又叫蛇形命名法，Python本质实锤了），而不是我喜欢用的驼峰命名法，Java才是驼峰命名法……以后要注意了。

## 需求

Where there is a need, there is programming. （伪）

这次的需求非常简单：在翻译游戏文本时，从游戏里提取出的游戏脚本夹杂着大量控制符。要翻译文本的话就得把文本部分全提取出来，又得把控制符好好保护起来，确保翻译前后控制符的内容和位置没有变化。脚本的格式是.iet，把他转化成便于翻译软件识别的.xml比较可行。

其实说得再多也不如直接看例子。

这是iet文件：

```bash
[bg_hyojib bgb="bg_black.png"]
[bg_kirikae]
[bg_hyojib bgb="bg_road1.png"]
[bg_kirikae]
　私は走り続けている。[sl]
　少しも息が切れたりしない。[sl]
　どこから力を得ているのか、自分でもわからない。[pg]


　走った。[pg]


[bgm bgm="bgm00" lp="true"]
「あは」[sl]
　笑った。[pg]


　速度が問題なのだと、柾木君は言った。[pg]
```

把它变成这样：

```xml
<!--[bg_hyojib bgb="bg_black.png"]-->
<!--[bg_kirikae]-->
<!--[bg_hyojib bgb="bg_road1.png"]-->
<!--[bg_kirikae]-->
<sl>　我持续地奔跑着。</sl>
<sl>　一点都没觉得气喘。</sl>
<pg>　这股力量哪里来的，我不知道。</pg>
<!--br-->
<!--br-->
<pg>　奔跑着。</pg>
<!--br-->
<!--br-->
<!--[bgm bgm="bgm00" lp="true"]-->
<sl>“哈……”</sl>
<pg>　我笑了。</pg>
<!--br-->
<!--br-->
<pg>　柾木君说过，速度才是关键。</pg>
```

xml的注释在导入时会被翻译软件忽略。

**我就是喜欢这种简单粗暴的需求。**开始吧！

## iet.py

先导入要用的包。

```python
import os, sys
```

建立一个iet类，定义encode函数，传入参数：输入和输出路径。

本来想用readline()，但是看到说readline比readlines慢，只要内存够都应该使用readlines的时候

我：“就这？文本文档能多大？给我全读！”

```python
class iet:
    def encode(ietFile, xmlFile):
        with open(ietFile, mode='r',encoding='UTF-8-sig') as r:
            content = r.readlines() #全部读取，按行导出一个列表
        contentNew = [] #建立一个处理过的内容的容器
```

然后识别每一行文本，这行是文本还是纯控制符还是纯空白？

好多if……

```python
        for line0 in content : #process each line
            line = line0[:-1] #del \n
            line = line.replace('--','Doublehyphen') #xml注释中不允许出现俩减号，如果原文里有要替换掉
            if line.startswith('//'):
                line = '<!--'+line+'-->'
            elif line == '':
                line = '<!--br-->'
            elif line.startswith('[') and line.endswith(']'):
                line = '<!--'+line+'-->'
            elif line.endswith('[sl]'):
                line = '<sl>'+line[:-4]+'</sl>'
            elif line.endswith('[pg]'):
                line = '<pg>'+line[:-4]+'</pg>'
            else:
                line = '<others>'+line+'</others>'
            line = line + '\n'
            contentNew.append(line)
```

接着把处理好的contentNew成xml文件，要遵守规范：

```python
        #Make XML file
        with open(xmlFile, mode='w',encoding='UTF-8-sig') as w:
            w.write('<?xml version="1.0" encoding="utf-8"?>\n')
            w.write('<iet>\n')
            for line in contentNew:
                w.write(line)
            w.write('</iet>')
```

最后再检查一下xml语法有没有错误：

```python
        try:
            xmlCheck(xmlFile)
            print('%s created' % xmlFile)
        except Exception as e:
            print('Error found in file:%s' % e)
            
from xml.sax.handler import ContentHandler
from xml.sax import make_parser
def xmlCheck(fileName):
    parser = make_parser()
    parser.setContentHandler(ContentHandler())
    parser.parse(fileName)
```

转换文件的部分写好了，但总不能一个一个转换，还要写一个批处理：

```python
    def encodeBash(path):
        parents = os.listdir(path)
        dir0 = os.path.join(path,'xml')# creat a child dir to store
        for parent in parents:
            child = os.path.join(path,parent)#now working on
            if os.path.isdir(child):
                iet.encodeBash(child)#recursive
            elif child.endswith('.iet'):
                try:
                    os.mkdir(dir0)
                except FileExistsError:# ignorance
                    pass
                name0 = str(parent).split('.')[0] #filename
                parent0 = name0 + '.xml'
                child0 = os.path.join(dir0,parent0)
                iet.encode(child,child0)
            else:
                pass
```

这个文件遍历比之前的inbox.py强一点了。

光写了iet转xml，将来翻译完成后xml还要转回iet，再写一个删除所有xml标记的：

```python
    def decode(xmlFile, ietFile):
        with open(xmlFile, mode='r',encoding='UTF-8-sig') as r:
            content = r.readlines()
        contentNew = []
        for line in content:
            line = line.replace('Doublehyphen','--')
            line = line.replace('<!--br-->','')
            line = line.replace('<!--','')
            line = line.replace('-->','')
            line = line.replace('<others>','')
            line = line.replace('</others>','')
            line = line.replace('<sl>','')
            line = line.replace('<pg>','')
            line = line.replace('</sl>','[sl]')
            line = line.replace('</pg>','[pg]')
            if line not in ['<?xml version="1.0" encoding="utf-8"?>\n','<iet>\n','</iet>']:
                contentNew.append(line)
        #Make iet file
        with open(ietFile, mode='w', encoding='UTF-8-sig') as w:
            for line in contentNew:
                w.write(line)
        print ('%s created' % ietFile)
```

往回转的批处理就不写了，大同小异。

## 总结

两个批处理中都用到了文件遍历，但是由于我不知道怎么给函数传递“执行某个函数”的参数，所以只好复制粘贴文件遍历的部分。

本来想彻底模拟程序读取脚本文件的过程，看见“//”就忽略一行，看见“[”就识别控制符开始，但是把控制符也转换成标签的话还得在翻译程序端设置忽略标签，好麻烦，直接转成注释算了，又不用管控制符里内容，不知道现在显示的立绘是谁的的话翻一下源文件就成了。

不过，翻译软件在导出翻译文件的时候要是把我的注释全删除了就有意思了……