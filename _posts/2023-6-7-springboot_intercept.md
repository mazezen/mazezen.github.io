---
layout: post
title: "springboot 封装http请求工具"
date:    2023-6-7
tags: [SpringBoot]
comments: true
author: mazezen
---



### Springboot 拦截器配置本地资源映射

配置本地资源映射，可以读取本地的图片、文件、音频、视频等



#### 创建WebAppConfigurer 拦截器类, 重写 addResourceHandlers方法

```java
package com.jeffcail.javamall.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * @ClassName WebAppConfigurer
 * @Description TODO
 * @Author cc
 * @Date 2023/6/7 2:38 下午
 * @Version 1.0
 */
@Configuration
public class WebAppConfigurer implements WebMvcConfigurer {
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {

        registry.addResourceHandler("/image/swiper/**").
                addResourceLocations("file:/Users/cc/project/github/java/java_mall_wechat_applt/java-mall/swiperImgs/"); // 最后一个 '/' 记得不能落下，否则拼接错误，访问不了
    }
}

```

