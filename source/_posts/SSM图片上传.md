---
title: SSM图片上传
date: 2019-04-09 10:44:32
tags: 
- java
categories: technology
keywords: java
description: SSM图片上传
---

## SSM图片上传


### 1、相关配置

#### Pom依赖

```xml
<!-- 文件上传的jar包 -->
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.4</version>
</dependency>
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.1</version>
</dependency>
```



#### SpringMvc的配置类(图片文件解析器，及一些参数)

```xml
<!-- 文件上传配置 -->
    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <!-- 默认编码 -->
        <property name="defaultEncoding" value="UTF-8"/>
        <!-- 上传文件大小限制为31M，31*1024*1024 -->
        <property name="maxUploadSize" value="32505856"/>
        <!-- 内存中的最大值 -->
        <property name="maxInMemorySize" value="4096"/>
    </bean>
```



#### Tomcat配置虚拟访问路径/物理存放路径的映射 => 用于映射请求

![](https://ws1.sinaimg.cn/large/006f2SyGgy1g1w5w3pqbtj30tx0ky75o.jpg)



### 2、代码部分



#### 上传图片工具类 PictureUtil.java

```java
public class PictureUtil {

    private static final Logger logger = LoggerFactory.getLogger(PictureUtil.class);

    private static final String CODES = "0123456789";
    private static final int NUM = 5;

    /**
     * 存入图片文件于本地
     *
     * @param picturePath  路径
     * @param picture 图片流
     * @return pictureName
     */
    public static String insertPicture(String picturePath, MultipartFile picture) {
        // 获取路径
        String path = Objects.requireNonNull(ClassUtils.getDefaultClassLoader().getResource("")).getPath() + "static/" + picturePath + '/';
        logger.info(path);
        if (picture.isEmpty()) {
            return null;
        }

        // 获取文件名
        String pictureName = picture.getOriginalFilename();
        logger.info("接收到的文件名为：" + pictureName);
        // 获取文件的后缀名
        String suffixName = pictureName.substring(pictureName.lastIndexOf("."));
        logger.info("获取后缀：" + suffixName);

        // 随机图片名
        pictureName = generateVerifyCode(NUM,CODES) + suffixName;
        File dest = new File(path, pictureName);

        // 检测是否存在目录
        if (!dest.getParentFile().exists()) {
            dest.getParentFile().mkdirs();
        }
        try {
            picture.transferTo(dest);
            logger.info("上传成功后的文件路径：" + path + pictureName);
        } catch (IllegalStateException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
        return pictureName;
    }

    /**
     * 使用指定源生成验证码
     *
     * @param verifySize 验证码长度
     * @param sources    验证码字符源
     * @return String
     */
    private static String generateVerifyCode(int verifySize, String sources) {
        int codesLen = sources.length();
        Random rand = new Random(System.currentTimeMillis());
        StringBuilder verifyCode = new StringBuilder(verifySize);
        for (int i = 0; i < verifySize; i++) {
            verifyCode.append(sources.charAt(rand.nextInt(codesLen - 1)));
        }
        return verifyCode.toString();
    }
}
```



#### 控制层图片上传

```java
@Controller
@RequestMapping(path = "/picture")
public class PictureController {

    private static Logger logger = LoggerFactory.getLogger(PictureController.class);

    /**
     * 图片上传target
     *
     * @param image 头像文件
     * @param id 用户id
     * @return com.lucky.base.common.Result<java.lang.String>
     */
    @ResponseBody
    @RequestMapping(value = "/upload/{id}", method = RequestMethod.POST)
    public String upload(@RequestParam("image") MultipartFile image, @PathVariable int id) {
        String path = "image/user";
        // 保存图片与本地
        String name = PictureUtil.insertPicture(path, image);
        if (name == null) {
            // 保存本地失败
            return "fail";
        }
        String imageUrl = "image/user/" + name;
        // TODO 根据id更新 用户头像
        return "http://localhost:8080/"+imageUrl;
    }
}
```



### 3、Postman测试图片上传

postman模拟上传图片：

![](https://ws1.sinaimg.cn/large/006f2SyGgy1g1w6nrle61j30od0bjjs6.jpg)

本地存储结果：

![](https://ws1.sinaimg.cn/large/006f2SyGgy1g1w6fo2nujj307o0bqmx8.jpg)

浏览器直接访问：http://localhost:8080/image/user/25042.jpg 可查看效果