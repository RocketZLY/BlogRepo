> 近期leader提了很多这样的需求：每隔几个小时拉取服务器的配置信息存在本地、每隔一段时间跟服务端校对一下本地时间、每隔一段时间上传一下本地日志等等。其实这些本质都是定时任务，隔一段时间去干xxx，那么在安卓中定时任务无非几种实现方式，Handler(CountDownTimer)、Timer、while循环、AlarmManager。**（如果有遗漏还望留言告知O(∩_∩)O谢谢）**前三种大家基本都用过，也就不多赘言，本篇blog专注介绍AlarmManager。

## **AlarmManager介绍**
---
见名知意闹钟管理者，当然不代表AlarmManager只是用来做闹钟应用的，作为一个系统级别的提示服务，其实它的作用和Timer有点相似

1. 在指定时长后执行某项操作
2. 周期性的执行某项操作

并且AlarmManager对象可以配合Intent使用，定时的开启一个Activity,发送一个BroadCast,或者开启一个Service.那么用它实现定时任务再好不过了。
</br>
</br>
## **使用**
---
### **AlarmManager初体验**
先来一发简单的Demo体验一下


    AlarmManager am = (AlarmManager) getSystemService(Context.ALARM_SERVICE);

    Intent intent = new Intent(this, AlarmService.class);
    intent.setAction(AlarmService.ACTION_ALARM);
    PendingIntent pendingIntent = PendingIntent.getService(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
    if(Build.VERSION.SDK_INT < 19){
        am.set(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 5000, pendingIntent);
    }else{
        am.setExact(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 5000, pendingIntent);
    }

这个例子就是5秒钟后发送一个Action为AlarmService.ACTION_ALARM的Intent到AlarmService。
</br>
</br>
</br>
### **AlarmManager的常用API**

    set(int type, long triggerAtMillis, PendingIntent operation)

该方法用于设置一次性闹钟，第一个参数表示闹钟类型，第二个参数表示闹钟执行时间，第三个参数表示闹钟响应动作。

    setRepeating(int type, long triggerAtMillis,long intervalMillis, PendingIntent operation)

该方法用于设置重复闹钟，第一个参数表示闹钟类型，第二个参数表示闹钟首次执行时间，第三个参数表示闹钟两次执行的间隔时间，第四个参数表示闹钟响应动作。

    setInexactRepeating(int type, long triggerAtMillis,long intervalMillis, PendingIntent operation)

该方法也用于设置重复闹钟，与第二个方法相似，不过闹钟时间不精确。

    setExact(int type, long triggerAtMillis, PendingIntent operation)
    setWindow(int type, long windowStartMillis, long windowLengthMillis,PendingIntent operation)

**方法1和方法2在SDK_INT 19以前是精确的闹钟,19以后为了节能省电（减少系统唤醒和电池使用）。使用Alarm.set()和Alarm.setRepeating()已经不能保证精确性,不过还好Google又提供了两个精确的Alarm方法setWindow()和setExact(),所以19以后需要精确的闹钟就需要上面两个方法,具体原因后面再说**

    cancel(PendingIntent operation)

取消Intent相同的闹钟,这里是根据Intent中filterEquals(Intent other)方法来判断是否相同

    public boolean filterEquals(Intent other) {
            if (other == null) {
                return false;
            }
            if (!Objects.equals(this.mAction, other.mAction)) return false;
            if (!Objects.equals(this.mData, other.mData)) return false;
            if (!Objects.equals(this.mType, other.mType)) return false;
            if (!Objects.equals(this.mPackage, other.mPackage)) return false;
            if (!Objects.equals(this.mComponent, other.mComponent)) return false;
            if (!Objects.equals(this.mCategories, other.mCategories)) return false;

            return true;
        }

从方法体可以看出mAction、mData、mType、mPackage、mComponent、mCategories这几个完全一样就认定为同一Intent
</br>
</br>
</br>
### **闹钟类型**
这个闹钟类型就是前面setxxx()方法第一个参数int type.

- AlarmManager.ELAPSED_REALTIME：使用相对时间，可以通过SystemClock.elapsedRealtime() 获取（从开机到现在的毫秒数，包括手机的睡眠时间），设备休眠时并不会唤醒设备。
- AlarmManager.ELAPSED_REALTIME_WAKEUP：与ELAPSED_REALTIME基本功能一样，只是会在设备休眠时唤醒设备。
- AlarmManager.RTC：使用绝对时间，可以通过 System.currentTimeMillis()获取，设备休眠时并不会唤醒设备。
- AlarmManager.RTC_WAKEUP: 与RTC基本功能一样，只是会在设备休眠时唤醒设备。
</br>
</br>
</br>
### **举个栗子**
#### **1.点击按钮,AlarmManager3秒后发送intent到service弹出toast提示.**

activity代码

    public class MainActivity extends AppCompatActivity {

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

        }

        public void alarm(View v){
            AlarmManager am = (AlarmManager) getSystemService(Context.ALARM_SERVICE);

            Intent intent = new Intent(this, AlarmService.class);
            intent.setAction(AlarmService.ACTION_ALARM);
            PendingIntent pendingIntent = PendingIntent.getService(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
            if(Build.VERSION.SDK_INT < 19){
                am.set(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 3000, pendingIntent);
            }else{
                am.setExact(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 3000, pendingIntent);
            }
        }

    }

service代码

    public class AlarmService extends Service {

        public static String ACTION_ALARM = "action_alarm";
        private Handler mHanler = new Handler(Looper.getMainLooper());

        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
            return null;
        }


        @Override
        public int onStartCommand(Intent intent, int flags, int startId) {
            mHanler.post(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(AlarmService.this, "闹钟来啦", Toast.LENGTH_SHORT).show();
                }
            });
            return super.onStartCommand(intent, flags, startId);
        }
    }

