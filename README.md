# 注意事项

- 本`FFmpeg`库版本为`4.3.2`。

- 如果是c++程序，需要在引入ffmpeg的头文件外面加上`extern "C"`，如下所示：

  ```c++
  extern "C"
  {
  #include "libavcodec/avcodec.h"
  #include "libavformat/avformat.h"
  }
  ```

后续会持续完善和更新...

