---
layout:     post
title:      "MultiDexApplication"
subtitle:   "64k方法数google的解决方案"
date:       2019-2-15 5:30:00
author:     "chchaooo"
header-img: "img/main_banner.jpg"
header-mask: 0.3
catalog:    true
tags:
    - MultiDex
   
---

近期在看android热更新和插件化相关主题的东西。想到Android官方对一个古老的64k方法数问题的解法也是通过多dex来完成的，于是乎过来看看google是怎么完成dex方法加载的。

### 64k方法数问题
 
Android APK 文件是一个压缩的文件，它的里面包含的classes.dex文件是可执行Dalvik文件，这个.dex文件中存放的是所有编译后的java代码。而Dalvik可执行文件规范限制了单个,dex文件最多能够引用的方法数是65536个(方法的计数使用了一个short类型数字，2的16次方为65536)，这其中包含了Android Framework、APP引用的第三方函数库以及APP自身的方法。这样对于现在的app来说明显不够用（各个引用库越来越好用，方法数也越来越大了。。）。

解决方案：
（1）android5.0之前使用dalvik虚拟机，使用MultidexApplication来解决
```
compile 'com.android.support:multidex:1.0.2'
```

```
defaultConfig {
    multiDexEnabled true
}
```

```
public class MyApplication extends MultiDexApplication {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
}
```
（2）android5.0之后使用art虚拟机，art天然支持从apk中加载多个dex来预编译生成.oat文件来加速运行

### MultiDexApplication源码

从 MultiDex.install(this)这里看下去，看看一个应用启动时，做了哪些操作来进行多dex兼容。

1. 检查当前的设备虚拟机是否自动支持多dex启动，检查的代码如下：
```
static boolean isVMMultidexCapable(String versionString) {
        boolean isMultidexCapable = false;
        if(versionString != null) {
            Matcher matcher = Pattern.compile("(\\d+)\\.(\\d+)(\\.\\d+)?").matcher(versionString);
            if(matcher.matches()) {
                try {
                    int major = Integer.parseInt(matcher.group(1));
                    int minor = Integer.parseInt(matcher.group(2));
                    isMultidexCapable = major > 2 || major == 2 && minor >= 1;
                } catch (NumberFormatException var5) {
                    ;
                }
            }
        }

        Log.i("MultiDex", "VM with version " + versionString + (isMultidexCapable?" has multidex support":" does not have multidex support"));
        return isMultidexCapable;
    }
```
虚拟机版本2.1以及以上版本均自动支持多Dex加载

install方法里面调用了下面的方法（抽取核心代码）
```
static void doInstallation(Context context, File sourceApk ...){
    ClassLoader loader = context.getApplicationContext().getClassLoader();
    // 删除packagename对应目录下所有的文件
    clearOldDexDir(); 
    // 在packagename对应目录下创建了/code_cache/secondary-dexes/目录
    File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
    // 为apk中的每个dex创建一个MultiDexExtractor.ExtractedDex（extends自File），
    //文件名为apk.name.classes + "2/3/4/..(第几个dex)"+".zip"
    List<? extends File> files = MultiDexExtractor.load(mainContext, sourceApk, dexDir, prefsKeyPrefix, false);
    // 最终的安装逻辑
    installSecondaryDexes(loader, dexDir, files);
}
```
安装的过程，根据当前的系统版本，选择不同策略
```
    private static void installSecondaryDexes(){
        if(!files.isEmpty()) {
            if(VERSION.SDK_INT >= 19) {
                MultiDex.V19.install(loader, files, dexDir);
            } else if(VERSION.SDK_INT >= 14) {
                MultiDex.V14.install(loader, files, dexDir);
            } else {
                MultiDex.V4.install(loader, files);
            }
        }

    }
```
而install的过程实际就是利用反射来向DexPathList中的dexElements数组中加入第二个（或更多个）dex文件

```
/**
     * Installer for platform versions 19.
     */
    private static final class V19 {

        private static void install(ClassLoader loader,
                List<? extends File> additionalClassPathEntries,
                File optimizedDirectory)
                        throws IllegalArgumentException, IllegalAccessException,
                        NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
            /* The patched class loader is expected to be a descendant of
             * dalvik.system.BaseDexClassLoader. We modify its
             * dalvik.system.DexPathList pathList field to append additional DEX
             * file entries.
             */
            Field pathListField = findField(loader, "pathList");
            Object dexPathList = pathListField.get(loader);
            ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
            expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList,
                    new ArrayList<File>(additionalClassPathEntries), optimizedDirectory,
                    suppressedExceptions));
            if (suppressedExceptions.size() > 0) {
                for (IOException e : suppressedExceptions) {
                    Log.w(TAG, "Exception in makeDexElement", e);
                }
                Field suppressedExceptionsField =
                        findField(dexPathList, "dexElementsSuppressedExceptions");
                IOException[] dexElementsSuppressedExceptions =
                        (IOException[]) suppressedExceptionsField.get(dexPathList);

                if (dexElementsSuppressedExceptions == null) {
                    dexElementsSuppressedExceptions =
                            suppressedExceptions.toArray(
                                    new IOException[suppressedExceptions.size()]);
                } else {
                    IOException[] combined =
                            new IOException[suppressedExceptions.size() +
                                            dexElementsSuppressedExceptions.length];
                    suppressedExceptions.toArray(combined);
                    System.arraycopy(dexElementsSuppressedExceptions, 0, combined,
                            suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
                    dexElementsSuppressedExceptions = combined;
                }

                suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
            }
        }
```
通过反射找到 classloader中的DexPathList，再找到DexpathList中的dexElements，然后通过反射调用DexPathList中的makeDexElements方法，将第二个dex转化成DexFile，最后通过反射将结果塞进dexElements数组中。这样就完成了第二个dex的加载

















