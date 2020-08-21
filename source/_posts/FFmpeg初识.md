---
title: FFmpeg初识
date: 2020-08-21 08:02:26
tags:
---
[TOC]

## 1. 制作ScreenMaps

[原文地址](http://dranger.com/ffmpeg/tutorial01.html)

文件本身是个容器

不同容器的种类决定了文件内的信息

常见的容器格式有AVI、Quicktime

在一个容器内，我们可以封装很多的流

常见的，我们都会有一个音频流和一个视频流

流内的元素我们称之为序列

每一条流都有其特定的编码格式，对应特定的codec编码解码器

codec决定了实际的数据如何被COded，如何被DECoded，这也是其名字CODEC的由来

常见的codec有DivX以及MP3

包Packets 是每次从流中读取得到的数据块，其内包含有我们最终需要输入应用中的信息

常见的视频流处理流程：

```
1. 从video.avi容器内打开video视频流
2. 从video视频流中源源不断的读取packet到frame内
3. 如果frame是不完整的，那么继续回到2
4. 对frame进行相应的处理
5. 返回2
```

```c++
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <ffmpeg/swscale.h>

int main(int argc, char* argv[])
{
    av_register_all();
}
```

av_register_all()将会在程序运行时对所有可用的codec以及format进行登记，一般来说，没有必要指定登记某一种codec或者format，且由于只需要登记一次，
我们也只需要在main()函数中调用一次即可

```c++
AVFormatContext* pFormatCtx = NULL;
// Open Video file
if(avformat_open_input(&pFormatCtx, argv[1], NULL, 0, NULL)!=0)
    return -1; // Return -1 if open input failed
```

avformat_open_input()函数将读取文件的头，并在AVFormatContext结构体变量pFormatCtx中存储相应的文件信息，其后三个参数分别对应的是 文件格式，缓冲
区大小以及格式选项，分别设置为NULL或者0可以令libavformat库自动检测这些

```c++
// retrieve stream information
if(avformat_find_stream_info(pFormatCtx, NULL) < 0)
    return -1; // Return -1 if find stream information failed
```

avformat_find_stream_info()函数将检查pFormatCtx变量内的流信息

```c++
// dump infomation about file onto standard error
av_dump_format(pFormatCtx, 0, argv[1], 0);
```

pFormatCtx->streams只是一个指针数组，其大小为pFormatCtx->nb_streams，遍历该指针数组直到我们找到一个videoStream
```c++
int i;
AVCodecContext* pCodecCtxOrig = NULL;
AVCodecContext* pCodecCtx = NULL;

// Find the first video stream
int videoStream = -1;
for(i=0; i < pFormatCtx->nb_streams; i++)
{
    if(pFormatCtx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO)
    {
        videoStream = i;
        break;
    }
}
if(videoStream == -1)
{
    return -1; // Failed to find a video stream
}

// get a pointer to the codec context for the video stream
pCodecCtx = pFormatCtx->streams[videoStream]->codec;
```

流内有关codec编解码器的信息我们称之为'codec context'编解码上下文，它包含了当前流使用的codec所有的信息，然后我们用一个指针指向它

```c++
AVCodec* pCodec = NULL;
// Find the decoder for the video stream
pCodec = avcodec_find_decoder(pCodecCtx->codec_id);
if(pCodec == NULL)
{
    fprintf(stderr, "Unsupported codec!\n");
    return -1;
}

// Copy Context
pCodecCtx = avcodec_alloc_context3(pCodec);
// avcodec_copy_context(target, src)，因为不能直接使用视频流内的codec上下文，我们需要将其拷贝出来到我们刚初始化的pCodecCtx 这个codec上下文中
if(avcodec_copy_context(pCodecCtx, pCodecCtxOrig)!=0)
{
    fprintf(stderr, "Couldn't copy codec context");
    return -1;  // Error while copying codec context
}

// Open codec
if(avcodec_open2(pCodecCtx, pCodec) < 0)
{
    return -1;
}
```

我们再定义一段内存用来存放packet流内的frame信息

```c++
AVFrame* pFrame = NULL;
// Allocate video frame;
pFrame = av_frame_alloc();
```

```c++
// Allocate an AVFrame structure
pFrameRGB = av_frame_alloc();
if(pFrameRGB == NULL)
    return -1;
```

```c++
uint8_t *buffer = NULL;

int numBytes;
// Determine required buffer size and allocate buffer
// 24 bits(3 bytes) x width x height
numBytes = avpicture_get_size(PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height);

buffer = (uint8_t*)av_malloc(numBytes * sizeof(uint8_t));
```

此处的av_malloc是原malloc的一个简单包装器，它只能确保内存地址是对齐的，但它仍无法防止一些内存泄露，double free以及其它的malloc常见问题

接下来要用avpicture_fill来将帧填充到新分配的缓冲区中，AVPicture是AVFrame结构体的子集

```c++
// Assign appropriate parts of buffer to image planesin pFrameRGB
// Note that pFrameRGB is an AVFrame, but AVFrame is a superset of AVPicture
avpicture_fill((AVPicture *) pFrameRGB, buffer, PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height);
```

**读取数据**

```c++
struct SwsContext *sws_ctx = NULL;
int frameFinished;
AVPacket packet;
// initialize SWS context for software scaling
sws_ctx = sws_getContext(pCodecCtx->width,
                                pCodecCtx->height,
                                pCodecCtx->pix_fmt,
                                pCodecCtx->weight,
                                pCodecCtx->height,
                                PIX_FMT_RGB24,
                                SWS_BILINEAR,
                                NULL,
                                NULL,
                                NULL);

i=0;
while(av_read_frame(pFormatCtx, &packet) >= 0)
{
    // 判定该packet是否是video stream的packet
    if(packet.stream_index == videoStream)
    {
        // Decode video frame
        avcodec_decode_video2(pCodecCtx, pFrame, &frameFinished, &packet);

        // Did we get a video frame?
        if(frameFinished)
        {
            // Convert the image from its native format to RGB
            sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data, pFrame->linesize, 0, pCodecCtx->height, pFrameRGB->data, pFrameRGB->linesize);

            // save the frame to disk
            if(++i <= 5)
            {
                SaveFrame(pFrameRGB, pCodecCtx->width, pCodecCtx->height, i);
            }
        }
    }
    // Free the packet that was allocated by av_read_frame
    av_free_packet(&packet);
}
```

SaveFrame()函数是我们定义的图像写入函数，下面将帧写入到ppm文件中，有关ppm格式的相关介绍可参考:arrow_right:

```c++
void SaveFrame(AVFrame *pFrame, int width, int height, int iFrame)
{
    FILE *pFile;
    char szFileName[32];
    int y;

    // Open file
    sprintf(szFileName, "frame%d.ppm", iFrame);
    pFile = fopen(szFileName, "wb");
    if(pFile == NULL)
            return;

    // write header
    fprintf(pFile, "P6\n%d %d\n255\n", width, height);

    // write pixel data
    for(y=0; y<height; y++)
    {
        fwrite(pFrame->data[0] + y*pFrame->linesize[0], 1, width*3, pFile);
    }

    // close file
    fclose(pFile);
}
```

PPM图像就是一个将原有RGB信息以字符串形式平铺开来的文件，其头文件会包含图像的宽和高大小，以及RGB值的最大值。当完成video stream的读取操作后，清
除everything

```c++
// Free the RGB image
av_free(buffer);
av_free(pFrameRGB);
// Free the YUV frame
av_free(pFrame);

// Close the codecs
avcodec_close(pCodecCtx);
avcodec_close(pCodecCtxOrig);

// Close the video file
avformat_close_input(&pFormatCtx);

return 0;
```

下面，上gcc编译命令:

```shell
gcc tutorial1.c -o tutorial -lavutil -lavformat -lavcodec -lz -lm
```

## 2. 投影到屏幕上 Output to the screen

为了将pixel绘制(draw)到屏幕上，我们需要用到SDL库（现在用的是SDL2.0库）

SDL就是Simple Direct Layer，是一个跨平台的多媒体库

YUV是一种存储raw image数据的方式，Y是brightness亮度组件，U和V是颜色组件。SDL库中有一种叫做YUV overlay的方法可以将movies放映到screen上，该方法
仅需输入YUV data作为raw数组即可display它，它接受4种YUV形式，其中YV12是最快的，YUV420P与YV12速度差不多，只是U和V的顺序被替换了。YUV420P中420指
代下采样的比例是4:2:0



首先，我们需要初始化SDL，SDL_Init()函数会告诉函数库将会使用哪些特性:

```c
if(SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER))
{
    fprintf(stderr, "Could not initialize SDL - %s\n", SDL_GetError());
    exit(1);
}
```

然后，我们需要在屏幕上定义一块区域来放置我们的stuff，SDL1.0库中用的是SDL_Surface，SDL2.0库中用的是SDL_Window：

```c
SDL_Window *screen;
screen = SDL_CreateWindow("FFmpeg play",
                                    SDL_WINDOWPOS_UNDEFINED,
                                    SDL_WINDOWPOS_UNDEFINED,
                                    pCodecCtx->width,
                                    pCodecCtx->height,
                                    0);
if(!screen)
{
    fprintf(stderr, "SDL: Could not create window!\n");
    exit(1);
}
```

