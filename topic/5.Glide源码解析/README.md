# Glide源码解析
===========
![](./image/glide_logo.png)

## 功能介绍
Android图片加载框架，支持Video，Gif，SVG格式，支持缩略图请求，目标是打造更好的图片列表滑动体验。Glide有生命周期的概念，可以与Activity和Fragment绑定，默认使用基于HttpUrlConnection的自定义网络层，支持Okhttp和Volley，友好的内存管理和磁盘管理。

## 简单用法
```Java
Glide.with(context).load(url).into(imageView);
```

## 框架图
![](./image/glide_module.jpg)

## 基本概念


## 基本流程
### 创建RequestManager
```
Glide.with(context);
```
Context支持五类参数
* android.app.Activity
* android.support.v4.app.FragmentActivity
* android.app.Fragment
* android.support.v4.app.Fragment
* Context

创建流程
* 创建RequestManagerRetriever单例，实例化一个RequestMnanger，管理Activity和Fragment生命周期。
* 通过Activity和Fragment获取FragmentManager, 并创建一个对应的空页面RequestManagerFragment，添加到FragmentTransaction。
* RequestManagerFragment，管理子RequestManagerFragment，并实例化一个ActivityFragmentLifecycle，并将它传递给RequestManager。
```
// RequestManagerFragment生命周期管理
@Override
   public void onStart() {
       super.onStart();
       lifecycle.onStart();
   }

   @Override
   public void onStop() {
       super.onStop();
       lifecycle.onStop();
   }

   @Override
   public void onDestroy() {
       super.onDestroy();
       lifecycle.onDestroy();
   }
```

### 创建Request
支持的类型有网络图片，本地图片，应用资源，二进制流，Uri对象等。

加载普通图片，创建DrawableTypeRequest，所有Request的基类为GenericRequestBuilder。
```
Glide.with(context).load(url);
```

RequestManager实例化DrawableTypeRequest时，会将lifecycle传递给DrawableTypeRequest
```
return optionsApplier.apply(
             new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                     glide, requestTracker, lifecycle, optionsApplier));
```

DrawableTypeRequest提供类型转换，做一些特殊处理，例如生成bitmap，加载gif，下载图片等。
```
// 转换为BitmapTypeRequest
asBitmap();

// 转换为GifTypeRequest
asGif();

// 转换为FutureTarget<File>，同步方法
downloadOnly();
```

### 创建Target
Request可以创建多种Target，例如ImageView，自定义Target，
```
// 将图片加载到ImageView
into(ImageView);

// 自定义Target，得到图片加载各个状态的和生命周期回调。
into(Y target)；

// 不推荐，使用override(int width, int height).into(target)替代
into(width, height);
```


## Glide其他用法
### GlideModule

## 附录
### 参考资料
* [Glide 源码解析](https://github.com/android-cn/android-open-project-analysis/tree/master/tool-lib/image-cache/glide)

### 图片加载框架
* [Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)，个人发起的项目，Android早期开发比较火，于2015年停止维护。
* [Picasso](https://github.com/square/picasso)，Square公司出品
* [Fresco](https://github.com/facebook/fresco)，Facebook公司出品
