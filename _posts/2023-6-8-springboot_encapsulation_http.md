---
layout: post
title: "springboot 封装http请求工具"
date:    2023-6-8
tags: [SpringBoot]
comments: true
author: mazezen
---


在日常项目中，可能经常会遇到发起http请求，调用别的资源拿到需要的数据。比方说调用公司的某一个http服务、第三方http接口等

http请求方式分为好几种，如果遵循resultful api格式，那就分为 post、get、delete、put等

最常见的为 `POST` `GET` 两种方式

基于springboot的http请求工具类

依赖

```xml
<dependency>
  <groupId>org.apache.httpcomponents</groupId>
  <artifactId>httpclient</artifactId>
  <version>4.5.2</version>
</dependency>
```



HttpClient.java

```java
package com.jeffcail.javamall.util;

import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.NameValuePair;
import org.apache.http.client.ClientProtocolException;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.conn.ssl.DefaultHostnameVerifier;
import org.apache.http.conn.util.PublicSuffixMatcher;
import org.apache.http.conn.util.PublicSuffixMatcherLoader;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.message.BasicNameValuePair;
import org.apache.http.util.EntityUtils;
import org.springframework.stereotype.Component;

import java.io.*;
import java.net.URL;
import java.util.ArrayList;
import java.util.Map;

/**
 * @ClassName HttpClient
 * @Description TODO
 * @Author cc
 * @Date 2023/6/8 3:14 下午
 * @Version 1.0
 */
@Component
public class HttpClient {

    /**
     * 默认参数设置
     * setConnectTimeout：设置连接超时时间，单位毫秒。
     * setConnectionRequestTimeout：设置从connect Manager获取Connection 超时时间，单位毫秒。
     * setSocketTimeout：请求获取数据的超时时间，单位毫秒。访问一个接口，多少时间内无法返回数据，就直接放弃此次调用。 暂时定义15分钟
     */
    private RequestConfig requestConfig = RequestConfig.custom().
            setSocketTimeout(700000).
            setConnectTimeout(700000).
            setConnectionRequestTimeout(700000).
            build();

    /**
     * 静态内部类---作用：单例产生类的实例
     * @author jeffcail
     *
     */
    private static class LazyHolder {
        private static final HttpClient INSTANCE = new HttpClient();
    }

    private HttpClient(){}

    public static HttpClient getInstance() {
        return LazyHolder.INSTANCE;
    }

    /**
     * 发送 post 请求
     * @param url
     * @return
     */
    public String sendHttpPost(String url) {
        HttpPost httpPost = new HttpPost(url);
        return sendHttpPost(httpPost);
    }

    /**
     *  发送 post 请求
     * @param url
     * @param Params 参数(格式:key1=value1&key2=value2)
     * @return
     */
    public String sendHttpPost(String url, String Params) {
        HttpPost httpPost = new HttpPost(url);
        try {
            StringEntity stringEntity = new StringEntity(Params, "UTF-8");
            stringEntity.setContentType("application/x-www-form-urlencoded");
            httpPost.setEntity(stringEntity);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return sendHttpPost(httpPost);
    }

    /**
     * 发送 post 请求
     * @param url
     * @param maps map参数
     * @return
     */
    public String sendHttpPost(String url, Map<String, String> maps) {
        HttpPost httpPost = new HttpPost(url);
        ArrayList<NameValuePair> nameValuePairs = new ArrayList<>();
        for (String key : maps.keySet()) {
            nameValuePairs.add(new BasicNameValuePair(key, maps.get(key)));
        }

        try {
            httpPost.setEntity(new UrlEncodedFormEntity(nameValuePairs, "UTF-8"));
        } catch (Exception e) {
            e.printStackTrace();
        }

        return sendHttpPost(httpPost);
    }

    /**
     * 发送Post请求
     * @param httpPost
     * @return
     */
    private String sendHttpPost(HttpPost httpPost) {
        CloseableHttpClient httpClient = null;
        CloseableHttpResponse response = null;
        HttpEntity entity = null;
        String responseContent = null;
        try {
            // 创建默认的httpClient实例
            httpClient = HttpClients.createDefault();
            httpPost.setConfig(requestConfig);
            // 执行请求
            long execStart = System.currentTimeMillis();
            response = httpClient.execute(httpPost);
            long execEnd = System.currentTimeMillis();
            System.out.println("=================执行post请求耗时："+(execEnd-execStart)+"ms");
            long getStart = System.currentTimeMillis();
            entity = response.getEntity();
            responseContent = EntityUtils.toString(entity, "UTF-8");
            long getEnd = System.currentTimeMillis();
            System.out.println("=================获取响应结果耗时："+(getEnd-getStart)+"ms");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                // 关闭连接,释放资源
                if (response != null) {
                    response.close();
                }
                if (httpClient != null) {
                    httpClient.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return responseContent;
    }

    /**
     * 发送 get请求
     * @param url
     */
    public String sendHttpGet(String url) {
        HttpGet httpGet = new HttpGet(url);// 创建get请求
        return sendHttpGet(httpGet);
    }

    /**
     * 发送 get请求Https
     * @param httpUrl
     */
    public String sendHttpsGet(String httpUrl) {
        HttpGet httpGet = new HttpGet(httpUrl);// 创建get请求
        return sendHttpsGet(httpGet);
    }

    /**
     * 发送Get请求
     * @param httpGet
     * @return
     */
    private String sendHttpGet(HttpGet httpGet) {
        CloseableHttpClient httpClient = null;
        CloseableHttpResponse response = null;
        HttpEntity entity = null;
        String responseContent = null;
        try {
            // 创建默认的httpClient实例.


            httpClient = HttpClients.createDefault();

            httpGet.setConfig(requestConfig);
            // 执行请求
            response = httpClient.execute(httpGet);
            entity = response.getEntity();
            responseContent = EntityUtils.toString(entity, "UTF-8");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                // 关闭连接,释放资源
                if (response != null) {
                    response.close();
                }
                if (httpClient != null) {
                    httpClient.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return responseContent;
    }

    /**
     * 发送Get请求Https
     * @param httpGet
     * @return
     */
    private String sendHttpsGet(HttpGet httpGet) {
        CloseableHttpClient httpClient = null;
        CloseableHttpResponse response = null;
        HttpEntity entity = null;
        String responseContent = null;
        try {
            // 创建默认的httpClient实例.
            PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader.load(new URL(httpGet.getURI().toString()));
            DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(publicSuffixMatcher);
            httpClient = HttpClients.custom().setSSLHostnameVerifier(hostnameVerifier).build();
            httpGet.setConfig(requestConfig);
            // 执行请求
            response = httpClient.execute(httpGet);
            entity = response.getEntity();
            responseContent = EntityUtils.toString(entity, "UTF-8");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                // 关闭连接,释放资源
                if (response != null) {
                    response.close();
                }
                if (httpClient != null) {
                    httpClient.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return responseContent;
    }

    /**
     * 发送xml数据
     * @param url
     * @param xmlData
     * @return
     * @throws ClientProtocolException
     * @throws IOException
     */
    public static HttpResponse sendXMLDataByPost(String url, String xmlData)
            throws ClientProtocolException, IOException {
        org.apache.http.client.HttpClient httpClient = HttpClients.createDefault();
        HttpPost httppost = new HttpPost(url);
        StringEntity entity = new StringEntity(xmlData);
        httppost.setEntity(entity);
        httppost.setHeader("Content-Type", "text/xml;charset=UTF-8");
        HttpResponse response = httpClient.execute(httppost);
        return response;
    }

    /**
     * 获得响应HTTP实体内容
     *
     * @param response
     * @return
     * @throws IOException
     * @throws UnsupportedEncodingException
     */
    public static String getHttpEntityContent(HttpResponse response) throws IOException, UnsupportedEncodingException {
        HttpEntity entity = response.getEntity();
        if (entity != null) {
            InputStream is = entity.getContent();
            BufferedReader br = new BufferedReader(new InputStreamReader(is, "UTF-8"));
            String line = br.readLine();
            StringBuilder sb = new StringBuilder();
            while (line != null) {
                sb.append(line + "\n");
                line = br.readLine();
            }
            return sb.toString();
        }
        return "";
    }

}

```

