---
title: Android Bitmap 解析
date: 2018-02-05 15:34:42
tags:
---
# Android Bitmap 解析

本篇文章基於8.0.0_r1的AOSP

## copy

```java=
public Bitmap copy(Config config, boolean isMutable) {
        checkRecycled("Can't copy a recycled bitmap");
        ...
        Bitmap b = nativeCopy(mNativePtr, config.nativeInt, isMutable);
        ...
        return b;
    }
```

copy對應的JNI實作在[^2]中呼叫`createBitmap`產生java object,詳細請見[createBitmap](#createBitmap)

```cpp=
static jobject Bitmap_copy(JNIEnv* env, jobject, jlong srcHandle,
                           jint dstConfigHandle, jboolean isMutable) {
    SkBitmap src;
    reinterpret_cast<BitmapWrapper*>(srcHandle)->getSkBitmap(&src);
    ......

    SkColorType dstCT = GraphicsJNI::legacyBitmapConfigToColorType(dstConfigHandle);
    SkBitmap result;
    HeapAllocator allocator;

    if (!bitmapCopyTo(&result, dstCT, src, &allocator)) {
        return NULL;
    }
    auto bitmap = allocator.getStorageObjAndReset();
    return createBitmap(env, bitmap, getPremulBitmapCreateFlags(isMutable));
}
```

bitmapCopyTo的實現同在[^2]中,設定了新的SkBitmap的特性
並且呼叫[^5]中的`copyTo`讓skia完成SkBitmap最底層的copy

```cpp=
static bool bitmapCopyTo(SkBitmap* dst, SkColorType dstCT, const SkBitmap& src,
        SkBitmap::Allocator* alloc) {
	...
        SkImageInfo dstInfo = src.info().makeColorType(dstCT);
        if (dstCT == kRGBA_F16_SkColorType) {
             dstInfo = dstInfo.makeColorSpace(SkColorSpace::MakeSRGBLinear());
        }
        if (!dst->setInfo(dstInfo)) {return false;}
        if (!dst->tryAllocPixels(alloc, nullptr)) {return false;}
        switch (dstCT) {
            case kRGBA_8888_SkColorType:
            case kBGRA_8888_SkColorType: {...}
            case kRGB_565_SkColorType: {...}
            case kRGBA_F16_SkColorType: {...}
            default:
                return false;
        }
    }
    return src.copyTo(dst, dstCT, alloc);
}
```




## createBitmap
Android Bitmap和Canvas綁定做使用並透過`createBitmap`產生

```java=
Bitmap b = Bitmap.createBitmap(400, 400, Bitmap.Config.ARGB_8888);  
Canvas canvas = new Canvas(b);
```

其中`createBitmap`會呼叫[^1]中的
```java=
public static Bitmap createBitmap(@NonNull DisplayMetrics display,
            @NonNull @ColorInt int[] colors, int offset, int stride,
            int width, int height, @NonNull Config config) {
        ...
        Bitmap bm = nativeCreate(colors, offset, stride, width, height,
                            config.nativeInt, false, null, null);
        ...
        return bm;
    }
```

*nativeCreate*對應的JNI實作在[^2]

```cpp=
static jobject Bitmap_creator(JNIEnv* env, jobject, jintArray jColors,
                              jint offset, jint stride, jint width, jint height,
                              jint configHandle, jboolean isMutable,
                              jfloatArray xyzD50, jobject transferParameters) {
    SkColorType colorType = GraphicsJNI::legacyBitmapConfigToColorType(configHandle);
    ...
    SkBitmap bitmap;
    ...
    bitmap.setInfo(SkImageInfo::Make(width, height, colorType, kPremul_SkAlphaType, colorSpace));

    sk_sp<Bitmap> nativeBitmap = Bitmap::allocateHeapBitmap(&bitmap, NULL);
    if (!nativeBitmap) {
        return NULL;
    }
    ...
    return createBitmap(env, nativeBitmap.release(), getPremulBitmapCreateFlags(isMutable));
}
```

```cpp=
sk_sp<Bitmap> Bitmap::allocateHeapBitmap(SkBitmap* bitmap, SkColorTable* ctable) {
   return allocateBitmap(bitmap, ctable, &android::allocateHeapBitmap);
}
```

最終的android::allocateHeapBitmap如下
```cpp=
static sk_sp<Bitmap> allocateHeapBitmap(size_t size, const SkImageInfo& info, size_t rowBytes,
        SkColorTable* ctable) {
    void* addr = calloc(size, 1);
    if (!addr) {
        return nullptr;
    }
    return sk_sp<Bitmap>(new Bitmap(addr, size, info, rowBytes, ctable));
}
```


allocateHeapBitmap又將`android::allocateHeapBitmap`這個function的function pointer傳給`allocateBitmap`
透過alloc指到的function建立bitmap並回傳
```cpp=
static sk_sp<Bitmap> allocateBitmap(SkBitmap* bitmap, SkColorTable* ctable, AllocPixeRef alloc) {
    ...
    auto wrapper = alloc(size, info, rowBytes, ctable);
    if (wrapper) {
        wrapper->getSkBitmap(bitmap);
        // since we're already allocated, we lockPixels right away
        // HeapAllocator behaves this way too
        bitmap->lockPixels();
    }
    return wrapper;
}
```


Storage除了heap外,[^4]:還提供了三種方法
1. external:傳入外部address和對應的free function pointer
    ```cpp=
    struct {
    void* address;
    void* context;
    FreeFunc freeFunc;
    } external;
    ```
2. ashmem:呼叫[^3]中的`Bitmap::allocateAshmemBitmap`使用`mmap`建立Shared memory
3. hardware:呼叫[^3]中的`Bitmap::allocateHardwareBitmap`建立GraphicBuffer


最後會return`createBitmap`,實作同在[^2]中
```cpp=
jobject createBitmap(JNIEnv* env, Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {
    ...
    BitmapWrapper* bitmapWrapper = new BitmapWrapper(bitmap);
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isMutable, isPremultiplied, ninePatchChunk, ninePatchInsets);

    if (env->ExceptionCheck() != 0) {
        ALOGE("*** Uncaught exception returned from Java call!\n");
        env->ExceptionDescribe();
    }
    return obj;
}
```
此處會透過JNI在將skia建立好的native bitmap address傳回[^1]的constructor,以jlong直接存在Bitmap.java中mNativePtr中
```java=
Bitmap(long nativeBitmap, int width, int height, int density,
        boolean isMutable, boolean requestPremultiplied,
        byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
    ...
    mWidth = width;
    mHeight = height;
    mIsMutable = isMutable;
    mRequestPremultiplied = requestPremultiplied;

    mNinePatchChunk = ninePatchChunk;
    mNinePatchInsets = ninePatchInsets;
    if (density >= 0) {
        mDensity = density;
    }

    mNativePtr = nativeBitmap;
    ...
}
```

## recycle

回收機制本身是非同步的

```java=
public void recycle() {
if (!mRecycled && mNativePtr != 0) {
    if (nativeRecycle(mNativePtr)) {
        mNinePatchChunk = null;
    }
    mRecycled = true;
    }
}
```

JNI的實作在[^2]中
```cpp=
static jboolean Bitmap_recycle(JNIEnv* env, jobject, jlong bitmapHandle) {
    LocalScopedBitmap bitmap(bitmapHandle);
    bitmap->freePixels();
    return JNI_TRUE;
}

class BitmapWrapper {
public:
    BitmapWrapper(Bitmap* bitmap)
        : mBitmap(bitmap) { }

    void freePixels() {
        mInfo = mBitmap->info();
        mHasHardwareMipMap = mBitmap->hasHardwareMipMap();
        mAllocationSize = mBitmap->getAllocationByteCount();
        mRowBytes = mBitmap->rowBytes();
        mGenerationId = mBitmap->getGenerationID();
        mIsHardware = mBitmap->isHardware();
        mBitmap.reset();
    }
}
```
最後呼叫skia SkBitmap[^5]的reset呼叫sk_bzero將bitmapinstance清空

[^1]:[frameworks/base/graphics/java/android/graphics/Bitmap.java](https://android.googlesource.com/platform/frameworks/base/+/android-8.0.0_r1/graphics/java/android/graphics/Bitmap.java)

[^2]:[frameworks/base/core/jni/android/graphics/Bitmap.cpp](https://android.googlesource.com/platform/frameworks/base/+/android-8.0.0_r1/core/jni/android/graphics/Bitmap.cpp)

[^3]:[frameworks/base/libs/hwui/hwui/Bitmap.cpp](https://android.googlesource.com/platform/frameworks/base/+/android-8.0.0_r1/libs/hwui/hwui/Bitmap.cpp)

[^4]:[frameworks/base/libs/hwui/hwui/Bitmap.h](https://android.googlesource.com/platform/frameworks/base/+/android-8.0.0_r1/libs/hwui/hwui/Bitmap.h)

[^5]:[external/skia/src/core/SkBitmap.cpp](https://android.googlesource.com/platform/external/skia/+/android-8.0.0_r1/src/core/SkBitmap.cpp)

## Ref

- [Bitmap](https://developer.android.com/reference/android/graphics/Bitmap.html)
- [Android6.0 Bitmap存储以及Parcel传输源码分析](http://blog.csdn.net/xxxzhi/article/details/51490253)
