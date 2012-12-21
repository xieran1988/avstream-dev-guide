
这是一个简单的程序，它打开了一个 mp4 文件并且解析文件中存在哪些流。找到 h264 流对应的解码器。

    #include <stdio.h>
    #include <libavcodec/avcodec.h>
    #include <libavformat/avformat.h>
    #include <libavutil/avutil.h>
    #include <libavutil/dict.h>
    #include <ctype.h>
    #include <stdlib.h>
    #include <unistd.h>
    
    int main() 
    {
      AVFormatContext *ifc;
    	char *input_filename = "1.mp4", *output_filename;
    	AVStream *st_h264 = NULL;
    	int i;
    
      // 在使用 libav 之前，必须使用这两句做初始化
    	av_log_set_level(AV_LOG_DEBUG);
    	av_register_all();
    
      // 使用 avformat_open_input 打开文件
    	printf("opening %s\n", input_filename);
    	i = avformat_open_input(&ifc, input_filename, NULL, NULL);
    	printf("open %d, nb_streams: %d\n", i, ifc->nb_streams);
    
      // 使用 avformat_find_stream_info 解析文件中有哪些 stream
    	avformat_find_stream_info(ifc, NULL);
    	for (i = 0; i < ifc->nb_streams; i++) {
    		AVStream *st = ifc->streams[i];
    		AVCodec *c = avcodec_find_decoder(st->codec->codec_id);
    		printf("%s\n", c->name);
    		if (!strcmp(c->name, "h264")) // h264 流的 codec name 为 "h264"
    			st_h264 = st;
    	}
    
      // 先 find_decoder 然后用 avcodec_open2 来找到解码器
    	AVCodec *c = avcodec_find_decoder(st_h264->codec->codec_id);
    	i = avcodec_open2(st_h264->codec, c, NULL);
    	printf("codec_open %d\n", i);
    }
    
代码编译之后的输出为
    opening 1.mp4
    open 0, nb_streams: 2
    h264
    aac
    codec_open 0
可见 1.mp4 中有两个流，一个是 aac 流，一个是 h264 流。目前流行的封装就是如此。
