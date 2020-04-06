# AiYaCompressHelper
压缩，图片压缩，压缩Bitmap，Compress,CompressImage,CompressFile,CompressBitmap<br><br>

主要通过尺寸压缩和质量压缩，以达到清晰度最优
本项目 [fork自https://github.com/nanchen2251/CompressHelper](https://github.com/nanchen2251/CompressHelper)
修改了//import android.media.ExifInterface;>import androidx.exifinterface.media.ExifInterface;
## 效果图<br>
 ![](/111.png)

#### ⊙开源不易，已经fork

## 特点
  1、支持压缩单张图片和多张图片<br>
## 使用方法
#### 1、添加依赖<br>
##### Step 1. Add the dependency
```java
dependencies {
	   implementation 'com.mhy.compress:CompressHelper:1.0.0'
	}
```
#### 2、在Activity里面使用<br>
```java
   File newFile = CompressHelper.getDefault(this).compressToFile(oldFile);
```
#### 3、你也可以自定义属性
```java
   File newFile = new CompressHelper.Builder(this)
                    .setMaxWidth(720)  // 默认最大宽度为720
                    .setMaxHeight(960) // 默认最大高度为960
                    .setQuality(80)    // 默认压缩质量为80
		    .setFileName(yourFileName) // 设置你需要修改的文件名
                    .setCompressFormat(CompressFormat.JPEG) // 设置默认压缩为jpg格式
                    .setDestinationDirectoryPath(Environment.getExternalStoragePublicDirectory(
                            Environment.DIRECTORY_PICTURES).getAbsolutePath())
                    .build()
                    .compressToFile(oldFile);
```
该项目参考了：

* [https://github.com/zetbaitsu/Compressor](https://github.com/zetbaitsu/Compressor) 
* [https://github.com/Curzibn/Luban](https://github.com/Curzibn/Luban)

图片常常导致OOM

　　1）大家一定都知道，手机相机拍照的图片都是几千像素的，而我们的手机屏幕却一般是720的，并且一张大图，动则几M，上传起来用户流量着实受不了，那么压缩资源体积就变得相当麻烦了，这里就给大家讲解一下压缩上传的正确姿势。


　　2）Bitmap是引起OOM的罪魁祸首之一，所以我们一定为了节约内存，一般都会在服务器上缓存一个缩略图，这样不但可以提升下载速度，减少用户流量，还达到了很好的节约内存的目的。

　　要是我们能把bitmap设置为imageView的大小，根据要显示的ImageView来压缩Bitmap那肯定最好了。

　　根据这样的思路，我们肯定得首先算出imageView的宽高，这个很简单；直接imageView.getWidth()和imageView.getHeight()方法就可以达到目的。

　　3）如果你操作图片，你一定知道BitmapFatory，因为我们通常使用它来操作图片。

　　　  BitmapFactory这个类提供了多个解析方法（decodeByteArray,decodeFile,decodeResource等）用来创建bitmap对象，其中sd卡图片用decodeFile，传入path路径和Options就可以了，而网络图片我们通常使用decodeStream方法，资源文件采用decodeResource；

　　　　然而这些方法，都会为bitmap分配内存，图片太大一定会导致OOM的，所以我们需要先进行压缩，使用BitmapFatory.Options。

　　4）BitmapFactory.Options

　　　　它有一个inJustDecodeBounds属性，当这个属性为true的时候，调用上面三个方法返回的就不是一个完整的bitmap对象，而是null。因为它禁止这些方法为bitmap分配内存，当时设置这个属性为true的时候，Options的outWidth，outHeight和outMimeType属性就会被复制。这样我们就可以在加载图片之前获取到图片的长宽和MIME类型。就等于不读取这个图片，却获取到了它的参数，的确很6。

　　　　说到这里，必须说到一个很6的属性了，inSampleSize，可以理解为压缩比率，设置好这个比率，就能调用上面的decodeXXXX方法获得缩略图了，如果图片大小都一致，那还可以定死它，可我们的图片却大小不一，那我们应该如何获得正确的inSampleSize值呢？可以通过下面的方法，动态计算。
/**
     * @description 计算图片的压缩比率
     *
     * @param options 参数
     * @param reqWidth 目标的宽度
     * @param reqHeight 目标的高度
     * @return
     */
    private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
        // 源图片的高度和宽度
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;
        if (height > reqHeight || width > reqWidth) {
            final int halfHeight = height / 2;
            final int halfWidth = width / 2;
            // Calculate the largest inSampleSize value that is a power of 2 and keeps both
            // height and width larger than the requested height and width.
            while ((halfHeight / inSampleSize) > reqHeight
                    && (halfWidth / inSampleSize) > reqWidth) {
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
　　然而，事实却不如我们想象那么美好，inSampleSize官方注释告诉我们一个必须注意的点：因为inSampleSize只能是2的整数次幂，意味着如果上面我们算出来inSampleSize为6的话，这时候只能向下取得整数次幂，就是4。这样设计的原因很可能是为了渐变bitmap压缩，毕竟按照2的次方进行压缩会比较高效和方便。

　　那遇上这样的问题，肯定是无法达到我们想要的效果的，比如我们计算出来的inSampleSize是15，向下取就成了8，明显差距太大。那有没有一种方法达到我们的效果呢?

　　答案是肯定的！

　　5）再次压缩

　　别忽略了Bitmap有这么一个方法：createScaleBitmap！！

　　这个方法可以给我们按照要求拉伸/缩小一个bitmap，我们可以通过这个方法把我们之前得到的较大的缩略图进行缩小，让其完全符合我们的需求。
　　
/**
     * @description 通过传入的bitmap，进行压缩，得到符合标准的bitmap
     * @param src
     * @param dstWidth
     * @param dstHeight
     * @return
     */
    private static Bitmap createScaleBitmap(Bitmap src, int dstWidth, int dstHeight, int inSampleSize) {
        //如果inSampleSize是2的倍数，也就说这个src已经是我们想要的缩略图了，直接返回即可。
        if (inSampleSize % 2 == 0) {
            return src;
        }
        // 如果是放大图片，filter决定是否平滑，如果是缩小图片，filter无影响，我们这里是缩小图片，所以直接设置为false
        Bitmap dst = Bitmap.createScaledBitmap(src, dstWidth, dstHeight, false);
        if (src != dst) { // 如果没有缩放，那么不回收
            src.recycle(); // 释放Bitmap的native像素数组
        }
        return dst;
    }
　　或许你会问，为啥要先从inSampleSize产生一个缩略图A，而不直接通过createScaseBitmap()方法缩放呢？

　　因为如果要从原始的图片直接缩放的话，就需要把原始图片直接放入内存 中，这将十分危险！！！先通过inSmapleSize得到一个较大的缩略图，它会比原图小很多，直接加载到内存中再进行拉伸/缩小就比较安全了！

　　6）完整代码：

package com.example.nanchen.aiyaschoolpush.utils;

import android.content.res.Resources;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;

/**
 *
 * bitmap相关操作工具类
 *
 * @author nanchen
 * @fileName AiYaSchoolPush
 * @packageName com.example.nanchen.aiyaschoolpush.utils
 * @date 2016/11/28  09:45
 */

public class BitmapUtil {

    /**
     * @description 计算图片的压缩比率
     *
     * @param options 参数
     * @param reqWidth 目标的宽度
     * @param reqHeight 目标的高度
     * @return
     */
    private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
        // 源图片的高度和宽度
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;
        if (height > reqHeight || width > reqWidth) {
            final int halfHeight = height / 2;
            final int halfWidth = width / 2;
            // Calculate the largest inSampleSize value that is a power of 2 and keeps both
            // height and width larger than the requested height and width.
            while ((halfHeight / inSampleSize) > reqHeight
                    && (halfWidth / inSampleSize) > reqWidth) {
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }

    /**
     * @description 从Resources中加载图片
     *
     * @param res
     * @param resId
     * @param reqWidth
     * @param reqHeight
     * @return
     */
    public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId, int reqWidth, int reqHeight) {
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true; // 设置成了true,不占用内存，只获取bitmap宽高
        BitmapFactory.decodeResource(res, resId, options); // 读取图片长款
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight); // 调用上面定义的方法计算inSampleSize值
        // 使用获取到的inSampleSize值再次解析图片
        options.inJustDecodeBounds = false;
        Bitmap src = BitmapFactory.decodeResource(res, resId, options); // 载入一个稍大的缩略图
        return createScaleBitmap(src, reqWidth, reqHeight, options.inSampleSize); // 进一步得到目标大小的缩略图
    }

    /**
     * @description 通过传入的bitmap，进行压缩，得到符合标准的bitmap
     *
     * @param src
     * @param dstWidth
     * @param dstHeight
     * @return
     */
    private static Bitmap createScaleBitmap(Bitmap src, int dstWidth, int dstHeight, int inSampleSize) {
        //如果inSampleSize是2的倍数，也就说这个src已经是我们想要的缩略图了，直接返回即可。
        if (inSampleSize % 2 == 0) {
            return src;
        }
        // 如果是放大图片，filter决定是否平滑，如果是缩小图片，filter无影响，我们这里是缩小图片，所以直接设置为false
        Bitmap dst = Bitmap.createScaledBitmap(src, dstWidth, dstHeight, false);
        if (src != dst) { // 如果没有缩放，那么不回收
            src.recycle(); // 释放Bitmap的native像素数组
        }
        return dst;
    }

    /**
     * @description 从SD卡上加载图片
     *
     * @param pathName
     * @param reqWidth
     * @param reqHeight
     * @return
     */
    public static Bitmap decodeSampledBitmapFromFile(String pathName, int reqWidth, int reqHeight) {
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFile(pathName, options);
        options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
        options.inJustDecodeBounds = false;
        Bitmap src = BitmapFactory.decodeFile(pathName, options);
        return createScaleBitmap(src, reqWidth, reqHeight, options.inSampleSize);
    }
}
