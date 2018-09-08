---
title: Java处理Webp图片格式转换
date: 2018-09-09

---

## 前言

Webp是Google推出的一种新型图片格式，相比于 传统的PNG/JPG图片有着更小体积的优势，在Web中有着广泛的应用。由于Webp格式推出比较晚， Jdk 内置的图片编解码库对此并不支持。

网上给出的Java环境解决方案往往需要手动在`java.library.path`中安装对应的动态链接库，windows是dll文件，linux是so文件。这对于开发部署非常不方便。

本文提供一种无需手动安装动态链接库，同时可以方便处理Webp的解决方案

<!--more-->

## 准备

先从github上面下载所需要的jar包

[webp-imageio-core-0.1.0.jar](https://github.com/nintha/webp-imageio-core/releases)

由于这个项目并未发布到maven中央仓库，所以需要手动导入本地jar包.

如果你用的是gradle，可以把jar包放入`src/main/resource/libs`目录，并在`build.gradle`中加入依赖

```groovy
dependencies {
    compile fileTree(dir:'src/main/resources/libs',include:['*.jar'])
}
```

如果你用的是maven，可以把jar包放入`${project.basedir}/libs`目录，并在`pom.xml`中加入依赖

```
<dependency>  
    <groupId>com.github.nintha</groupId>  
    <artifactId>webp-imageio-core</artifactId>  
    <version>{versoin}</version>  
    <scope>system</scope>  
    <systemPath>${project.basedir}/libs/webp-imageio-core-{version}.jar</systemPath>  
</dependency>
```

## 例子

完整代码见 https://github.com/nintha/webp-imageio-core，以下为部分摘录

Webp编码

```java
public static void main(String args[]) throws IOException {
    String inputPngPath = "test_pic/test.png";
    String inputJpgPath = "test_pic/test.jpg";
    String outputWebpPath = "test_pic/test_.webp";

    // Obtain an image to encode from somewhere
    BufferedImage image = ImageIO.read(new File(inputJpgPath));

    // Obtain a WebP ImageWriter instance
    ImageWriter writer = ImageIO.getImageWritersByMIMEType("image/webp").next();

    // Configure encoding parameters
    WebPWriteParam writeParam = new WebPWriteParam(writer.getLocale());
    writeParam.setCompressionMode(WebPWriteParam.MODE_DEFAULT);

    // Configure the output on the ImageWriter
    writer.setOutput(new FileImageOutputStream(new File(outputWebpPath)));

    // Encode
    writer.write(null, new IIOImage(image, null, null), writeParam);
}
```

Webp解码

```java
public static void main(String args[]) throws IOException {
    String inputWebpPath = "test_pic/test.webp";
    String outputJpgPath = "test_pic/test_.jpg";
    String outputJpegPath = "test_pic/test_.jpeg";
    String outputPngPath = "test_pic/test_.png";

    // Obtain a WebP ImageReader instance
    ImageReader reader = ImageIO.getImageReadersByMIMEType("image/webp").next();

    // Configure decoding parameters
    WebPReadParam readParam = new WebPReadParam();
    readParam.setBypassFiltering(true);

    // Configure the input on the ImageReader
    reader.setInput(new FileImageInputStream(new File(inputWebpPath)));

    // Decode the image
    BufferedImage image = reader.read(0, readParam);

    ImageIO.write(image, "png", new File(outputPngPath));
    ImageIO.write(image, "jpg", new File(outputJpgPath));
    ImageIO.write(image, "jpeg", new File(outputJpegPath));

}
```

## 关于webp-imageio-core项目

这个项目是基于于 [qwong/j-webp](https://github.com/qwong/j-webp)项目，而 [qwong/j-webp](https://github.com/qwong/j-webp) 是基于 [webp project of Luciad](https://bitbucket.org/luciad/webp-imageio) 0.4.2项目。

webp project of Luciad这个项目提供了java上一个关于处理webp的可用实现，但是它需要开发者手动`java.library.path`中安装对应的动态链接库，非常不方便。qwong/j-webp项目作者为了解决这个问题，改进了对动态链接库的读取方式，把从`java.library.path`读取改成了从项目resource文件中读取（具体内容见`com.luciad.imageio.webp.WebP.loadNativeLibrary`方法）。

虽然qwong/j-webp项目解决了动态链接库依赖问题，但是它的作者并未对这些代码提供一个良好封装，毕竟开发者不希望在自己项目里面直接引入第三方包的源码，所以有了`webp-imageio-core`提供一个可用的jar包，只要导入项目即可使用。