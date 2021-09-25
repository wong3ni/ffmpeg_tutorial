# 从mp4中提取YUV

本文中的测试媒体文件下载地址：https://sample-videos.com/index.php#sample-mp4-video。

本例子从mp4文件中提取yuv并写入文件中(**不包含音频部分**)，可以使用`ffplay`直接进行播放(`ffplay -f rawvideo -video_size 1280x720 out.yuv`)，代码如下

```c++
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
#include <stdio.h>

void decode(AVCodecContext *avcc, AVFrame *frame, AVPacket *packet, int y_size, FILE *fp)
{
    // 送入解码
    int ret = avcodec_send_packet(avcc, packet);
    if (ret < 0)
    {
        printf("Sent packet fail!\n");
        exit(-1);
    }
    while (ret >= 0)
    {
        // 读取解码后的数据，即frame存放原始数据
        ret = avcodec_receive_frame(avcc, frame);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            return;
        else if (ret < 0)
        {
            printf("Decoded fail!\n");
            exit(-1);
        }
        if (frame->format != AV_PIX_FMT_YUV420P)
        {
            printf("Image pixel format is not yuv420P!\n");
            exit(-1);
        }
        fwrite(frame->data[0], 1, y_size, fp);
        fwrite(frame->data[1], 1, y_size / 4, fp);
        fwrite(frame->data[2], 1, y_size / 4, fp);
        printf("Wirte the %d frame!\n", avcc->frame_number);
    }
}

int main()
{
    const char *filename = "test.mp4";
    const char *outfile = "out.yuv";
    FILE *fp;
    AVFormatContext *avf_ctx = avformat_alloc_context();
    if (avformat_open_input(&avf_ctx, filename, NULL, NULL) < 0)
    {
        printf("Open input fail!\n");
        exit(-1);
    }
    avformat_find_stream_info(avf_ctx, NULL);
    // 寻找视频流
    int video_index = -1;
    for (uint32_t i = 0; i < avf_ctx->nb_streams; i++)
    {
        if (avf_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
        {
            video_index = i;
            break;
        }
    }
    if (video_index == -1)
    {
        printf("Can not found video info!\n");
        exit(-1);
    }

    AVCodecParameters *codec_params = avf_ctx->streams[video_index]->codecpar;
    AVCodec *codec = avcodec_find_decoder(codec_params->codec_id);
    AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
    avcodec_parameters_to_context(codec_ctx, codec_params);

    if (avcodec_open2(codec_ctx, codec, NULL) < 0)
    {
        printf("Can not open the codec!\n");
        exit(-1);
    }

    AVPacket *packet = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();

    fp = fopen(outfile, "wb");
    // 这里的y是指yuv中y的分量大小
    int y_size = codec_ctx->width * codec_ctx->height;

    while (av_read_frame(avf_ctx, packet) >= 0)
    {
        // 如果是视频数据
        if (packet->stream_index == video_index)
        {
            // 解码数据
            decode(codec_ctx, frame, packet, y_size, fp);
        }
        // 释放引用的数据，重用frame和packet
        av_frame_unref(frame);
        av_packet_unref(packet);
    }

    // 输出缓存的数据
    decode(codec_ctx, frame, NULL, y_size, fp);

    // 内存释放
    av_frame_free(&frame);
    av_packet_free(&packet);
    avformat_free_context(avf_ctx);
    fclose(fp);

    return 0;
}

```

## 参考文献

https://blog.csdn.net/hsq1596753614/article/details/81749576

https://blog.csdn.net/hsq1596753614/article/details/81783094
