# 提取h264

本文中的测试媒体文件下载地址：https://sample-videos.com/index.php#sample-mp4-video。

本例子从mp4文件中提取h264流并写入文件中(**不包含音频部分**)，可以使用`ffplay`直接进行播放(`ffplay -f h264 video.h264`)，代码如下。

```c
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
#include <stdio.h>

static void decoded(AVBSFContext *h264bsfc, AVPacket *packet);

int main()
{
    const char *filename = "SampleVideo.mp4";
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

    // 获取流信息
    if (avformat_find_stream_info(avformat_ctx, NULL) < 0)
    {
        printf("无法获取流信息\n");
        return -1;
    }

    AVCodecParameters *codec_parameters = avcodec_parameters_alloc();
    int video_index = -1;
    for (uint32_t i = 0; i < avformat_ctx->nb_streams; i++)
    {
        AVStream *stream = avformat_ctx->streams[i];
        if (stream->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
        {
            video_index = i;
            codec_parameters = avformat_ctx->streams[i]->codecpar;
            break;
        }
    }

    const AVBitStreamFilter *h264_stream_filter = av_bsf_get_by_name("h264_mp4toannexb");
    if (h264_stream_filter == NULL)
    {
        printf("没有找到h264_mp4toannexb\n");
        return -1;
    }
    AVBSFContext *h264bsfc = NULL;
    av_bsf_alloc(h264_stream_filter, &h264bsfc);
    h264bsfc->par_in = codec_parameters;
    av_bsf_init(h264bsfc);

    FILE *fp = fopen("video.h264", "wb");
    AVPacket *packet = av_packet_alloc();

    while (av_read_frame(avformat_ctx, packet) >= 0)
    {
        if (packet->stream_index == video_index)
        {
            decoded(h264bsfc, packet);
            // 写入到文件中
            if (packet->size)
            {
                fwrite(packet->data, packet->size, 1, fp);
            }
        }
        av_packet_unref(packet);
    }

    decoded(h264bsfc, NULL);
    if (packet->size)
    {
        fwrite(packet->data, packet->size, 1, fp);
    }

    fclose(fp);
    av_packet_free(&packet);
    avformat_free_context(avformat_ctx);

    return 0;
}

static void decoded(AVBSFContext *h264bsfc, AVPacket *packet)
{
    // 发送packet让你添加额外信息
    int ret = av_bsf_send_packet(h264bsfc, packet);
    if (ret < 0)
    {
        printf("av_bsf_send_packet失败\n");
        exit(-1);
    }
    while (ret >= 0)
    {
        // 接收已经添加好的packet
        ret = av_bsf_receive_packet(h264bsfc, packet);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            return;
        else if (ret < 0)
        {
            exit(-1);
        }
    }
}
```

## 关键代码

```c
...
const AVBitStreamFilter *h264_stream_filter = av_bsf_get_by_name("h264_mp4toannexb");
if (h264_stream_filter == NULL)
{
    printf("没有找到h264_mp4toannexb\n");
    return -1;
}
AVBSFContext *h264bsfc = NULL;
av_bsf_alloc(h264_stream_filter, &h264bsfc);
h264bsfc->par_in = codec_parameters;
av_bsf_init(h264bsfc);
...

static void decoded(AVBSFContext *h264bsfc, AVPacket *packet)
{
    // 发送packet让你添加额外信息
    int ret = av_bsf_send_packet(h264bsfc, packet);
    if (ret < 0)
    {
        printf("av_bsf_send_packet失败\n");
        exit(-1);
    }
    while (ret >= 0)
    {
        // 接收已经添加好的packet
        ret = av_bsf_receive_packet(h264bsfc, packet);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            return;
        else if (ret < 0)
        {
            exit(-1);
        }
    }
}
```

现在网上大部分代码都是很老的版本使用方法了，比如使用`av_bitstream_filter_init`或`av_bitstream_filter_filter`之类的函数，这些函数已经被标记废弃，所以不再使用这类函数。上面函数参数、用法和含义可在ffmpeg官方文档中查阅，**注意不要选择默认最新的文档(因为可能又有变动)，选择4.1版本即可**，传送门：https://ffmpeg.org/doxygen/4.1/index.html。

## 参考文献

https://ffmpeg.org/doxygen/4.1/group__lavc__misc.html#ga8aa30391ba53516321c69ec0cea34e39

https://blog.csdn.net/m0_37346206/article/details/94029945

https://blog.csdn.net/gavinr/article/details/7183499