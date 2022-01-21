# 摄像头的基本使用

-------------------------------------

## 1. CSI摄像头

首先将讲解CSI摄像头的使用：

![](../_static/useCamera/01.png)

### 1.1 picamera
> 树莓派自带的摄像头为CSI摄像头，在树莓派平台中，发行版的Raspbian默认安装了树莓派平台的picamera:

[相关官方文档链接](https://picamera.readthedocs.io/en/release-1.13/index.html)

> 执行该脚本将录制一段10秒的视频到本地：

```python
import picamera

camera = picamera.PiCamera()
camera.resolution = (640, 480)
camera.start_recording('my_video.h264')
camera.wait_recording(10)
camera.stop_recording()
```

> 但由于picamera输出的数据是RGB格式的，需要通过Numpy转换为BGR格式，代码如下：

```python
import time
import picamera
import numpy as np
import cv2

with picamera.PiCamera() as camera:
    camera.resolution = (320, 240)
    camera.framerate = 24
    time.sleep(2)
    image = np.empty((240 * 320 * 3,), dtype=np.uint8)
    # save as bgr format for OpenCV
    camera.capture(image, 'bgr')
    image = image.reshape((240, 320, 3))
    cv2.imshow("img", image)
    if(cv2.waitKey(0) == ord('q')):
        exit(0)
```

> 当然，picamera 也提供了PIRGBArray对象，用来保存RGB 图像数据,下面将在capture_continuous函数中，不断地把RGB转换成BGR图像，供OpenCV显示，以形成视频流。  

```python
from picamera.array import PiRGBArray
from picamera import PiCamera
import time
import cv2

# initialize the camera and grab a reference to the raw camera capture
camera = PiCamera()
camera.resolution = (640, 480)
camera.framerate = 32
rawCapture = PiRGBArray(camera)
# allow the camera to warmup
time.sleep(0.1)
# capture frames from the camera
for frame in camera.capture_continuous(rawCapture, format="bgr", use_video_port=True):
    image = frame.array
    # show the frame
    cv2.imshow("Frame", image)
    # prepare for net stream
    rawCapture.truncate(0)

    if(cv2.waitKey(1) == ord('q')):
        cv2.destroyAllWindows()
        break;
```

> 但由于picamera为树莓派或兼容树莓派平台专用的库，且仅能供树莓派使用，下面将介绍更为通用的使用方法。


### 1.2 V4L2

目前,CSI摄像头可通过picamera等库进行调用，但在Raspbian系统中，并不属于一个video设备而存在，若想将CSI摄像头作为一个视频设备文件使用，则要开启V4L2(Video for Linux 2)模块以开启：

开启方法如下:

```shell
sudo vim /etc/modules
```

在末尾添加一行：

```shell
bcm2835-v4l2
```

此时，CSI摄像头便成为了一个video设备，此时在终端执行：

```shell
ls -ltrh /dev/video*
```

便可查看到video设备，其中video0便是CSI摄像头：

## 2. USB摄像头

树莓派除了支持CSI摄像头外，同样也支持USB摄像头：

若采用USB摄像头，若摄像头本身免驱，接入树莓派后，将直接识别的video设备，在/dev下可找到对应的video设备，在中断，输入命令：

```shell
ls -ltrh /dev/video*
```

即可查询到新接入的video设备。

但使用Python，C++使用摄像头，区分不同的摄像头时：

c++：

```c++
VideoCapture cap(0);
```

python:

```python
cap = cv2.VideoCapture(0)
```

传入参数0代表使用CSI摄像头，传入其它参数代表使用其它video序号对应的USB摄像头。  


## 3. 网络摄像头

除了使用CSI摄像头，USB摄像头外，还可以将手机变成一个IP摄像头，具体方法如下：

在手机应用市场中，搜索这款名为IP摄像头的APP

安装好后，连接wifi，确保树莓派和手机处在同一局域网内，手机打开IP摄像头软件，同时点击打开IP摄像头服务器，即可将手机变成一个IP摄像头。如下图所示：

此时,在浏览器中打开该IP地址，输入用户名:admin，密码:admin，即可浏览摄像头画面：

此时说明手机成为了一个IP摄像头，此摄像头可供OpenCV调用。

OpenCV调用IP摄像头的源程序如下：

### 3.1 Python3

```python
#!/usr/bin/env python

'''
Waveshare OpenCV Tutorial
01_IP_Camera.py
A demo to show whether The OpenCV and IP camera is well installed
'''

import numpy as np
import cv2

def main():
    print("OpenCV Version:{}".format(cv2.__version__))
    # 0: use CSI camera,1：use USB camera
    ip_camera_url = 'http://admin:admin@192.168.10.215:8081'
    cap = cv2.VideoCapture(ip_camera_url)
    if(not cap.isOpened()):
        print("can't open this camera")

    while(True):
        ret, FrameImage = cap.read()
        if ret == True:
            cv2.imshow('Camera Capture',FrameImage)
            #Press Q to quit
            if (cv2.waitKey(1)) == ord('q'):
                cap.release()
                break
        else:
            break

if __name__ == '__main__':
    print(__doc__)
    main()
    # Release resource
    cv2.destroyAllWindows()
```

### 3.2 c++ 

```c++
#include<iostream>
#include<opencv2/opencv.hpp>
#include<opencv2/highgui/highgui.hpp>
#include<opencv2/imgproc/imgproc.hpp>

using namespace std;
using namespace cv;

int main()
{
    cout << "Waveshare OpenCV Tutorial"<< endl;
    cout << "00_Test_OpenCV"<cout << "A demo to show whether The OpenCV and IP camera is well installed" <cout << endl;
    cout << "OpenCV Version:" << CV_VERSION << endl;
    const string ip_camera_url = "http://admin:admin@192.168.10.215:8081";
    VideoCapture cap(ip_camera_url);
    if(!cap.isOpened())
    {
        cout<<"can't open this camera"<return -1;
    }    
    Mat FrameImage;
    while(1)
    {
        cap>>FrameImage;
        imshow("Camera Capture",FrameImage);
        //Press Q to quit
        if(char(waitKey(1)) == 'q')
        {
            break;
        }
    }
    return 0;
}
```

