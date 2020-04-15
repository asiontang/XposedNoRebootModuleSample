# XposedNoRebootModuleSample
Xposed Module Sample No Need To Reboot When Debug  不需要频繁重启调试的Xposed 模块源码例子


## 实际可用的完整项目

* [asiontang/78.Xposed](https://github.com/asiontang/78.Xposed) 

## 部分核心代码

1. [78.Xposed/BaseXposedHookLoadPackage.java at master · asiontang/78.Xposed](https://github.com/asiontang/78.Xposed/blob/master/src/main/java/cn/asiontang/xposed/BaseXposedHookLoadPackage.java#L73) 
2. [78.Xposed/DebugModeUtils.java at master · asiontang/78.Xposed](https://github.com/asiontang/78.Xposed/blob/master/src/main/java/cn/asiontang/xposed/DebugModeUtils.java) 

```java

    /**
     * 在调试模式为了不频繁重启，使用反射的方式调用指定函数。来动态触发 Hook.
     */
    private boolean invokeHandleLoadPackage4release(final XC_LoadPackage.LoadPackageParam loadPackageParam)//
            throws IOException, NoSuchMethodException, IllegalAccessException, InstantiationException, ClassNotFoundException, InvocationTargetException
    {
        /*【方法5】*/
        {
            //需要解决新版本系统调试版本的路径随机化的问题:
            //Android 8拿到的调试包地址为: /data/app/cn.asiontang.xposed.auto_js_pro-TbJBoU7ixhS0Fiv8Z6qf7g==/base.apk 地址
            final String selfPackageName = this.getClass().getPackage().getName();
            final String hookPackageName = loadPackageParam.packageName;
            final String newApkFullPath = DebugModeUtils.getNewApkFullPathAcrossProcessByRoot(selfPackageName, hookPackageName);
            if (newApkFullPath == null)
            {
                LogEx.log(TAG, loadPackageParam.packageName, "Error:在指定目录(/data/data/{hookPackageName}/cache/!Ye.DebugModeUtils)获取不到.apk包最新地址");
                return false;
            }
            File file = new File(newApkFullPath);
            if (!file.exists())
            {
                LogEx.log(TAG, loadPackageParam.packageName, "Error:在/data/app找不到插件对应的.apk包", "apkFilePath=", newApkFullPath);
                return false;
            }
            LogEx.log(TAG, loadPackageParam.packageName, "newApkFullPath=", newApkFullPath);

            ClassLoader parentClassLoader;
            //在Android9.0报错:java.lang.ClassNotFoundException: Didn't find class "de.robv.android.xposed.IXposedHookLoadPackage"
            //NO:parentClassLoader = ClassLoader.getSystemClassLoader();
            //NO:parentClassLoader = loadPackageParam.classLoader;
            //在Android9.0会导致反射失效,无法拿到最新的代码.
            //NO:parentClassLoader = this.getClass().getClassLoader();
            parentClassLoader = XC_LoadPackage.LoadPackageParam.class.getClassLoader();
            LogEx.log(TAG, "parentClassLoader=", parentClassLoader);

            final PathClassLoader pathClassLoader = new PathClassLoader(newApkFullPath, parentClassLoader);
            LogEx.log(TAG, "pathClassLoader=", pathClassLoader);

            //在Android9.0上使用 getClassNameFromAsset(pathClassLoader) 会报错
            // java.io.FileNotFoundException: File doesn't exist: /data/app/cn.asiontang.xposed.heart_study-IBxQU_4AA4zvKX8qrEr-SA==/base.apk
            //因为当存在 parentClassLoader 时,会优先找父的资源,结果此时APK文件已经删除掉了,所以报错找不到.
            //所以需要一个单独的 resourcesLoader 使用 ClassLoader.getSystemClassLoader 作为parentClassLoader即可.
            final PathClassLoader resourcesLoader = new PathClassLoader(newApkFullPath, ClassLoader.getSystemClassLoader());
            final Class<?> aClass = Class.forName(getClassNameFromAsset(resourcesLoader), true, pathClassLoader);

            final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
            return (Boolean) aClassMethod.invoke(aClass.newInstance(), loadPackageParam);
        }
        /*【方法4】
        {
            //因为新旧版本的APK所在目录不一样,所以换一种更加通用的办法获取所在路径.
            // 1.旧版本路径是:/data/app/%s-%s.apk
            // 2.新版本路径是:/data/app/%s-%s/base.apk

            //TEST1: // com.thirdparty.superuser｜XposedBridge.BOOTCLASSLOADER｜dalvik.system.PathClassLoader[DexPathList[[zip file "/system/framework/XposedBridge.jar"],nativeLibraryDirectories=[/vendor/lib, /system/lib, /system/lib/arm]]]
            //TEST1: LogEx.log(loadPackageParam.packageName, "XposedBridge.BOOTCLASSLOADER", XposedBridge.BOOTCLASSLOADER);

            //TEST2: // com.thirdparty.superuser｜loadPackageParam.classLoader｜dalvik.system.PathClassLoader[DexPathList[[zip file "/system/app/Superuser/Superuser.apk"],nativeLibraryDirectories=[/system/app/Superuser/lib/x86, /vendor/lib, /system/lib, /system/lib/arm]]]
            //TEST2: LogEx.log(loadPackageParam.packageName, "loadPackageParam.classLoader", loadPackageParam.classLoader);

            //TEST3: // com.thirdparty.superuser｜this.getClass().getPackage().getName()｜cn.asiontang.xposed.auto_js_pro
            //TEST3: LogEx.log(loadPackageParam.packageName, "this.getClass().getPackage().getName()", this.getClass().getPackage().getName());

            //TEST4: // com.thirdparty.superuser｜this.getClass().getPackage().getClassLoader()｜dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/cn.asiontang.xposed.auto_js_pro-1/base.apk"],nativeLibraryDirectories=[/vendor/lib, /system/lib, /system/lib/arm]]]
            //TEST4: LogEx.log(loadPackageParam.packageName, "this.getClass().getPackage().getClassLoader()", this.getClass().getClassLoader());

            //输入:dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/cn.asiontang.xposed.auto_js_pro-1/base.apk"],nativeLibraryDirectories
            String apkFilePath = String.valueOf(getClass().getClassLoader());
            //结果:/data/app/cn.asiontang.xposed.auto_js_pro-1/base.apk
            apkFilePath = apkFilePath.substring(apkFilePath.indexOf("/data/app"), apkFilePath.indexOf(".apk\"]") + 4);
            if (!new File(apkFilePath).exists())
            {
                //每次DEBUG运行时,就会自动增加到2.所以需要适配.
                if (apkFilePath.contains("1/base.apk"))
                    apkFilePath = apkFilePath.replace("1/base.apk", "2/base.apk");
                else if (apkFilePath.contains("2/base.apk"))
                    apkFilePath = apkFilePath.replace("2/base.apk", "1/base.apk");
                else
                {
                    LogEx.log(loadPackageParam.packageName, "Error:意料外的路径格式,文件夹尾字符既不是1也不是2", "apkFilePath=", apkFilePath);
                    return false;
                }

                if (!new File(apkFilePath).exists())
                {
                    LogEx.log(TAG, loadPackageParam.packageName, "Error:在/data/app找不到插件对应的.apk包", "apkFilePath=", apkFilePath);
                    return false;
                }
            }

            final PathClassLoader pathClassLoader = new PathClassLoader(apkFilePath, ClassLoader.getSystemClassLoader());
            final Class<?> aClass = Class.forName(getClassNameFromAsset(pathClassLoader), true, pathClassLoader);
            final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
            return (Boolean) aClassMethod.invoke(aClass.newInstance(), loadPackageParam);
        }*/

        /*【方法3】*/
        /************************************************************
         *获取当前APK的包名,而不是当前框架的包名
         *以下方法都不行:返回值为 cn.asiontang.xposed
         * 1.BaseXposedHookLoadPackage.class.getPackage().getName();
         * 2.BuildConfig.APPLICATION_ID
         ************************************************************/
        //final String packageName = this.getClass().getPackage().getName();
        //String apkFilePath = String.format("/data/app/%s-%s.apk", packageName, 1);
        //if (!new File(apkFilePath).exists())
        //{
        //    apkFilePath = String.format("/data/app/%s-%s.apk", packageName, 2);
        //    if (!new File(apkFilePath).exists())
        //    {
        //        LogEx.log(loadPackageParam.packageName, "Error:在/data/app找不到.apk包" + packageName);
        //        return false;
        //    }
        //}
        //final PathClassLoader pathClassLoader = new PathClassLoader(apkFilePath, ClassLoader.getSystemClassLoader());
        //final Class<?> aClass = Class.forName(getClassNameFromAsset(pathClassLoader), true, pathClassLoader);
        //final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
        //return (Boolean) aClassMethod.invoke(aClass.newInstance(), loadPackageParam);

        /*【方法2】*/
        //final String packageName = 项目的类.class.getPackage().getName();
        //String apkFilePath = String.format("/data/app/%s-%s.apk", packageName, 1);
        //if (!new File(apkFilePath).exists())
        //{
        //    apkFilePath = String.format("/data/app/%s-%s.apk", packageName, 2);
        //    if (!new File(apkFilePath).exists())
        //    {
        //        LogEx.log(loadPackageParam.packageName, "Error:在/data/app找不到APK文件" + packageName);
        //        return;
        //    }
        //}
        //final PathClassLoader pathClassLoader = new PathClassLoader(apkFilePath, ClassLoader.getSystemClassLoader());
        //final Class<?> aClass = Class.forName(getClassNameFromAsset(pathClassLoader), true, pathClassLoader);
        //final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
        //Object isHandled = aClassMethod.invoke(aClass.newInstance(), loadPackageParam);

        /*【方法1】：无法达到效果*/
        //final Class<MainActivity> pathClassLoader = MainActivity.class;
        //final Class<?> aClass = Class.forName(pathClassLoader.getPackage().getName() + "." + XposedHookLoadPackage.class.getSimpleName(), true, pathClassLoader.getClassLoader());
        //final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
        //aClassMethod.invoke(aClass.newInstance(), mLoadPackageParam);
    }
```
