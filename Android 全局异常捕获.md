今天就来说说作为程序猿的我们每天都会遇到的东西bug,出bug不可怕可怕的是没有出bug时的堆栈信息,那么对于bug的信息收集就显得尤为重要了,一般用第三方bugly或者友盟等等都能轻易收集,但是由于公司不让使用第三方,而安卓正好有原生的异常收集类UncaughtExceptionHandler,那么今天博客就从这个类开始.

UncaughtExceptionHandler见名知意,即他是处理我们未捕获的异常,具体使用分两步
1.实现我们自己的异常处理类
```
public class CrashHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {

    }
}
```
需要我们实现Thread.UncaughtExceptionHandler接口当有未捕获的异常的时候会回调uncaughtException(Thread thread, Throwable ex)方法

2.设置该CrashHandler为系统默认的

```
Thread.setDefaultUncaughtExceptionHandler(crashHandler);
```
---------------------------------
以上只是使用步骤的介绍,具体项目中的使用我直接贴代码
在application中初始化
```
package com.zly.www.basedemo.base;

import android.app.Application;
import android.content.Context;
import com.zly.www.basedemo.exception.CrashHandler;

/**
 * Created by zly on 2016/6/11.
 */
public class AppApplication extends Application {


    private static Context mContext;

    @Override
    public void onCreate() {
        super.onCreate();
        this.mContext = this;
        CrashHandler.getInstance().init(this);//初始化全局异常管理
    }

    public static Context getContext(){
        return mContext;
    }
}

```

CrashHandler 实现类如下

```
package com.zly.www.basedemo.exception;

import android.content.Context;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Environment;
import android.os.Looper;
import android.util.Log;
import android.widget.Toast;

import com.zly.www.basedemo.utils.AppManager;

import java.io.File;
import java.io.FileOutputStream;
import java.io.PrintWriter;
import java.io.StringWriter;
import java.io.Writer;
import java.lang.reflect.Field;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * 全局异常捕获
 * Created by zly on 2016/7/3.
 */
public class CrashHandler implements Thread.UncaughtExceptionHandler {

    /**
     * 系统默认UncaughtExceptionHandler
     */
    private Thread.UncaughtExceptionHandler mDefaultHandler;

    /**
     * context
     */
    private Context mContext;

    /**
     * 存储异常和参数信息
     */
    private Map<String,String> paramsMap = new HashMap<>();

    /**
     * 格式化时间
     */
    private SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss");

    private String TAG = this.getClass().getSimpleName();

    private static CrashHandler mInstance;

    private CrashHandler() {

    }

    /**
     * 获取CrashHandler实例
     */
    public static synchronized CrashHandler getInstance(){
        if(null == mInstance){
            mInstance = new CrashHandler();
        }
        return mInstance;
    }

    public void init(Context context){
        mContext = context;
        mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();
        //设置该CrashHandler为系统默认的
        Thread.setDefaultUncaughtExceptionHandler(this);
    }

    /**
     * uncaughtException 回调函数
     */
    @Override
    public void uncaughtException(Thread thread, Throwable ex) {
        if(!handleException(ex) && mDefaultHandler != null){//如果自己没处理交给系统处理
            mDefaultHandler.uncaughtException(thread,ex);
        }else{//自己处理
            try {//延迟3秒杀进程
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                Log.e(TAG, "error : ", e);
            }
            //退出程序
            AppManager.getAppManager().AppExit(mContext);
        }

    }

    /**
     * 收集错误信息.发送到服务器
     * @return 处理了该异常返回true,否则false
     */
    private boolean handleException(Throwable ex) {
        if (ex == null) {
            return false;
        }
        //收集设备参数信息
        collectDeviceInfo(mContext);
        //添加自定义信息
        addCustomInfo();
        //使用Toast来显示异常信息
        new Thread() {
            @Override
            public void run() {
                Looper.prepare();
                Toast.makeText(mContext, "程序开小差了呢..", Toast.LENGTH_SHORT).show();
                Looper.loop();
            }
        }.start();
        //保存日志文件
        saveCrashInfo2File(ex);
        return true;
    }


    /**
     * 收集设备参数信息
     * @param ctx
     */
    public void collectDeviceInfo(Context ctx) {
        //获取versionName,versionCode
        try {
            PackageManager pm = ctx.getPackageManager();
            PackageInfo pi = pm.getPackageInfo(ctx.getPackageName(), PackageManager.GET_ACTIVITIES);
            if (pi != null) {
                String versionName = pi.versionName == null ? "null" : pi.versionName;
                String versionCode = pi.versionCode + "";
                paramsMap.put("versionName", versionName);
                paramsMap.put("versionCode", versionCode);
            }
        } catch (PackageManager.NameNotFoundException e) {
            Log.e(TAG, "an error occured when collect package info", e);
        }
        //获取所有系统信息
        Field[] fields = Build.class.getDeclaredFields();
        for (Field field : fields) {
            try {
                field.setAccessible(true);
                paramsMap.put(field.getName(), field.get(null).toString());
            } catch (Exception e) {
                Log.e(TAG, "an error occured when collect crash info", e);
            }
        }
    }

    /**
     * 添加自定义参数
     */
    private void addCustomInfo() {

    }

    /**
     * 保存错误信息到文件中
     *
     * @param ex
     * @return  返回文件名称,便于将文件传送到服务器
     */
    private String saveCrashInfo2File(Throwable ex) {

        StringBuffer sb = new StringBuffer();
        for (Map.Entry<String, String> entry : paramsMap.entrySet()) {
            String key = entry.getKey();
            String value = entry.getValue();
            sb.append(key + "=" + value + "\n");
        }

        Writer writer = new StringWriter();
        PrintWriter printWriter = new PrintWriter(writer);
        ex.printStackTrace(printWriter);
        Throwable cause = ex.getCause();
        while (cause != null) {
            cause.printStackTrace(printWriter);
            cause = cause.getCause();
        }
        printWriter.close();
        String result = writer.toString();
        sb.append(result);
        try {
            long timestamp = System.currentTimeMillis();
            String time = format.format(new Date());
            String fileName = "crash-" + time + "-" + timestamp + ".log";
            if (Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)) {
                String path = Environment.getExternalStorageDirectory().getAbsolutePath() + "/crash/";
                File dir = new File(path);
                if (!dir.exists()) {
                    dir.mkdirs();
                }
                FileOutputStream fos = new FileOutputStream(path + fileName);
                fos.write(sb.toString().getBytes());
                fos.close();
            }
            return fileName;
        } catch (Exception e) {
            Log.e(TAG, "an error occured while writing file...", e);
        }
        return null;
    }
}

```

