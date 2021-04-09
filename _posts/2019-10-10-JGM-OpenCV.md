---
layout: post
title:  "在《家国梦》中学习openCV"
author: sal
categories: [ Coding ]
tags: [ OpenCV ]
image: https://pic1.zhimg.com/v2-e7532cece10e42968918b9ece25bd97b_b.jpg
rating: 4.5
---

记我是如何完善一个家国梦py自动脚本的
## 游戏类型
这一看就知道是个挂机(肝)游戏. 在线与离线的差距巨大.
在线,可以干很多事情,收金币,升级建筑,收火车,开红包
离线,啥都没有(只有线性增长的金币)
## 前因
这么简单个东西,不随随便便用py写个脚本?
先上最大交友网站搜下家国梦,看看有没有人已经写了,免得重复造轮子
![1](https://pic1.zhimg.com/v2-6d21f0a1733f564ce183353054aa6384_b.png)

果然有, 还挺多,有py的,有什么鬼按键精灵的,还有AutoJS的. 
计系人怎么会用按键精灵这种玩意? AutoJS会有内存泄漏问题, 作者早就跑路了, 所以最后选了@Jiahonzheng写的这个脚本作为切入点.
他也在知乎写了一篇文章介绍了他当时的脚本情况:
[Jiahonzheng：《家国梦》游戏自动化测试](zhuanlan.zhihu.com)

当时的实现主要就是两点:
调用opencv模板匹配检测货物, 然后拖动到预设位置
自动扫金币
虽然说已经大四了,但我openCV还没怎么用过. 之前也就做用Yolov3做目标检测的时候拿它来画个框框,然后把静态画面组合成视频. 
既然如此,就当是学习openCV好了.
## 开发过程
1. 实现自动升级政策功能
这游戏的政策系统乍一看不起眼, 进去一翻才知道, 原来政策加成如此之高,到最后甚至加成好几千%:

![工业建筑加成3000%](https://pic2.zhimg.com/v2-bda57838ab4587ccc0b375d63b337cfd_b.png)
也就是说, 只要政策一完成, 马上升级, 就能更快的在政策上获取优势. 马上开干, 刻不容缓.
政策完成时会在政策中心出现一个带绿色箭头的气泡: 

![左上角有个绿色箭头的气泡](https://pic4.zhimg.com/v2-e9da573829cebdf1a6481e0d45ee6313_b.png)
观察画面发现, 整个界面中好像没别的地方有类似这么耀眼的绿色了, 于是调用openCV, 分离画面的三个通道, 对绿色通道进行二值化:

![二值化后的绿色箭头](https://pic2.zhimg.com/v2-5144ce6acc98d39bbe0f9f16ef439d15_b.png)
实际上我还加了对红色通道和蓝色通道的处理, 最后把三个通道二值化后的图片相与:
```py
def findGreenArrow(screen):
        '''
        检测政策界面中 绿箭头的中心位置
        @return: 绿箭头坐标list
        '''
        if screen.size:
            dstPoints = []
            img2 = cv2.split(screen)
            # 分离R 二值化
            ret, dst1 = cv2.threshold(img2[0], 20, 255, cv2.THRESH_BINARY_INV)
            # 分离G 二值化
            ret, dst2 = cv2.threshold(img2[1], 220, 255, cv2.THRESH_BINARY)
            # 分离B 二值化
            ret, dst3 = cv2.threshold(img2[2], 20, 255, cv2.THRESH_BINARY_INV)
            img2 = dst1&dst2&dst3 # 相与
            # 找轮廓
            cnts = cv2.findContours(img2, cv2.RETR_EXTERNAL,cv2.CHAIN_APPROX_SIMPLE)
            if cnts[1]:
                for c in cnts[1]:
                    # 获取中心点
                    M = cv2.moments(c)
                    cX = int(M["m10"] / M["m00"])
                    cY = int(M["m01"] / M["m00"])
                    #
                    dstPoints.append((cX,cY))
            return dstPoints
```
然后政策中心里面, 绿色箭头是一模一样的, 所以可以复用这个方法(实际上本来就是为了检测里面这些箭头):
![](https://pic3.zhimg.com/v2-99b0d536c9029917e097ed60a2622be2_b.png)
![](https://pic2.zhimg.com/v2-5512882cc1618b2197197c5da2b065fd_b.png)

检测实现了, 每次只要先看看政策中心有没有冒绿色箭头气泡, 如果有的话就点击政策中心, 然后确认一下上次完成的政策, 拉到顶(复位), 再慢慢往下拉, 拉一次找一次, 如果有绿色箭头就点它, 最后回到主界面就完成了这一功能的执行.
2. 收火车的改进
之前的收货都是调用openCV自带的模板检测货物, 然后根据我们手动填写好的`货物-建筑` 字典来拖动货物. 
本来的模板匹配准确率有点低的, 于是我尝试用 `SIFT` 算法(也是openCV自带的), 即特征值匹配来匹配货物:

![打上绿色遮罩希望它能减少错误(滑稽](https://pic1.zhimg.com/v2-eba2de03012d5c5f218b27945284b414_b.png)
**但是**! 无论是模板匹配还是特征值匹配, 总有两个东西配不上或者配混了: 石头 和 煤炭 
**再者**, 第一车厢和第三车厢的背景差别有点大, 在第一车厢截取的货物图片在第三车厢匹配不到, 反过来也是. 摊手...
**再再者**, 每次换建筑都要自己去修改那个映射字典. 实在麻烦.
于是想到这游戏收货的时候还有个特别重要的信息: **按住货物, 对应的建筑会有绿光包围**
那我能不能检测这个绿光呢? 当然可以. 但是直接检测这个包围的绿光会有个问题:

 ![其它地方也一大堆绿色啊](https://pic1.zhimg.com/v2-a23de76144a4f6162748e5f5e8c3fa08_b.png)
到处都是绿色....
主要是这绿色不像之前的那个绿色箭头那么均匀, 这里啥绿都有. (要不我训练个YOLO模型让它检测? )
还有, 咋按着的时候截图啊? 这边翻了一下只有click(), swipe(),drag()等方法, 都是执行完后(手指)会抬起屏幕的.  于是我又到 github 上翻翻有没有人和我一个想法的, 还真就发现了这么个东西 yusanshi/Jiaguomeng_Assist:
```py
def get_screenshot_while_touching(self, filename, location, pressed_time=6):
        '''
        Get screenshot with screen touched.
        Multiprocess or Multithread is needed.
        '''
        p = multiprocessing.Process(
            target=self.tap_continuously, args=[location, pressed_time])
        p.start()
        time.sleep(2)
        self.get_screenshot(filename)
        time.sleep(0.5)
        p.join()
```
它是用了py自带的多进程模块, 新开了个进程执行adb命令实现"按住", 然后本进程截图再wait/join新进程. 
我咋就没想到呢, 于是我也照葫芦画瓢, 截图之前先开个新进程执行adb命令, 这边uiautomator2再截图. 
能用是能用, 就是效果不好. 主要是执行adb命令 到 设备响应这个命令 的时间有点飘, 在我的XZ1上 几乎瞬间响应, 在XZ上要1秒到2秒后才响应, 在MuMu模拟器上稳定2秒响应...这...也不是不能搞, 就是sleep的冗余等待时间太长了, 不爽.
但是我用的是uiautomator2 这么强大的自动化测试框架了啊! 混用原生 shell adb 命令这不是没事给自己找事么.  仔细找找， 应该会有类似按住的命令吧. 
果不其然，在u2的readme文档中找到了这个：
> Touch and drap (Beta)
> 这个接口属于比较底层的原始接口，感觉并不完善，不过凑合能用。注：这个地方并不支持百分比

```py
d.touch.down(10, 10) # 模拟按下
time.sleep(.01) # down 和 move 之间的延迟，自己控制
d.touch.move(15, 15) # 模拟移动
d.touch.up() # 模拟抬起
```
nice啊， 果然用一个东西之前最好先看看它的文档，别没事给自己找事。
测试了一下，发现是可以用的，截图后再d.touch.up()就行，而且效果很好，响应非常快，但是为了保险起见，执行d.touch.down()之后还是sleep几十ms再截图比较好。
按住后的截图实现了，那么只要按住前截一张图，按住后也截一张图，前后两张图直接相减,不就能得到绿光了吗?
但是直接相减,效果并不好:
![](https://pic3.zhimg.com/v2-9c95cfcd6731661a9544d0f83d0e4a4a_b.png)

不仅是这个绿光从无到有,其他地方还有小车移动,人的走动,甚至些许光线的变化,都会导致这些花花绿绿的东西出现. 
更重要的一点是: openCV原生的图片读取都是uint8型组成的矩阵,也就是0~255, 换句话说就是0-1=255而不是-1. 于是在两张图片相减之前, 需要转换成有符号数:
```py
        # 转换成有符号数以处理相减后的负值
        screen_before = screen_before.astype(np.int16)
        screen_after = screen_after.astype(np.int16)
```
由于绿色通道是主要变化,  所以我们只需要处理绿色通道即可
```py
        B,G,R = cv2.split(diff)
        # 负值取0
        G[G < 0] = 0
        G = G.astype(np.uint8)
```
![](https://pic1.zhimg.com/v2-8d337e1f41e64afc388760c73001f174_b.png)
然后取一个区间范围的亮度值给它二值化, 再给他均值模糊一下处理小噪点:
```py
        # 二值化后相与, 相当于取中间范围内的值
        ret, G1 = cv2.threshold(G, 140, 255, cv2.THRESH_BINARY_INV)
        ret, G2 = cv2.threshold(G, 22, 255, cv2.THRESH_BINARY)
        img0 = G1&G2
        img0 = cv2.medianBlur(img0,9)
```
![](https://pic1.zhimg.com/v2-9145f084040fc09a0b23d5f66d08b014_b.png)
现在只需要对每个建筑物中心划一个小矩形来求矩形内平均亮度, 最后返回平均亮度最大的那个建筑即可.
3. 检测火车是否到来
有了上面的经验,检测火车就方便多了. 其实检测火车就是为了收货物, 那我们只需要检测货物即可.
可货物千变万化....有没有注意到每个货物下面都会有个X2,X3 的数字?
对了, 检测那个叉叉就好. 由于这个叉叉太小, 分辨率太低, SIFT算法根本匹配不上特征(可能是我参数调不对), 那索性还是检测灰度图二值化后每个叉叉区域的平均亮度好了:
![](https://pic3.zhimg.com/v2-6524b7e15614dddbb56dc5bf24a324ba_b.png)
然后返回平均亮度超过设定阈值的位置即可.
到这里  功能实现都差不多了, 开红包没有做是因为想体验抽卡的乐趣233, 其它再复杂一点的功能就应该用到OCR了, 这个工程量有点大, 还得训练这个幼圆字体的识别,权衡一下还是放弃了.
对了, 我的项目地址在这里,觉得好用的star一个哈
[Jiahonzheng/JGM-Automator](https://github.com/Jiahonzheng/JGM-Automator)
如果你有什么好的点子, 欢迎提pr.
<完>