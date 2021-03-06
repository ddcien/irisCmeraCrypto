# USB虹膜采集装置的**虹膜特征信息**数据的保护

[TOC]


## 问题描述
传统的USB接口的虹膜采集装置，多数是使用标准的UVC方案，只有极少数采用自定义传输协议，但他们均不具备虹膜图像数据加密功能。所以，黑客可以轻易地在USB通信链路上窃取用户的虹膜信息。

### 常用攻击方法 
一般攻击方式分为两步：
1. USB数据截获；
2. USB数据分析，提取虹膜信息；

#### USB数据截获
USB数据的截获，有两种方法：
1. 硬件方法：
USB数据嗅探器，是一种硬件装置，可以串联在USB主从端之间。首先，它能够完全模拟USB从主端的行为，从而USB主或从端端完全不会知道该设备的存在；然后，它能够截获USB主从端的所有通信内容。有些该类装置甚至具备USB数据分析能力，能够按照各种标准USB协议对所截获的数据进行分析，并将各种信息分门别类地分析出来。该方法，需要对原硬件设备进行改造，需要将该类装置串联在原USB通信链路上，可以加在主端，也可以加在从端，并且目前该类设备体量都稍大，用户能够很容易地觉察该设备的存在，所以该方式有一定的难度。但是随着芯片集成度和生产工艺的不断提高，相信很快该类装置就能够做都非常小，完全能够集成到USB从端，甚至集成在USB数据线上。

```sequence
Title: 正常USB主从连接情况
USB Host->USB Client:
USB Client->USB Host:

```
``` sequence
Title: 在USB主从端之间加入USB数据嗅探器的情况
USB Host-->USB Sniffer:
USB Sniffer-->USB Client:
USB Client-->USB Sniffer:
USB Sniffer-->USB Host: 
```

2. 软件方法：
市面上也存在一众软件，它们能够检测，并截获USB总线上的数据，如windows平台上的BusHound，或者Linux内核自带的USB Monitor功能等。一般该类软件并不具备强有力地分析能力，有针对行的数据分析功能，但是由于USB的各个子类协议都是公开的，还是很容易对数据进行分析，并提取出有用的信息。该方法，需要攻击者事先获取到目标机的系统管理员权限，才能安装或/和运行这些软件。

综上，黑客若想截获用户使用USB虹膜采集装置时的USB数据，还是有一定难度的。

#### USB数据分析
目前大部分USB接口的虹膜采集装置都是使用标准的UVC协议。该协议上的数据毫无秘密可言，稍微有能力的程序员均能从遵从UVC协议的USB数据流中提取图像/视频数据。

### 总结

虽然以上的攻击方法均具有一定实施难度，但是，我们不能寄希望于用户的高安全防范意识以及科技的不发展。我们要主动出击，从根本上杜绝用户虹膜信息的被盗取。



## 问题解决
根据针对虹膜识别过程中虹膜信息被盗取的高攻击方式和方法，可得知，若要从根本上解决问题，我们必须不能对**关键敏感数据**使用标准的UVC协议进行明文传输。因此，本文将按照这个思路设计并实现一套**安全的USB虹膜采集装置**，该装置作为一个USB从设备，跟USB主端进行通信，过程中**关键敏感数据**使用密文传输，确保在通线链路和USB物理层驱动中都无法截获**关键敏感数据**明文。

---
### 方案1：UVC负载数据全程加密
UVC是USB Video (device) Class的简称，它是一种专为USB视频流设备而定义的一套协议标准。目前市场上几乎所有的USB接口的视频流设备均采用该协议，同时该协议也得到主流操作系统的支持，如WINDOWS，LINUX，ANDROID，OS X等等。UVC协议标准主要规定了**基础通信协议**和**视频流/图像数据格式**两项内容。


