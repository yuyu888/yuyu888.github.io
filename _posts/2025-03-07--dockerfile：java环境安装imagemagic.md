---
layout: mypost
title: dockerfile:java环境安装imagemagic
categories: [DOCKER]
---

## 说明
基于alpine3.21.3搭建的文件微服务镜像  
支持java， 软件版本：java-21-openjdk  
支持 ImageMagick  软件版本：ImageMagick-7.1.1-41  
支持 HEIC 软件版本：libheif-1.19.5  
支持 webp 软件版本：libwebp-1.4.0  

## dockerfile

````dockerfile
FROM alpine:3.21.3


# 安装完整依赖
RUN apk add --no-cache \
    # 安装openjdk21
    openjdk21 \
    # 兼容bash
    bash gcompat \
    imagemagick \
    imagemagick-dev \
    libheif \
    libwebp \
    libjpeg \
    libpng \
    fontconfig \
    tiff \
    imagemagick-tiff \      
    freetype \
    ghostscript \
    libraw \
    zlib \
    libxml2 \
    build-base \  
    # 安装字体库
    font-droid-nonlatin \
    ttf-freefont \
    ttf-dejavu \
    font-noto-cjk \
    && rm -rf /var/cache/apk/* \
    apk del build-base imagemagick-dev

# 安装外部字体库
ADD Chinese.tar.gz /usr/share/fonts

RUN cd /usr/share/fonts \
    && chmod -R 755 /usr/share/fonts/Chinese \
    && cd /usr/share/fonts/Chinese \
    && mkfontscale \
    && mkfontdir \
    && fc-cache -fv

CMD ["/bin/sh"]
````

