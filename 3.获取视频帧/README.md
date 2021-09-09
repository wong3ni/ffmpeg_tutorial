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

