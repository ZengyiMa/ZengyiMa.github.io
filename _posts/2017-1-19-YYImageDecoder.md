---
layout: post
title: "YYImage之YYImageDecoder"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 


YYImageDecoder 这个类是 YYImage 底层对图片处理的解码库，它会对图片解析成 YYImageFrame 对象，如果是 GIF 等动态图片，会解码成多张 YYImageFrame。其核心包括了图片格式探测，图片解码，提供对 APNG 的解码（iOS 8 系统提供），WebP 的解码。

# 普通图片的解码

YYImageDecoder 通过暴露的

```
- (BOOL)updateData:(nullable NSData *)data final:(BOOL)final;

```
来解析图片数据

在内部有一个对应的

```
- (BOOL)_updateData:(NSData *)data final:(BOOL)final
```
在做幕后的工作。
在通过 data 的数据探测到图片格式后，如果是 WebP 会使用 WebP 解析方法

```
- (void)_updateSourceWebP;
```

如果是 PNG 则会使用

```
- (void)_updateSourceWebP;
```
其他图片则会使用

```
- (void)_updateSourceImageIO;
```

让我们先来看普通的图片的处理，这里的普通指的是格式是 Gif，jpg，icon 等的图片。

定位到`_updateSourceImageIO`方法，其实这一层大部分是 CGImage 层的操作，代码比较长，这里整理成一个简短的流程，把关键点标注出来：

1. 使用`CGImageSourceCreateWithData`来通过 data 创建 CGImage 对象，如果是增量的 data 则使用`CGImageSourceCreateIncremental`和`CGImageSourceUpdateData`来生成 CGImage 
2. 通过`CGImageSourceGetCount`来获取图片的帧数
3. 使用`CGImageSourceCopyProperties`获取图片原信息，如果多个帧的图片可以通过`CGImageSourceCopyPropertiesAtIndex`来获取每一帧的图片信息，这里有几个 key 可以注意一下。

```
kCGImagePropertyPixelWidth：宽的像素
kCGImagePropertyPixelHeight：高的像素
kCGImagePropertyGIFDictionary：GIF相关的属性
kCGImagePropertyGIFUnclampedDelayTime：Gif的duration
kCGImagePropertyOrientation：图片的方向
```
4. 把前3步收集的信息封装成`_YYImageDecoderFrame`对象。并封装到内部 frame 集合中

# APNG的解码

# WebP的解码


# 其他

## 后台线程解码图片
主要思路是在获取 image 对象之后，`CGBitmapContextCreate`生成 bitmap 上下文，`CGContextDrawImage`把图片内容画入上下文中，`CGBitmapContextCreateImage`获取 bitmap 图片完成解码

实例代码：

```
 CGImageAlphaInfo alphaInfo = CGImageGetAlphaInfo(imageRef) & kCGBitmapAlphaInfoMask;
        BOOL hasAlpha = NO;
        if (alphaInfo == kCGImageAlphaPremultipliedLast ||
            alphaInfo == kCGImageAlphaPremultipliedFirst ||
            alphaInfo == kCGImageAlphaLast ||
            alphaInfo == kCGImageAlphaFirst) {
            hasAlpha = YES;
        }
        // BGRA8888 (premultiplied) or BGRX8888
        // same as UIGraphicsBeginImageContext() and -[UIView drawRect:]
        CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Host;
        bitmapInfo |= hasAlpha ? kCGImageAlphaPremultipliedFirst : kCGImageAlphaNoneSkipFirst;
        CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, 0, YYCGColorSpaceGetDeviceRGB(), bitmapInfo);
        if (!context) return NULL;
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef); // decode
        CGImageRef newImage = CGBitmapContextCreateImage(context);
        CFRelease(context);
        return newImage;
```
## 侦测图片的格式
老生常谈的话题，可以根据文件头几个字节来获取相应的图片格式
如 png 头4个字节是`0x89, 'P', 'N', 'G'`，gif 头3个字节是`G', 'I', 'F'`
 
实例代码

```
YYImageType YYImageDetectType(CFDataRef data) {
    if (!data) return YYImageTypeUnknown;
    uint64_t length = CFDataGetLength(data);
    if (length < 16) return YYImageTypeUnknown;
    
    const char *bytes = (char *)CFDataGetBytePtr(data);
    
    uint32_t magic4 = *((uint32_t *)bytes);
    switch (magic4) {
        case YY_FOUR_CC(0x4D, 0x4D, 0x00, 0x2A): { // big endian TIFF
            return YYImageTypeTIFF;
        } break;
            
        case YY_FOUR_CC(0x49, 0x49, 0x2A, 0x00): { // little endian TIFF
            return YYImageTypeTIFF;
        } break;
            
        case YY_FOUR_CC(0x00, 0x00, 0x01, 0x00): { // ICO
            return YYImageTypeICO;
        } break;
            
        case YY_FOUR_CC(0x00, 0x00, 0x02, 0x00): { // CUR
            return YYImageTypeICO;
        } break;
            
        case YY_FOUR_CC('i', 'c', 'n', 's'): { // ICNS
            return YYImageTypeICNS;
        } break;
            
        case YY_FOUR_CC('G', 'I', 'F', '8'): { // GIF
            return YYImageTypeGIF;
        } break;
            
        case YY_FOUR_CC(0x89, 'P', 'N', 'G'): {  // PNG
            uint32_t tmp = *((uint32_t *)(bytes + 4));
            if (tmp == YY_FOUR_CC('\r', '\n', 0x1A, '\n')) {
                return YYImageTypePNG;
            }
        } break;
            
        case YY_FOUR_CC('R', 'I', 'F', 'F'): { // WebP
            uint32_t tmp = *((uint32_t *)(bytes + 8));
            if (tmp == YY_FOUR_CC('W', 'E', 'B', 'P')) {
                return YYImageTypeWebP;
            }
        } break;
        /*
        case YY_FOUR_CC('B', 'P', 'G', 0xFB): { // BPG
            return YYImageTypeBPG;
        } break;
        */
    }
    
    uint16_t magic2 = *((uint16_t *)bytes);
    switch (magic2) {
        case YY_TWO_CC('B', 'A'):
        case YY_TWO_CC('B', 'M'):
        case YY_TWO_CC('I', 'C'):
        case YY_TWO_CC('P', 'I'):
        case YY_TWO_CC('C', 'I'):
        case YY_TWO_CC('C', 'P'): { // BMP
            return YYImageTypeBMP;
        }
        case YY_TWO_CC(0xFF, 0x4F): { // JPEG2000
            return YYImageTypeJPEG2000;
        }
    }
    
    // JPG             FF D8 FF
    if (memcmp(bytes,"\377\330\377",3) == 0) return YYImageTypeJPEG;
    
    // JP2
    if (memcmp(bytes + 4, "\152\120\040\040\015", 5) == 0) return YYImageTypeJPEG2000;
    
    return YYImageTypeUnknown;
}

```
