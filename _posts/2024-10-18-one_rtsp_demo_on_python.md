---
layout: post
title: 一个使用 python 进行 rtsp 拉流的例子
date: 2024-10-18
tags:
  - rtsp
  - python
  - opencv
comments: true
author: sgarifool
toc: false
---

> 本文实现了一个使用 python-opencv 进行 rtsp 拉流的 demo, 参考了大佬的案例和 chat 的回答

<!-- more -->

近在做一个与船舶辅助系统有关的项目, 做了很久感觉没有什么实际的进展, 不久前解决了一个有关 rtsp 拉流不同步的问题, 想着还是写一篇 blog 至少留下一点痕迹吧

### 1. 两个 rtsp 流的同步

最初我预想的逻辑非常简单, 由于使用的是 opencv 库, 对于一个 rtsp 流的拉取可以如下编写: (这个 demo 里的视频流可以按 "q" 键进行视频流的退出)

```python
import cv2 as cv  
  
url = 'rtsp://admin:spark@123456@192.168.1.64'  
cap = cv.VideoCapture(url)  
  
while 1:  
    ret, image = cap.read()  
    cv.imshow("frame", image)  
    if cv.waitKey(1) & 0xFF == ord("q"):  
        break
```

那如果有两个 rtsp 流直接再同理复制一份不就好了吗, 于是改为下述代码: 

```python
import cv2  
  
url_1 = 'rtsp://admin:spark@123456@192.168.1.64'  
url_2 = 'rtsp://admin:spark@123456@192.168.1.65'  
cap_1 = cv2.VideoCapture(url_1)  
cap_2 = cv2.VideoCapture(url_2)  
  
while 1:  
    ret_1, image_1 = cap_1.read()  
    ret_2, image_2 = cap_2.read()  
    cv2.imshow("frame1", image_1)  
    cv2.imshow("frame2", image_2)  
    if cv2.waitKey(1) & 0xFF == ord("q"):  
        break
```

但是在使用上述代码测试的时候, 发现 "frame1" 和 "frame2" 之间有 1s 左右的延时, frame2 的流要快一些, 在我目前的认知里, 代码运行的速度都是非常快的, 这两个视频流在 `cv.read()` 的时候的间隔应该没有这么大 (后面其实才意识到根本不是这样的), 于是我将代码扔给了 chat, 让它分析一下

chat 的部分回答如下: 


```markdown
3. OpenCV处理机制
**原因** ：OpenCV在读取多路视频流时，可能会因为内部缓冲机制而导致一定的延时。
**解决方法** ：
* 使用多线程或多进程来同时读取视频流。这样可以减少主线程等待的时间。
* 调整OpenCV的内部缓冲区大小。可以通过设置OpenCV的属性来尝试优化。
```

chat 还给出了一个示例: 

```python
import cv2  
import threading  
  
  
def read_camera(cap, frame_buffer):  
    while True:  
        ret, frame = cap.read()  
        if not ret:  
            break  
        frame_buffer.append(frame)  
  
  
url_1 = 'rtsp://admin:spark@123456@192.168.1.64'  
url_2 = 'rtsp://admin:spark@123456@192.168.1.65'  
  
cap_1 = cv2.VideoCapture(url_1)  
cap_2 = cv2.VideoCapture(url_2)  
  
frame_buffer_1 = []  
frame_buffer_2 = []  
  
# 创建并启动线程  
thread_1 = threading.Thread(target=read_camera, args=(cap_1, frame_buffer_1))  
thread_2 = threading.Thread(target=read_camera, args=(cap_2, frame_buffer_2))  
  
thread_1.start()  
thread_2.start()  
  
while True:  
    if len(frame_buffer_1) > 0 and len(frame_buffer_2) > 0:  
        image_1 = frame_buffer_1.pop(0)  
        image_2 = frame_buffer_2.pop(0)  
        cv2.imshow("frame1", image_1)  
        cv2.imshow("frame2", image_2)  
  
    if cv2.waitKey(1) & 0xFF == ord("q"):  
        break  
  
# 结束线程  
thread_1.join()  
thread_2.join()  
  
cap_1.release()  
cap_2.release()  
cv2.destroyAllWindows()
```

但是测试发现, 上述代码的两个 rtsp 流的延时并没有得到改善

于是我在 `while` 循环里取视频帧之前, 先 `print` 一下 `frame_buffer` 的 `size` , 看看每个 `buffer` 里有多少帧, 得到的打印输出如下: 

```shell
frame_buffer_1 = 29
frame_buffer_2 = 1
frame_buffer_1 = 28
frame_buffer_2 = 1
```

发现 frame1 的 `buffer` 里有很多帧, 所以每次取的帧, 都不是实时的最新的帧, 于是引起了两个流不同步的问题

关于如何同步 rtsp 流, 我在一篇博客[^1] 里找到了大佬的逻辑, 顺着逻辑, 我在取视频帧之前下加入了丢弃的逻辑: 

```python
while True:  
    if len(frame_buff_1) > 0 and len(frame_buff_2) > 0:  
        print(len(frame_buff_1))  
        while len(frame_buff_1):  
            image_1 = frame_buff_1.pop(0)  
        print(len(frame_buff_2))  
        while len(frame_buff_2):  
            image_2 = frame_buff_2.pop(0)
```

上述逻辑, 可以保证读到的视频流都是最新的 (其实这里应该按照参考的博客里, 将 list 替换成队列, 来避免数据竞争的问题)

### 2. 退出逻辑的改进

由于此时取视频的逻辑是在新的线程里进行的, 并且没有退出逻辑, 所以, 当按 "q" 键的时候, 主线程会退出, 而子线程还会一直运行

参考的文章[^1] 里是直接使用的 `cap.release()` 的方法直接释放资源, 但是在终端里会有一些报错

所以在 `read_camera()` 函数里加入了退出的逻辑

```python
stop = False  
  
  
def read_camera(cap, frame_buff):  
    while stop is not True:  
        ret, frame = cap.read()  
        if not ret:  
            break  
        frame_buff.append(frame)
# ...
while True:  
    # ...
    if cv2.waitKey(1) & 0xFF == ord("q"):  
        stop = True  
        break

```

### 参考文献

[^1]: [https://blog.csdn.net/sheqianweilong/article/details/105657100](https://blog.csdn.net/sheqianweilong/article/details/105657100)