activity管理类

```
package com.zly.www.basedemo.utils;

import java.util.Stack;

import android.app.Activity;  
import android.app.ActivityManager;  
import android.content.Context;

/**
 * Activity管理类：用于管理Activity和退出程序
 * Created by zly on 2016/6/11.
 */
public class AppManager {  

    // Activity栈  
    private static Stack<Activity> activityStack;  
    // 单例模式  
    private static AppManager instance;  

    private AppManager() {  
    }  

    /**
     * 单一实例
     */  
    public static AppManager getAppManager() {  
        if (instance == null) {  
            instance = new AppManager();  
        }  
        return instance;  
    }  

    /**
     * 添加Activity到堆栈
     */  
    public void addActivity(Activity activity) {  
        if (activityStack == null) {  
            activityStack = new Stack<Activity>();  
        }  
        activityStack.add(activity);  
    }  

    /**
     * 获取当前Activity（堆栈中最后一个压入的）
     */  
    public Activity currentActivity() {  
        Activity activity = activityStack.lastElement();  
        return activity;  
    }  

    /**
     * 结束当前Activity（堆栈中最后一个压入的）
     */  
    public void finishActivity() {  
        Activity activity = activityStack.lastElement();  
        finishActivity(activity);  
    }  

    /**
     * 结束指定的Activity
     */  
    public void finishActivity(Activity activity) {  
        if (activity != null) {  
            activityStack.remove(activity);  
            activity.finish();  
            activity = null;  
        }  
    }  

    /**
     * 结束指定类名的Activity
     */  
    public void finishActivity(Class<?> cls) {  
        for (Activity activity : activityStack) {  
            if (activity.getClass().equals(cls)) {  
                finishActivity(activity);
                break;
            }  
        }  
    }  

    /**
     * 结束所有Activity
     */  
    public void finishAllActivity() {  
        for (int i = 0; i < activityStack.size(); i++) {  
            if (null != activityStack.get(i)) {  
                activityStack.get(i).finish();  
            }  
        }  
        activityStack.clear();  
    }  

    /**
     * 退出应用程序
     */  
    public void AppExit(Context context) {  
        try {  
            finishAllActivity();  
            //退出程序
            android.os.Process.killProcess(android.os.Process.myPid());
            System.exit(1);
        } catch (Exception e) {  
        }  
    }  
}  
```
如有疑问欢迎留言~~
