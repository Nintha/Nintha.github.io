---
title: 从图片流生成直播视频流
date: 2018-11-21
categories: opencv
---

## 前言

很多时候我们获取到了一些实时的二进制图片流（如摄像头），并希望可以在浏览器中实时查看，类似直播那样。



## 准备

我们需要javacv，它封装了opencv，ffpmeg等一系列工具

build.gradle

```groovy
dependencies {
    compile group: 'org.bytedeco', name: 'javacv-platform', version: '1.4.3'
}
```

思路是这样的，我们用javacv把二进制图片流转换成rmtp视频直播流，推送到媒体服务器，然后用客户端从媒体服务器获取直播流。

这里我使用的是livego，因为它使用起来比较简单，上行支持rmtp，下行支持rmtp/flv/hls。

rmtp格式可以使用`VLC media player`播放器进行播放

flv格式可以配合`flv.js`在浏览器中进行播放或使用`VLC media player`播放器进行播放

hls格式可以使用`VLC media player`播放器进行播放



<!--more-->

## Code

代码用kotlin写的，使用时调用`transferRtmp`方法

```kotlin
fun imageBytesToBufferedImage(data: ByteArray): BufferedImage {
    return ImageIO.read(ByteArrayInputStream(data))
}

fun bufferedImageToIplImage(bufImage: BufferedImage): opencv_core.IplImage {
    val iplConverter = OpenCVFrameConverter.ToIplImage()
    val java2dConverter = Java2DFrameConverter()
    return iplConverter.convert(java2dConverter.convert(bufImage))
}

fun imageBytesToIplImage(data: ByteArray): opencv_core.IplImage {
    return bufferedImageToIplImage(imageBytesToBufferedImage(data))
}
/**
  * 参考 https://blog.csdn.net/redfoxtao/article/details/78080924
  * imagesPath: 图片所在文件夹, eg:'./jpgs/'
  * outRtmpUrl：rtmp推流地址，eg:'rtmp://localhost:1935/live/movie'
  */
fun transferRtmp(imagesPath: String, outRtmpUrl: String) {
    val frameRate = 25 //码率
    val recorder = FrameRecorder.createDefault(outRtmpUrl, 600, 480)
    recorder.videoCodec = avcodec.AV_CODEC_ID_H264
    recorder.format = "flv"
    recorder.frameRate = frameRate.toDouble()
    recorder.gopSize = 5
    // set audio
    recorder.setAudioOption("crf", "0")
    recorder.audioQuality = 0.0
    recorder.audioBitrate = 192000
    recorder.sampleRate = 44100
    recorder.audioChannels = 0
    recorder.audioCodec = avcodec.AV_CODEC_ID_AAC

    println("start pushing stream to $outRtmpUrl")
    recorder.start()

    val converter = OpenCVFrameConverter.ToIplImage()
    val fileList = Paths.get(imagesPath).toFile().listFiles().sortedBy { it.name }
    var startTime: Long? = null
    val durationTime: Long = 1000L / frameRate
    var loopId: Long = 0L
    println("durationTime=$durationTime")
    while (true) {
        try {
            // 循环目录中所有的图片
            for (file in fileList) {
                val bytes = file.readBytes()
                val image = imageBytesToIplImage(bytes)

                val now = System.currentTimeMillis()
                if (startTime != null) {
                    println("now=$now, startTime=$startTime, rst=${recorder.timestamp / 1000}")
                    val delta = now - startTime - recorder.timestamp / 1000
                    if (durationTime - delta > 0) {
                        println("sleep ${durationTime - delta} ms")
                        Thread.sleep(durationTime - delta)
                    } else {
                        println("[WARN] The frame delta time is $delta, frameRate is $frameRate")
                    }
                }

                startTime = startTime ?: now
                val videoTime = 1000 * (now - startTime)
                println("videoTime >\t$videoTime \n\n")
                if (videoTime > recorder.timestamp) recorder.timestamp = videoTime

                recorder.record(converter.convert(image))
            }
            println("Loop#$loopId\t${recorder.timestamp}")
            loopId++
        } catch (e: Exception) {
            System.err.println("ERROR!!!!")
            e.printStackTrace()
            break
        }
    }
    recorder.stop();
    recorder.release();
}

```



