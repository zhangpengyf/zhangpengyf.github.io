
###I420格式介绍
    在webrtc中android和ios系统采集摄像头获取到原始数据后，
    一帧原始图像会被转化为标准的YUV420P格式，也就是I420格式,
    转换的函数使用的是libyuv中的ConvertToI420()函数
###YUV格式详细讲解
    进行裁剪操作需要对I420格式的内存分布有深入的了解，推荐大家看这篇文章：
    http://blog.csdn.net/jefry_xdz/article/details/7931018 
    其中I420就是YUV420P格式

###裁剪图像的含义
	裁剪就是去掉某些区域，剩下指定的区域，
	本文所讲的裁剪就是抠图，抠去出原图像的某个内部区域
###裁剪的用途
	我们可以将裁剪后的视频传递给h264编码器，然后将输出保存成MP4文件（还需要加上声音），即可模仿Instagram软件中拍摄方形视频的功能。

###代码如下：
	代码中假设视频宽度和“Image Stride”相同
	什么是stride？ 参考这篇文章：
	https://msdn.microsoft.com/en-us/library/windows/desktop/aa473780(v=vs.85).aspx

	这里使用到了webrtc中的部分代码其中I420VideoFrame格式定义为
	https://code.google.com/p/webrtc/source/browse/trunk/webrtc/video_frame.h?r=8434
	所以dst_frame是这样生成的：
```
  dst_frame->CreateEmptyFrame(dst_width_, dst_height_,
                              dst_width_, (dst_width_ + 1) / 2,
                              (dst_width_ + 1) / 2);
```

```
int zp_getImageBlock(const I420VideoFrame& src_frame, I420VideoFrame& dst_frame,int start_w,int start_h)
    {

        //I420格式说明参考的这个博客 http://blog.csdn.net/jefry_xdz/article/details/7931018
        
        assert(start_w >= 0 && start_h >= 0);
        //start_w 开始点距离左边的距离
        //start_h 开始点距离上边的距离
        //起始点可以决定裁剪视频的哪个区域，不一定是中心区域
        
        int src_width = src_frame.width();
        int src_height = src_frame.height();
        int dst_width = dst_frame.width();
        int dst_height = dst_frame.height();
        
        if (start_w + dst_width > src_width ||
            start_h + dst_height > src_height) {
            //区域设置不合理，已经超出原始视频范围
            return -1;
        }
        if(dst_height > src_height || dst_width > src_width)
        {
            //只能缩小，不能扩大，扩大使用libyuv的scale函数
            return -1;
        }
        
        int w_cut = start_w;
        int h_cut = start_h;
        
        //切除的部分必须是偶数，因为每四个像素对应一个u,v
        assert(w_cut%2==0);
        assert(h_cut%2==0);
        
        const uint8* src_y = src_frame.buffer(kYPlane);
        uint8* dst_y = dst_frame.buffer(kYPlane);
        const uint8* src_u = src_frame.buffer(kUPlane);
        uint8* dst_u = dst_frame.buffer(kUPlane);
        const uint8* src_v = src_frame.buffer(kVPlane);
        uint8* dst_v = dst_frame.buffer(kVPlane);
        
        //取Y
        for (int row = 0; row < dst_height; ++row) {
            for (int col = 0; col < dst_width; ++col) {
                //计算目标区域的下标对应于源数据的哪个下标
                //目标区域的下表row*dst_width+col
                //(h_cut+row)*src_width+(w_cut+col)
                dst_y[row*dst_width+col] = src_y[(h_cut+row)*src_width+(w_cut+col)];
            }
        }
        
        //取U,V
        int k = 0;
        for (int row = 0; row < dst_height; row+=2) {
            for (int col = 0; col < dst_width; col+=2) {
                int old_index = (h_cut+row)*src_width/4+(w_cut+col)/2;
                dst_u[k] = src_u[old_index];
                dst_v[k] = src_v[old_index];
                k++;
            }
        }
        return 0;
    }
    
```