效果如下

![](http://of1ktyksz.bkt.clouddn.com/alarmManager_demo1.gif)
</br>
</br>
</br>
#### **2.具体年月日启动闹钟**
核心代码如下

    Calendar calendar = Calendar.getInstance();
            calendar.set(Calendar.YEAR,2016);
            calendar.set(Calendar.MONTH,Calendar.DECEMBER);
            calendar.set(Calendar.DAY_OF_MONTH,16);
            calendar.set(Calendar.HOUR_OF_DAY,11);
            calendar.set(Calendar.MINUTE,50);
            calendar.set(Calendar.SECOND,0);
            //设定时间为 2016年12月16日11点50分0秒

            AlarmManager am = (AlarmManager) getSystemService(Context.ALARM_SERVICE);

            Intent intent = new Intent(this, AlarmService.class);
            intent.setAction(AlarmService.ACTION_ALARM);
            PendingIntent pendingIntent = PendingIntent.getService(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
            if(Build.VERSION.SDK_INT < 19){
                am.set(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);
            }else{
                am.setExact(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);
            }

效果跟上面没区别
</br>
</br>
</br>
### **AlarmManager闹钟不准原因**
前面介绍的所有的set方法其实都是调用内部的一个private的方法setImpl（），只是不同的set方法传入的值不同而已

    private void setImpl(int type, long triggerAtMillis, long windowMillis, long intervalMillis,
                int flags, PendingIntent operation, final OnAlarmListener listener, String listenerTag,
                Handler targetHandler, WorkSource workSource, AlarmClockInfo alarmClock) {
            if (triggerAtMillis < 0) {
                /* NOTYET
                if (mAlwaysExact) {
                    // Fatal error for KLP+ apps to use negative trigger times
                    throw new IllegalArgumentException("Invalid alarm trigger time "
                            + triggerAtMillis);
                }
                */
                triggerAtMillis = 0;
            }

            ......

            try {
                mService.set(mPackageName, type, triggerAtMillis, windowMillis, intervalMillis, flags,
                        operation, recipientWrapper, listenerTag, workSource, alarmClock);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

这里只展示了相关代码,而具体控是否精确是靠windowMillis这个参数

在看看普通的set()与setRepeating()方法如何传递windowMillis参数

    public void set(int type, long triggerAtMillis, PendingIntent operation) {
            setImpl(type, triggerAtMillis, legacyExactLength(), 0, 0, operation, null, null,
                    null, null, null);
        }

    public void setRepeating(int type, long triggerAtMillis,
            long intervalMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, legacyExactLength(), intervalMillis, 0, operation,
                null, null, null, null, null);
    }

可以发现windowMillis参数为legacyExactLength()方法返回值的,那么我们接着在看legacyExactLength方法

![](http://of1ktyksz.bkt.clouddn.com/alarmManager_legacyExactLength.png)

可以看出mAlwaysExact这个变量控制着该方法的返回值，如果是小于API19的版本会使用
WINDOW_EXACT参数，这个参数是0（意思就是区间设置为0，那么就会按照triggerAtMillis这个时间准时触发，也就是精准触发）另一个参数WINDOW_HEURISTIC的值是-1，这个值具体的用法就要看AlarmManagerService具体的实现了，反正只要知道这个值是不精准就可以。而setExact()这个值为WINDOW_EXACT,setWindow()的话这个值你可以自己传所以19以后他们是精准的.
</br>
</br>
</br>
## **相关知识**
---
本篇blog只以getService()方式举了栗子，还可通过getBroadCast()发送广播或getActivity()启动Activity来执行某项固定任务。其中各方法的最后一个参数含有以下常量分别代表不同含义的任务执行效果：

- FLAG_CANCEL_CURRENT:如果当前系统中已经存在一个相同的PendingIntent对象，那么就将先将已有的PendingIntent取消，然后重新生成一个PendingIntent对象。

- FLAG_NO_CREATE:如果当前系统中不存在相同的PendingIntent对象，系统将不会创建该PendingIntent对象而是直接返回null。

- FLAG_ONE_SHOT:该PendingIntent只作用一次。在该PendingIntent对象通过send()方法触发过后，PendingIntent将自动调用cancel()进行销毁，那么如果你再调用send()方法的话，系统将会返回一个SendIntentException。

- FLAG_UPDATE_CURRENT:如果系统中有一个和你描述的PendingIntent对等的PendingInent，那么系统将使用该PendingIntent对象，但是会使用新的Intent来更新之前PendingIntent中的Intent对象数据，例如更新Intent中的Extras。


</br>
</br>
</br>
## **总结**
---
AlarmManager非常适合Android中定时任务.并且因为他具有唤醒CPU的功能，可以保证每次需要执行特定任务时CPU都能正常工作，
或者说当CPU处于休眠时注册的闹钟会被保留(可以唤醒CPU),**(老司机们请注意此处有弯道减速慢行)**但是国内Rom众多.有的可能休眠时候无法唤醒..但是我还是推荐用AlarmManager....233333

ps:对对对,差点忘了赶紧加上...是咨询了锤哥(锤厂CTO)才知道AlarmManager也可以做定时任务的..
