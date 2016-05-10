# XposedNoRebootModuleSample
Xposed Module Sample No Need To Reboot When Debug  不需要频繁重启调试的Xposed 模块源码例子

#####
    @Override
    public void handleLoadPackage(final XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable
    {
        if (!"cn.asiontang.location.demo".equals(loadPackageParam.packageName))
            return;
        try
        {
            //在发布时，直接调用即可。
            if (!BuildConfig.DEBUG)
            {
                handleLoadPackage4release(loadPackageParam);
                return;
            }
            //在调试模式为了不频繁重启，使用反射的方式调用自身的指定函数。

            /*【方法2】*/
            final String packageName = MainXposedHook.class.getPackage().getName();
            String filePath = String.format("/data/app/%s-%s.apk", packageName, 1);
            if (!new File(filePath).exists())
            {
                filePath = String.format("/data/app/%s-%s.apk", packageName, 2);
                if (!new File(filePath).exists())
                {
                    XposedBridge.log("Error:在/data/app找不到APK文件" + packageName);
                    return;
                }
            }
            final PathClassLoader pathClassLoader = new PathClassLoader(filePath, ClassLoader.getSystemClassLoader());
            final Class<?> aClass = Class.forName(packageName + "." + MainXposedHook.class.getSimpleName(), true, pathClassLoader);
            final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
            aClassMethod.invoke(aClass.newInstance(), loadPackageParam);

            /*【方法1】：无法达到效果*/
            //final Class<MainActivity> pathClassLoader = MainActivity.class;
            //final Class<?> aClass = Class.forName(pathClassLoader.getPackage().getName() + "." + MainXposedHook.class.getSimpleName(), true, pathClassLoader.getClassLoader());
            //final Method aClassMethod = aClass.getMethod("handleLoadPackage4release", XC_LoadPackage.LoadPackageParam.class);
            //aClassMethod.invoke(aClass.newInstance(), loadPackageParam);
        }
        catch (final Exception e)
        {
            XposedBridge.log(e);
        }
    }

    public void handleLoadPackage4release(final XC_LoadPackage.LoadPackageParam loadPackageParam)
    {
        XposedBridge.log("------------5");
    }
