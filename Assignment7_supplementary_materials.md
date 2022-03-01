# Assignment7_supplementary_materials

视觉组考核(OpenCV)

---

## 关于评分方案

* **采用方案1**，从而更好系统地了解大家的完成过程，避免过多的文本书写。
* 请制作一个**简要的PPT**，用于答辩时展示，涵盖以下内容：*思路+代码逻辑*，*心得体会*，*[Optional]效果展示(现场演示为佳)*。
* **不再**要求总结文档，但是建议你有时间写一下，它应当会成为一份很有意义的笔记。

## 关于rosbag

Ref：http://wiki.ros.org/rosbag/Commandline#rosbag_info

## 关于image_transport

试图通过以下简单的方式来订阅 */galaxy_camera/image_raw/compressed*，你会看到类似的Warning：
```
#include <ros/ros.h>
#include <image_transport/image_transport.h>

void imageCallback(const sensor_msgs::ImageConstPtr& msg)
{
  // ...
}
ros::NodeHandle nh;
image_transport::ImageTransport it(nh);
image_transport::Subscriber sub = it.subscribe("/galaxy_camera/image_raw/compressed", 1, imageCallback);
```

[ WARN] [1645977457.181674171]: [image_transport] It looks like you are trying to subscribe directly to a transport-specific image topic '/galaxy_camera/image_raw/compressed', in which case you will likely get a connection error. Try **subscribing to the base topic '/galaxy_camera/image_raw'** instead with parameter ~image_transport set to 'compressed' (**on the command line, _image_transport:=compressed**). See http://ros.org/wiki/image_transport for details.


* 我加粗的地方表明了解决方法：**rosrun &nbsp;&nbsp;xxxxx&nbsp;&nbsp; _image_transport:=compressed**
(同样这在论坛上也有个答案：https://answers.ros.org/question/11118/exporting-compressed-video/)
* https://cse.sc.edu/~jokane/teaching/574/notes-images.pdf 的第二页下半部分同样给出了细致的说明。
  (你可以用**一般的Subscriber**，但是这背离了image_transport设计的初衷)
* 你也许已经了解到了这个东西：[**compressed_image_transport**][1]，用它也可以。但是由于马上就要说的原因，你肯定想把它扔到一边。

### 怎么同时拿到图像和CameraInfo？

~~&nbsp;&nbsp;用两个Subscriber分别订阅&nbsp;&nbsp;~~(反正是可行的)

使用**image_transport**！在[4.2节][2]，看到这个了不：**image_transport::CameraSubscriber**

点进API链接，看看Member Typedef Documentation里**Callback**是长什么样子的。

因此，可以使用这样的方法：
```
class ImageConverter
{
public:
  ImageConverter(ros::NodeHandle& p_nh) : it_(p_nh)
  {
    cam_sub_ = it_.subscribeCamera("/galaxy_camera/image_raw", 1, &ImageConverter::onFrameCb, this);
  }

private:
  void onFrameCb(const sensor_msgs::ImageConstPtr& img, const sensor_msgs::CameraInfoConstPtr& info)
  {
    cv_image_ = cv_bridge::toCvCopy(img, "bgr8");
    cam_info_ = info;
  }

  image_transport::ImageTransport it_;
  image_transport::CameraSubscriber cam_sub_;
  static cv_bridge::CvImagePtr cv_image_;
  static sensor_msgs::CameraInfoConstPtr cam_info_;
};
```

但是这要求Image和CameraInfo是同步的。你可能看到(或者跑着跑着出现)以下Warning：

[ WARN ] [1645980834.506724356]: [image_transport] Topics '/galaxy_camera/image_raw/compressed' and '/galaxy_camera/camera_info' do not appear to be synchronized. In the last 10s:
	Image messages received:      1057
	CameraInfo messages received: 1041
	Synchronized pairs:           1
	

**增大queue_size**以解决：
```
cam_sub_ = it_.subscribeCamera("/galaxy_camera/image_raw", 10, &ImageConverter::onFrameCb, this);
```

### 有没有办法不加上 _image_transport:=compressed ？

~~&nbsp;&nbsp;rosrun  &nbsp;&nbsp; xxxxx&nbsp;&nbsp;   _image_transport:=compressed&nbsp;&nbsp;~~ “我不想要命令行，我想在CLion里点”。**可以的！** 

方法1(Ref：https://github.com/Ronan0912/ros_opentld/issues/5)：
```
// In order to select a specific transport, you have to add hints
cam_sub_ = it_.subscribeCamera("/galaxy_camera/image_raw", 10, &ImageConverter::onFrameCb, this, image_transport::TransportHints("compressed_image_transport", ros::TransportHints()));
```

方法2：
```
int main(int argc, char **argv)
{
  ros::init(argc, argv, "xxxxxx");
  ros::NodeHandle nh("~");
  nh.setParam("image_transport", "compressed");
  // ...your code...
  return 0;
}
```

  [1]: http://wiki.ros.org/compressed_image_transport
  [2]: http://wiki.ros.org/image_transport#image_transport_Subscribers
