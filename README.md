# 伪云端部署Paddle模型实现单目标垃圾分类

利用花生壳5和Flask将模型部署在远端服务器，树莓派负责图像采集，通过指定的公网端口与服务器通信，服务器接收图像后基于Paddle模型进行在线推理，将分类结果返回给树莓派，实现模型应用于硬件。

服务器配置：

| 编号 | 项目 | 配置 |  
|--|--|--|
| 1 | CPU | i9 9900KF 8C16T|
| 2 | RAM | 32GB|
| 3 | GPU | RTX 2080 8G|
| 4 | Python | 3.7 |

树莓派配置：

| 编号 | 项目 | 配置 |  
|--|--|--|
| 1 | 树莓派 | 3B+ |
| 2 | 树莓派摄像头 |  usb摄像头|
| 3 | 操作系统 | 官方操作系统 |
| 4 | 舵机 | SG90 |
| 5 | cmake | 3.10 |

## 一、伪云端实现：
使用[花生壳5](https://hsk.oray.com/) 对本地IP实现内网穿透（IP地址及端口的映射），实现公网与内网的数据通信交互，与之后介绍的基于Flask的部署方式有关。使用这种方法需考虑网络安全问题，在此不多做解释。  
<div align="center"><img  src="/src/花生壳5内网穿透示例.jpg"/></div>

## 二、服务器端模型部署：
目前，我们已经将在AI Studio上训练好的模型保存至本地，那么根据test.py中的代码，我们可以实现Paddle模型的“一次性推断”。其中，在Line 4导入的model1是由PaddlePaddle提供的残差神经网络ResNet模型库；Line 46中定义的read_image()方法中，需要手动添加一个等待预测的图片的相对路径或绝对路径。经过模型的计算，会将预测的类别存入变量lab中，并且输出。
```
/code/server/test.py
```
接下来，我们将“一次性推断”转换为“在线推理”。计划是将这些代码包装在Flask应用程序中。Flask是一个非常轻量级的Python Web框架，它允许您用最少的工作来创建一个HTTP API服务器。根据server.py代码，我们可以实现“在线推理”。
```
/code/server/server.py
```
其实从本质上来说，无论是图片还是文字或者字符，都是数据，即最原始的一串0和1组成的二进制数据，API接收或者返回图片，本质上也就是接收或返回一段数据流。
向服务器发送一张图片的方式，我们在下一个部分介绍。从服务器接收图片，我的做法是服务器端将通过base64将字节转化为图片。

将转化的图片保存到本地后，接下来进行推理的过程，与在本地进行“一次性推推断”是一致的。用Flask实现正是我们所需要的——Flask与PaddlePaddle是完全同步的，他们将按照接收到的顺序一次处理一个请求，并且可以将处理的结果返回给客户端。

在最后一行的方法app.run("", port=)中，我们需要填写用花生壳5实现内网穿透的本地IP地址及端口号，完成路由注册。

## 三、客户端上传/接收实现：
向服务器发送图片，具体的实现步骤是：编写Python脚本，使用OpenCV读取摄像头，设置延迟3秒后拍一张照片，并将照片保存在本地；接着，以二进制的方式打开，将其转换为base64后添加的List型变量image中，再post给服务器。其中，Line 30中定义的方法requests.post(url,data=res)，第一个参数需要填写一个和服务器端相同的IP地址，由于我们利用花生壳5实现内网穿透，可以脱离局域网的限制，可以利用公网IP与服务器实现通信，因此，这个参数url可以填写花生壳5提供的域名及端口号。
```
/code/client/controller.py
```
经过服务器的处理，会将推理生成的Label值以byte型变量传回客户端，具体可参考Line31，32代码实现的过程，利用str()强制类型转换成str型的label后，根据判断调用树莓派GPIO的指定端口，实现舵机的运作。
