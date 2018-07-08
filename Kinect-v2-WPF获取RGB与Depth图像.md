---
title: Kinect v2 + WPF获取RGB与Depth图像
date: 2017-09-04 14:51:07
categories:
  - WPF
  - Kinect
---

Kinect V2的Depth传感器采用的是「Time of Flight(TOF)」的方式，
通过从投射的红外线反射后返回的时间来取得Depth信息。
<br>
本文将Kinect v2 + WPF来得到Kinect所获取的RGB（1920×1080）及Depth（512×424）图像

## 第一步：Kinect v2开发环境（仅限于本文）

* Visual Studio 2017
* 下载[Kinect for Windows SDK 2.0](https://developer.microsoft.com/zh-cn/windows/kinect/tools)并安装

## 第二步：创建工程
1. 打开Visual Studio 2017, 创建一个WPF工程，名字随意（本文中例子为KinectTest）
1. 在Solution Explorer中，右键单击KinectTest，在右键菜单中选择“Add Reference…”。弹出对话框后，在程序集中选择“Microsoft.Kinect”

## 第三步：编写代码
MainWindow.xaml代码
```xaml
<Window x:Class="KinectTest.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:KinectTest"
        mc:Ignorable="d"
        Title="KinectTest" Height="1080" Width="1600">
    <Grid>
        <Image Name="RGB_Viewer" Source="{Binding ColorSource}" HorizontalAlignment="Left" Width="960" Height="540" Margin="0, 0, 0, 0"></Image>
        <Image Name="Depth_Viewer" Source="{Binding DepthSource}" Width="512" Height="424" HorizontalAlignment="Right"></Image>
    </Grid>
</Window>
```
<br>
MainWindow.xaml.cs代码
```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;
using Microsoft.Kinect;

namespace KinectTest
{
    /// <summary>
    /// MainWindow.xaml 的交互逻辑
    /// </summary>
    public partial class MainWindow : Window
    {
        private KinectSensor kinect = null; //用来标记kinect摄像头的实例

        private ColorFrameReader colorFrame = null; //用来处理和保存摄像头传过来的彩色影像帧数据
        private WriteableBitmap colorBitmap = null; //用来向Image控件中填充彩色影像数据
        private FrameDescription colorFrameDescription = null; //用来描述彩色影响帧数据形态的参数

        private const int MapDepthToByte = 8000 / 256;
        private DepthFrameReader depthFrame = null;
        private WriteableBitmap depthBitmap = null;
        private FrameDescription depthFrameDescription = null;
        private byte[] depthPixels = null;

        public MainWindow()
        {
            this.kinect = KinectSensor.GetDefault(); //获取第一个或默认的Kinect摄像头

            this.colorFrame = kinect.ColorFrameSource.OpenReader(); //打开彩色影像数据的获取接口
            this.colorFrame.FrameArrived += ColorFrame_FrameArrived; //建立监听事件，当有彩色影像数据帧到达时触发
            this.colorFrameDescription = this.kinect.ColorFrameSource.CreateFrameDescription(ColorImageFormat.Bgra);
            //按照Bgra格式设定数据帧的描述信息（B:Blue,G:Green,R:Red,A:Alpha）
            this.colorBitmap = new WriteableBitmap(colorFrameDescription.Width, colorFrameDescription.Height, 96.0, 96.0, PixelFormats.Bgr32, null);
            //根据数据帧的宽高创建colorBitmap的实例

            this.depthFrame = kinect.DepthFrameSource.OpenReader();
            this.depthFrame.FrameArrived += DepthFrame_FrameArrived;
            this.depthFrameDescription = kinect.DepthFrameSource.FrameDescription;
            this.depthBitmap = new WriteableBitmap(depthFrameDescription.Width, depthFrameDescription.Height, 96.0, 96.0, PixelFormats.Gray8, null);
            this.depthPixels = new byte[this.depthFrameDescription.Width * this.depthFrameDescription.Height];

            this.kinect.Open(); //启动kinect摄像头
            this.DataContext = this;
            InitializeComponent();
        }

        private void ColorFrame_FrameArrived(object sender, ColorFrameArrivedEventArgs e)
        {
            using (ColorFrame frame = e.FrameReference.AcquireFrame()) //建立一个ColorFrame的实例frame保存送过来的帧，通过using保证走完函数后及时释放相应资源
            {
                if (frame != null)
                {
                    this.colorBitmap.Lock(); //锁定一下数据文件，准备进行填充
                    frame.CopyConvertedFrameDataToIntPtr(this.colorBitmap.BackBuffer, (uint)(this.colorFrameDescription.Width * this.colorFrameDescription.Height * 4), ColorImageFormat.Bgra);
                    //提供给函数一个空间接收帧数据，将数据储存进colorBitmap的后台缓存中
                    this.colorBitmap.AddDirtyRect(new Int32Rect(0, 0, this.colorBitmap.PixelWidth, this.colorBitmap.PixelHeight));
                    //设定colorBitmap需要更改的位图区域，此处设定为整个图片
                    this.colorBitmap.Unlock(); //解锁位图资源
                }
            }
        }

        private void DepthFrame_FrameArrived(object sender, DepthFrameArrivedEventArgs e)
        {
            using (DepthFrame frame = e.FrameReference.AcquireFrame())
            {
                if (frame != null)
                {
                    this.depthBitmap.Lock();
                    using (KinectBuffer _buffer = frame.LockImageBuffer())
                    {
                        Cvt_process(_buffer.UnderlyingBuffer, _buffer.Size, frame.DepthMinReliableDistance, ushort.MaxValue);

                        this.depthBitmap.WritePixels(
                            new Int32Rect(0, 0, this.depthBitmap.PixelWidth, this.depthBitmap.PixelHeight),
                            this.depthPixels,
                            this.depthBitmap.PixelWidth,
                            0);
                    }
                    this.depthBitmap.Unlock();
                }
            }
        }

        private unsafe void Cvt_process(IntPtr depthFrameDate, uint depthFrameDateSize, ushort minDepth, ushort maxDepth)
        {
            ushort* frameDate = (ushort*)depthFrameDate;//强制深度帧数据转为ushort型数组，frameDate指针指向它
            for (int i = 0; i < (int)(depthFrameDateSize / this.depthFrameDescription.BytesPerPixel); ++i)
            {
                //深度帧各深度（像素）点逐个转化为灰度值
                ushort depth = frameDate[i];
                this.depthPixels[i] = (byte)((depth >= minDepth) && (depth <= maxDepth) ? (depth / MapDepthToByte) : 0);
            }
        }

        public ImageSource ColorSource
        {
            get
            {
                return this.colorBitmap;
            }
        }

        public ImageSource DepthSource
        {
            get
            {
                return this.depthBitmap;
            }
        }
    }
}
```