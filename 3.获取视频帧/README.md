# 获取视频帧

下面举一个打开一个媒体文件的例子来学习一下ffmpeg的日常操作。

```c
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
#include <stdio.h>

int main()
{
    const char *filename = "SampleVideo.mp4";
  	// 分配一个avformat context
    AVFormatContext *avformat_ctx = avformat_alloc_context();
    if (avformat_ctx == NULL)
    {
        printf("分配avformat cotext失败\n");
        return -1;
    }

    // 打开输入流，读取其头部
    // 使用avformat_close_input来关闭输入流
    // 此时解码器还没有打开
    if (avformat_open_input(&avformat_ctx, filename, NULL, NULL) < 0)
    {
        printf("打开媒体文件失败\n");
        return -1;
    }

    // 探测流信息，例如图像的像素格式
    if (avformat_find_stream_info(avformat_ctx, NULL) < 0)
    {
        printf("无法获取流信息\n");
        return -1;
    }

    AVCodec *codec = NULL;
    AVCodecContext *codec_context;
    AVCodecParameters *codec_parameters = avcodec_parameters_alloc();
    int video_index = -1;

    for (uint32_t i = 0; i < avformat_ctx->nb_streams; i++)
    {
        AVStream *stream = avformat_ctx->streams[i];
        printf("=====\n编码类型: %d\n流类型: %d\n像素格式: %d\n比特率:%lld\n",
               stream->codecpar->codec_id,
               stream->codecpar->codec_type,
               stream->codecpar->format,
               stream->codecpar->bit_rate);
        if (stream->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
        {
            video_index = i;
            if (avformat_ctx->streams[i]->codecpar)
            {
                avcodec_parameters_free(&codec_parameters);
            }
            codec_parameters = avformat_ctx->streams[i]->codecpar;
            break;
        }
    }
		
    codec = avcodec_find_decoder(codec_parameters->codec_id);
    codec_context = avcodec_alloc_context3(codec);
    avcodec_parameters_to_context(codec_context, codec_parameters);

    // 打开解码器
    avcodec_open2(codec_context, codec, NULL);

    AVPacket *packet = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
		
  	// av_read_frame读取每一帧压缩数据
    while (av_read_frame(avformat_ctx, packet) >= 0)
    {
      	// 如果是视频数据
        if (packet->stream_index == video_index)
        {
          	// 送入进行解码
            int ret = avcodec_send_packet(codec_context, packet);
            if (ret < 0)
            {
                printf("解码数据包失败\n");
                return -1;
            }
            while (ret >= 0)
            {
              	// 读取解码后的数据，即frame存放原始数据
                ret = avcodec_receive_frame(codec_context, frame);
                if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
                    continue;
                else if (ret < 0)
                {
                    printf("解码数据时出错\n");
                    return -1;
                }
                printf("解码第%d帧\n", codec_context->frame_number);
            }
        }
      	// 释放引用的数据，重用frame和packet
        av_frame_unref(frame);
        av_packet_unref(packet);
    }
	
  	// 内存释放
    av_frame_free(&frame);
    av_packet_free(&packet);
    avcodec_free_context(&codec_context);
    avformat_free_context(avformat_ctx);

    return 0;
}

```

## 主要步骤

1. `avformat_alloc_context()`

   创建一个`AVFormatContext`对象。几乎所有的程序中都会创建这一个对象，不管是读取数据还是写入数据，都需要创建这一个对象。

2. `avformat_open_input()`

   打开一个媒体文件。

3. `avformat_find_stream_info()`

   探测有关流的信息，此函数可以省略，但是如果想提前获取更多关于流中的信息，就需要调用此函数。

4. 获取有关编解码的参数。

```c++
// nb_streams存放了流的数量，大部分情况下是2，一个是视频流，另一个是音频流
for (uint32_t i = 0; i < avformat_ctx->nb_streams; i++)
{
  	// 获取索引为i的流
    AVStream *stream = avformat_ctx->streams[i];
  	// 判断是否为视频流
    if (stream->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
    {
      	// 保存相关视频流信息
        video_index = i;
				// 最近的版本需要使用AVCodecParameters对象，利用AVCodecParameters对象来填充AVCodecContext对象中的一些信息
      	// 注意codec_parameters需要提前使用avcodec_parameters_alloc()函数分配空间
        codec_parameters = avformat_ctx->streams[i]->codecpar;
        break;
    }
}
```

5. 对编解码对象(`codec`)和编解码上下文(`codec_context`)赋值

```c++
// 根据codec_parameters中的编解码id寻找合适的编解码对象
// codec不需用提前分配空间
codec = avcodec_find_decoder(codec_parameters->codec_id);
// 必须提前分配空间，否则会报错
codec_context = avcodec_alloc_context3(codec);
// 从codec_parameters参数中填充codec_context对象
avcodec_parameters_to_context(codec_context, codec_parameters);
```

6. `avcodec_open2()`

   打开编解码。

7. 循环读取数据，获取其中的帧数据并进行解码操作。

```c++
// av_read_frame读取每一帧压缩数据到packet中
while (av_read_frame(avformat_ctx, packet) >= 0)
{
    // 如果是视频数据
    if (packet->stream_index == video_index)
    {
        // 将packet送入进行解码
        int ret = avcodec_send_packet(codec_context, packet);
        if (ret < 0)
        {
            printf("解码数据包失败\n");
            return -1;
        }
      	// 不使用循环可能会造成内存泄漏
        while (ret >= 0)
        {
            // 读取解码后的数据，即frame存放原始数据
            ret = avcodec_receive_frame(codec_context, frame);
            if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
              continue;
            else if (ret < 0)
            {
              printf("解码数据时出错\n");
              return -1;
            }
            printf("解码第%d帧\n", codec_context->frame_number);
        }
    }
    // 释放引用的数据，重用frame和packet。此操作并不会释放frame和packet结构，只释放它们引用的数据
    av_frame_unref(frame);
    av_packet_unref(packet);
}
```

## AVPacket结构体

这里只简单介绍几个基本的参数


```c++
typedef struct AVPacket {
		// 显示时间戳
		int64_t pts;
  	// 解码时间戳
		int64_t dts;
  	// 压缩数据的指针
		uint8_t *data;
  	// 压缩数据的大小
		int size;
		...
} AVPacket;
```

## AVFrame结构体

```c++
typedef struct AVFrame {
  	// 原始数据的指针，即解码后的数据，对于rgb类的数据而言data[0]就是指向的原始数据，对于yuv类的数据而言data[0],data[1],data[2]分别指向y,u,v三个分量的数据，AV_NUM_DATA_POINTERS为8
  	uint8_t *data[AV_NUM_DATA_POINTERS];
  	// 对于视频数据而言，对应上面data中每个分量的长度，注意这个值可能会因cpu对齐要求而超出上面实际内容的长度
  	int linesize [AV_NUM_DATA_POINTERS];
  	// 像素格式
  	int format;
  	// 是否为关键帧
  	int key_frame;
  	// 压缩的数据大小，即AVPacket中的数据大小
  	int pkt_size;
  	...
} AVFrame;
```

`AVPacket`结构体中的具体内容参考地址：https://ffmpeg.org/doxygen/4.1/structAVPacket.html

`AVFrame`结构体中的具体内容参考地址：https://ffmpeg.org/doxygen/4.1/structAVFrame.html
