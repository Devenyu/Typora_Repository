# Typora图床教程

typora支持将图片上传到服务器中并返回url，有了这个功能可以直接导入到博客中不再需要担心图片丢失了。

------

## 1、确保typora的版本在0.9.86以上

![image-20210824102238813](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210824102238813.png)

## 2、打开文件的偏好设置，对图像进行设置

![image-20210824102745757](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210824102745757.png)

## 3、在七牛云中注册、实名认证完成，并创建一个存储空间

![image-20210824103028496](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210824103028496.png)

## 4、在操作2打开的配置文件中写入七牛云对应的信息

```json
{
  "picBed": {
    "uploader": "qiniu",
    "qiniu": {
      "accessKey": "",
      "secretKey": "",
      "bucket": "", // 存储空间名
      "url": "http://", // 自定义域名
      "area":  "z2", // 存储区域编号
      "options": "", // 网址后缀，比如？imgslim
      "path": "img/" // 自定义存储路径，比如 img/
    }
  },
  "picgoPlugins": {}
}
```

## 5、验证图片上传选项

![image-20210824103348201](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210824103348201.png)

![image-20210824103323075](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210824103323075.png)

## 6、上传成功