下图为传统通用型USB接口摄像头的大致框架
![](http://7xrn7f.com1.z0.glb.clouddn.com/16-3-22/32831688.jpg)


本方案，不对UVC协议标准关于**基础通信协议**的规定作任何改动，而仅仅是在将**视频流/图像数据**即**UVC负载数据**送入UVC协议前，即，上图中，图像数据进入DMA通道前， 将其进行加密，然后在按照UVC协议标准进行封包，送入数据传输端点。

该方案，使用标准的UVC协议标准，所以USB主机端不需要修改，或者重新编写设备驱动程序，而使用默认的UVC驱动程序就可以完全驱动的该设备。但是，应用层上，由于UVC负载数据并不是视频流/图像数据明文，而是经过加密的密文，所以，应用层程序需要对数据进行解密操作后方可使用。

该方案虽然思路和实现都相对简单，但是，**视频流/图像数据**的数据量非常大，对所有这些数据进行加密解密操作将非常耗时，会严重影响**视频流/图像数据**的实时性，造成预览反馈延时，会极大地影响用户交互使用体验。


---
### 方案2：UVC负载数据大图格式加密
UVC协议标准允许同一设备支持多种不同的数据格式，如我们常用的非压缩格式有YUV，压缩格式有MJPEG等等。每种不同的数据格式，还支持多种不同的**视频流/图像数据**尺寸，例如320x240、640x480、1280x1024、1920x1080等等。不同数据格式，以及不同**视频流/图像数据**尺寸都能自由选择。

所以，本方案首先设置虹膜采集装置使用小尺寸（如320x240，640x480，或其他尺寸）的数据格式进行图像数据传输。由于小尺寸的图像数据是经过降采样的而得到的，因此图像数据中**虹膜特征信息**会被破坏，即使该数据被攻击者截获，也不能恢复出完整的**虹膜特征信息**数据。所以在预览阶段使用使用明文传输小尺寸图像数据，理论上是安全的。同时，USB主端的应用程序依然能够从小尺寸的图像数据中提取到必要的**图像质量信息**，并以此为评判当前图像质量是否能够满足后续虹膜识别算法的要求。所以，关于小尺寸的选择标准为：
1. ***绝不能恢复出足够进行虹膜识别的虹膜特征信息数***；
2. ***能够提供足够的虹膜图像质量信息***；

一旦图像质量能够满足后续虹膜识别算法的要求，立即设置使用**大尺寸**图像数据格式进行**密文**传输。这样虹膜采集装置只需要对大尺寸图像格式数据进行加密就可以了。而且**大尺寸**图像数据不是用来预览和实时反馈，所以即使对其进行加密解密操作也不会影响到用户体验。USB主端的应用程序大抵流程如下：

```flow
start=>start: Start
setupS=>operation: Set small size
readS=>operation: Grap small image
iqm=>condition: Image quality check
setupB=>operation: Change to full size
readB=>operation: Grap full size image
decrypt=>operation: Decrypt the image
iris=>operation: IRIS recognition
end=>end: End

start->setupS->readS->iqm
iqm(no)->readS
iqm(yes)->setupB->readB->decrypt->iris->end
```

该方案，同方案一一样使用标准UVC协议，不需要修改或重写设备驱动程序，同时解决了方案一因为加密解密操作而影响用户使用体验的弊端。但是，该方案还是有缺陷的：切换图像尺寸的时间不能太长，如果太长的话，我们无法保证**大尺寸**图像数据的有效性。举例说明：

我们在$T_1$时刻得到一帧小图，该小图经过USB传输到USB主端，USB主端的应用程序对其进行一系列图像质量计算，判定为符合虹膜识别算法的要求，此时时刻为$T_2$；然后USB主端应用程序执行**切换图像尺寸**操作，虹膜采集装置接收到相关指令，并开始进行切换操作，直到切换完成，此时时刻为$T_3$；然后采集到一帧大图，此时时刻为$T_4$。那么，也就是说，在$T_4$时刻采集到的大图将会用于虹膜识别算法，那么如果$T_4$与$T_1$的时间间隔$\Delta T = T_4-T_1$很长，就没有办法保证$T_4$时刻采集到的大图的有效性。


---
### 方案3：数据缓存加UVC负载数据大图格式加密
方案二解决了方案一的预览反馈信息的实时性问题，但又引入了虹膜识别图像的时间有效性的问题。在本方案中，我们在USB从端，即虹膜图像采集装置端进行改造，增加**缓冲区**、**自定义传输协议**等，已解决虹膜识别图像的时间有效性的问题。

![](http://7xrn7f.com1.z0.glb.clouddn.com/16-3-24/37554043.jpg)

1. 虹膜采集装置实时采集**大图**；
2. 将**大图**直接保存到本地的缓冲区中；
3. 将**大图**进行降采样，压缩为**小图**，并采用标准UVC协议将**小图**进行明文传输；
4. 若**小图**通过USB主端的应用程序的虹膜图像质量评判，则主端应用程序不执行**切换图像尺寸**操作，而是使用非UVC标准协议（UVC扩展协议或者厂商自定义协议），向USB从端申请当前**小图**的原图，即，存储与USB从端缓冲区内的**大图**副本。
5. USB从端将**大图**副本从本地缓冲区内取出，加密，然后传输给USB主端。
6. USB主端的应用程序接收到**大图**副本的密文，对其进行解密，然后交由后续的虹膜识别操作。

以上过程，可以确保**虹膜特征信息**数据不能在传输的过程中被截获、窃取。

#### 缓存大小的选取
该方案需要USB从端具备大容量缓存的支持，且宜用环形缓冲区。如一颗2百万像素的传感器，采用RAW8数据格式（即每个像素点使用8bit数据表示），单通道（虹膜识别算法只需要亮度信息），那么一帧这样的图像的数据量刚好是2MB。如果采用JPEG压缩，数据量为500KB到800KB（JPEG的压缩率为25%至33%）。并且，USB主端的应用程序进行虹膜图像获取、质量评判等过程中，USB从端当然不能停止采集虹膜图像，它会不间断地采集图像，存入缓存。所以，USB从端图形传感器的采集时间间隔和USB主端的应用程序进行虹膜图像获取、质量评判的时间是决定缓存大小的关键。

举例来说，USB从端图形传感器每秒能采集30帧图像，即没每帧图像的间隔时间为33毫秒；USB主端的应用程序进行虹膜图像获取、质量评判时间为100毫秒。那么缓冲区就必须至少能够缓存4帧图像，否则就也会向方案二一样，无法保证数据有效性。

#### 自定义传输协议
同时，该方案还需要修改UVC协议（添加厂商自定义协议部分，甚至，直接抛弃UVC协议，采用完全自定义协议），需要自行编写该USB设备（即虹膜采集装置）的驱动程序。同时，USB从端，需要数据加密功能，数据缓存功能，所以传统的USB CAMERA方案芯片是不能满足我们这一需要求的，必须寻求新的解决方案。

如公司前期的一些产品（EX-100，DI-300，RZ500）等，体量较大，未采用UVC协议。所以，他们的使用需要特定的驱动程序的支持，或者使用诸如linusb等三方库来实现驱动。而我公司后来生产的各类USB接口的虹膜采集设备，均采用的是标准的USB摄像头方案，采用标准的UVC协议。他们可以直接使用各个操作系统内被集成的UVC驱动程序直接驱动，而不需特定的驱动程序的支持。


## 总结
虹膜识别应用中，需要对**虹膜特征信息**数据进行保护，从而能够避免攻击者通过窃取**虹膜特征信息**数据对系统进行攻击。本文详细分析了采用标准UVC协议的USB虹膜采集装置的虹膜数据流传输过程，并针对该过程，列举了各种截获、窃取**虹膜特征信息**数据的方法。然后又针对各种攻击手段，制定保护方案：**虹膜图像数据采用加密传输，以确保数据在传输线路上不被窃取**。

同时，虹膜识别系统也是需要被保护的。即使攻击者获取了**虹膜特征信息**数据，也不能攻破虹膜识别系统。可以考虑下**人脸识别**，人脸信息完全是公开透明的，随便任何人就能采集到别人的面部信息。所以，人脸识别要保护的不是人脸面部信息本身，而是需要保护人脸识别算法。

虹膜识别的精准度仅次于DNA，远远高于人脸识别、指纹识别等其他生物识别技术。所以，我们需要开发出更加安全的整套虹膜识别系统，包括硬件，软件。硬件上，我们需要保护虹膜图像数据不能被截获、窃取；软件上，我们需要使用注入ARM的TEE，INTEL的SGX等安全架构，来实现对敏感数据，对算法过程等保护。





