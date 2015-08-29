# Player
player demo cantain mpmovieplayercontroller

FFMPEG的使用已存放在Camera的帮助文档里


##使用ffmpeg步骤：http://www.cnblogs.com/bandy/archive/2013/02/19/2916641.html
```
av_register_all();//初始化ffmpeg库，如果系统里面的ffmpeg没配置好这里会出错
if (isNetwork) {//需要播放网络视频
    avformat_network_init();
}
avformat_open_input();//打开视频文件，如果使用的ffmpeg编译库太旧该方法将会始终返回非0值，即失败。
avformat_find_stream_info();//查找文件的流信息
av_dump_format();//dump只是个调试函数，输出文件的音、视频流的基本信息了，帧率、分辨率、音频采样等等
for(...);//遍历文件的各个流，找到第一个视频流，并记录该流的编码信息
sws_getContext();//根据编码信息设置渲染格式
avcodec_find_decoder();//在库里面查找支持该格式的解码器
avcodec_open2();//打开解码器
pFrame=avcodec_alloc_frame();//分配一个帧指针，指向解码后的原始帧
pFrameRGB=avcodec_alloc_frame();//分配一个帧指针，指向存放转换成RGB后的帧
avpicture_fill(pFrameRGB);//给pFrameRGB帧加上分配的内存;
while(true)
{
    av_read_frame();//读取一个帧（到最后帧则break）
    avcodec_decode_video2();//解码该帧
    sws_getCachedContext()sws_scale（）;//把该帧转换（渲染）成RGB
    SaveFrame();//对前5帧保存成ppm图形文件(这个是自定义函数，非API)
    av_free_packet();//释放本次读取的帧内存
}
```

##可以正常使用的ffmpeg的编译库来源：
可用：通过FFmpeg在iOS上完美编译（http://www.cocoachina.com/ios/20150514/11827.html）上下载的的demo(FFmpegTest)中的编译库FFmpeg-iOS可用
不可用：通过pod 'FFmpeg', '~> 2.2' 下载的ffmpeg-ios-static-libs不可用

##编译FFmpeg库的一些参数学习了解：
(摘自：iOS: FFmpeg编译和使用问题总结http://www.cnblogs.com/smileEvday/archive/2013/11/21/ffmpeg.html)

1、裁剪(--disable-xxx 或者 --enable-xxx)
我们知道FFmpeg库是一个非常庞大的库，包括编码，解码以及流媒体的支持等，如果不做裁剪全部编译进来的话，最后生成的静态库会很大。实际使用中我们可能只想用到解码(例如播放器)，因此我们可以使用相关选项指定编译时禁用编码部分。当然我们还可以做进一步的裁剪，例如只打开部分常用格式的解码，禁用掉其他的解码，这样编译出来的静态库将会更小。
```
　　要想裁剪，我们的先知道有哪些部分，使用下面的命令可以查看FFMpeg库支持的组件列表。
--list-decoders          show all available decoders
--list-encoders          show all available encoders
--list-hwaccels          show all available hardware accelerators
--list-muxers            show all available muxers
--list-demuxers          show all available demuxers
--list-parsers           show all available parsers
--list-protocols         show all available protocols
--list-bsfs              show all available bitstream filters
--list-indevs            show all available input devices
--list-outdevs           show all available output devices
--list-filters           show all available filters

　　我们可以根据实际需要把不用的部分都禁用掉，这样编译快，包也会比较小，常用的裁剪选项如下:
--disable-doc            do not build documentation
--disable-ffmpeg         disable ffmpeg build
--disable-ffplay         disable ffplay build
--disable-ffserver       disable ffserver build
--disable-network        disable network support [no]
--disable-encoder=NAME   disable encoder NAME
--enable-encoder=NAME    enable encoder NAME
--disable-encoders       disable all encoders
--disable-decoder=NAME   disable decoder NAME
--enable-decoder=NAME    enable decoder NAME
--disable-decoders       disable all decoders
--disable-hwaccel=NAME   disable hwaccel
　　举个例子，如果我们需要做一款本地视频播放器，那么我们可以使用如下配置:
./configure --disable-doc       \
            --disable-ffmpeg    \
            --disable-ffserver  \
            --disable-networt   \
            --disable-encoders
当然你还可以根据帮助列表进行更细粒度的裁剪，例如只支持哪几种格式的解码等等。
```

2、指定编译环境
FFMpeg作为一个跨平台的库，不同的平台，不同的人的计算机上编译器的路径都可能不尽相同，所以我们需要为编译脚本指定编译器的路径。同事我们还可以指定其他编译选项，如是否交叉编译，目标平台系统，CPU架构，需要依赖的其他库的路径已经指定是否禁用汇编优化等。
```
--enable-cross-compile   assume a cross-compiler is used//是否交叉编译，enable允许
--sysroot=PATH           root of cross-build tree       //sysroot即iOS SDK的路径，注意编译真机版本的库时需要使用iPhoneOS.platform中SDK的路径，编译模拟器版本的库使用iPhoneSimulator.platform中SDK的路径。
--sysinclude=PATH        location of cross-build system headers
--target-os=OS           compiler targets OS []         //target-os填写darwin(苹果系统的内核)，
--cc=CC                  use C compiler CC [gcc]
--extra-cflags=ECFLAGS   add ECFLAGS to CFLAGS []
--extra-ldflags=ELDFLAGS add ELDFLAGS to LDFLAGS []
--arch=ARCH              select architecture []         //arch可以根据具体的情况添加i386(模拟器)，armv6，armv7等。
--cpu=CPU                select the minimum required CPU (affects
instruction selection, may crash on older CPUs)         //cpu根据具体类型可填写cortex-a8，cortox-a9，i386等。
--disable-asm            disable all assembler optimizations//禁用所有的汇编优化
```



问：怎么在打开流媒体的时候传一些附加参数？
即：首先我们知道使用ffmpeg类库打开流媒体（或本地文件）的函数是avformat_open_input()。一般情况下，打开媒体只要传入流媒体的url就可以了。但是在打开某些流媒体的时候，我们可能还需要附加一些参数。这时候我们怎么在打开流媒体的时候传一些附加参数？
答：FFMPEG类库打开流媒体的方法（需要传参数的时候） http://blog.csdn.net/leixiaohua1020/article/details/14215393
即：我们不难发现avformat_open_input()的第4个参数是一个AVDictionary类型的参数，而这个参数就是传入的附加参数。具体的使用如下：
```
命令：ffplay -rtsp_transport tcp -max_delay 5000000 rtsp://mms.cnr.cn/cnr003?MzE5MTg0IzEjIzI5NjgwOQ== 

转为代码：
AVFormatContext	*pFormatCtx;
pFormatCtx = avformat_alloc_context();
...代码略
AVDictionary *avdic=NULL;
char option_key[]="rtsp_transport";
char option_value[]="tcp";
av_dict_set(&avdic,option_key,option_value,0);
char option_key2[]="max_delay";
char option_value2[]="5000000";
av_dict_set(&avdic,option_key2,option_value2,0);
char url[]="rtsp://mms.cnr.cn/cnr003?MzE5MTg0IzEjIzI5NjgwOQ==";
avformat_open_input(&pFormatCtx,url,NULL,&avdic);
av_dict_free(&avdic);//别忘了最后还要av_dict_free(&avdic);
```


