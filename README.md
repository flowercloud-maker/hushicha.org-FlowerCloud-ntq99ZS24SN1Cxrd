[合集 \- Android应用开发(1\)](https://github.com)1\.【Android】谷歌应用关机闹钟 PowerOffAlarm 源码分析，并实现定时开、关机12\-14收起
## 前言


### RTC


RTC 即实时时钟（Real\-Time Clock），主要是功能有：


1. 时间保持：RTC可以在断电的时候，仍然保持计时功能，保证时间的连续性
2. 时间显示与设置：RTC可以向系统提供年、月、日、时、分、秒等信息，系统也可以通过接口校准RTC的时间保证准确性


### 关机闹钟PowerOffAlarm


PowerOffAlarm 是一个与安卓系统关机闹钟功能相关的应用或组件。
当用户设置好关机闹钟后，会向 PowerOffAlarm 发送设定关机闹钟广播并传入闹钟时间参数，PowerOffAlarm 接收到广播后，根据预设提前开机时间和闹钟时间往实时时钟（RTC）中写入时间，并将该时间写入文件中暂存
需要注意的是，PowerOffAlarm 中使用的时间默认是当前时区的时间，若传入的时间戳是其他时区的，则需要调整为当前时区的时间戳


### 时区


时区是为了适应地球自转造成的不同地区时间差异而划分的区域。
![](https://img2024.cnblogs.com/blog/3387181/202412/3387181-20241213152826255-1548066173.png)


## 正文


### 源码分析


关机闹钟 PowerOffAlarm 源码路径：`vendor/qcom/proprietary/PowerOffAlarm/src/com/qualcomm/qti/poweroffalarm`


首先查看 AndroidManifest.xml 中与关机闹钟相关的广播接收器的源代码：



```


|  |  |
| --- | --- |
|  | android:permission="org.codeaurora.permission.POWER_OFF_ALARM" |
|  | android:exported="true" |
|  | android:directBootAware="true" |
|  | android:label="PowerOffAlarmBroadcastReceiver"> |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  | android:name="com.qualcomm.qti.poweroffalarm.PowerOffAlarmDialog$ShutDownReceiver" |
|  | android:permission="org.codeaurora.permission.POWER_OFF_ALARM"> |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |


```

对于设置关机闹钟和取消关机闹钟的逻辑，跟踪到 PowerOffAlarmBroadcastReceiver 的源代码：`\vendor\qcom\proprietary\PowerOffAlarm\src\com\qualcomm\qti\poweroffalarm\PowerOffAlarmBroadcastReceiver.java`



```


|  | /******************************************************************************* |
| --- | --- |
|  | @file    PowerOffAlarmBroadcastReceiver.java |
|  | @brief   Receive "org.codeaurora.poweroffalarm.action.SET_ALARM" action to set |
|  | power off alarm and receive "org.codeaurora.poweroffalarm.action.CANCEL_ALARM" |
|  | action to cancel alarm. |
|  | ******************************************************************************/ |
|  |  |
|  | public class PowerOffAlarmBroadcastReceiver extends BroadcastReceiver { |
|  | //设置关机闹钟的动作 |
|  | private static final String ACTION_SET_POWEROFF_ALARM = "org.codeaurora.poweroffalarm.action.SET_ALARM"; |
|  | //设置取消关机闹钟的动作 |
|  | private static final String ACTION_CANCEL_POWEROFF_ALARM = "org.codeaurora.poweroffalarm.action.CANCEL_ALARM"; |
|  | //设置关机闹钟的意图携带的 extra 的 key |
|  | private static final String TIME = "time"; |
|  | //设置或取消闹钟 |
|  | if (ACTION_SET_POWEROFF_ALARM.equals(action)) {//设置闹钟动作 |
|  | long alarmTime = intent.getLongExtra(TIME, PowerOffAlarmUtils.DEFAULT_ALARM_TIME);//取出要设置的时间戳 |
|  | long alarmInPref = PowerOffAlarmUtils.getAlarmFromPreference(context);//获取之前设置的时间戳 |
|  | Log.d(TAG, "Set power off alarm : alarm time " + alarmTime + " time in pref " + alarmInPref); |
|  | PowerOffAlarmUtils.saveAlarmToPreference(context, alarmTime);//保存新的时间戳 |
|  | long alarmTimeToRtc = PowerOffAlarmUtils.setAlarmToRtc(alarmTime);//转化为 RTC 的时间 |
|  | if (alarmTimeToRtc != FAILURE) {//是个合法时间 |
|  | persistData.setAlarmTime(alarmTime); |
|  | persistData.setAlarmStatus(PowerOffAlarmUtils.ALARM_STATUS_TO_FIRE); |
|  | persistData.setSnoozeTime(PowerOffAlarmUtils.DEFAULT_ALARM_TIME); |
|  | persistData.writeDataToFile();//会把设置的 RTC 时间存在文件里面 |
|  | PowerOffAlarmUtils.saveRtcAlarmToPreference(context, alarmTimeToRtc); |
|  | } |
|  | } else if (ACTION_CANCEL_POWEROFF_ALARM.equals(action)){//取消闹钟动作 |
|  | long alarmTime = intent.getLongExtra(TIME, PowerOffAlarmUtils.DEFAULT_ALARM_TIME);//获取要取消的闹钟时间戳 |
|  | long alarmInPref = PowerOffAlarmUtils.getAlarmFromPreference(context);//获取之前设置的时间戳 |
|  | if (alarmTime == alarmInPref) {//与之前设置的时间戳相同，那就可以取消设置的闹钟 |
|  | PowerOffAlarmUtils.saveAlarmToPreference(context, PowerOffAlarmUtils.DEFAULT_ALARM_TIME); |
|  | PowerOffAlarmUtils.saveRtcAlarmToPreference(context, PowerOffAlarmUtils.DEFAULT_ALARM_TIME); |
|  | int rtc = PowerOffAlarmUtils.cancelAlarmInRtc(); |
|  | if (rtc < 0) { |
|  | Log.d(TAG, "Cancel alarm time in rtc failed "); |
|  | } |
|  | persistData.setAlarmStatus(PowerOffAlarmUtils.ALARM_STATUS_NONE); |
|  | persistData.setSnoozeTime(PowerOffAlarmUtils.DEFAULT_ALARM_TIME); |
|  | persistData.setAlarmTime(PowerOffAlarmUtils.DEFAULT_ALARM_TIME); |
|  | persistData.writeDataToFile(); |
|  | } |
|  | } |
|  | } |


```

最后追踪到 PowerOffAlarmUtils 的源代码：`\vendor\qcom\proprietary\PowerOffAlarm\src\com\qualcomm\qti\poweroffalarm\PowerOffAlarmUtils.java`



```


|  | /** |
| --- | --- |
|  | * Set alarm time to rtc register |
|  | * |
|  | * @param time alarm time based on current time (ms) |
|  | * @return set result -- Fail, return FAILURE; Success, |
|  | *         return the alarm time to rtc |
|  | */ |
|  | public static final long MS_IN_ONE_MIN = 60000L;//一分钟 |
|  | private static final long SEC_TO_MS = 1000L;//将秒转化为毫秒 |
|  |  |
|  | public static long setAlarmToRtc(long alarmTime/*未来时间的时间戳*/) { |
|  | long currentTime = System.currentTimeMillis();//当前时间的时间戳 |
|  | long alarmInRtc = getAlarmFromRtc(); |
|  | long rtcTime = getRtcTime(); |
|  | // MS_IN_ONE_MIN 是系统预留的开机时间 |
|  | long timeDelta = alarmTime - currentTime - MS_IN_ONE_MIN;//获取当前到未来的时间戳的差值（单位：ms） |
|  | // alarm in two minute is not supported |
|  | if (timeDelta <= 0) { |
|  | Log.d(TAG, "setAlarmToRtc failed: alarm time is in one minute"); |
|  | return FAILURE;//设置的时间若过短就返回失败 |
|  | } |
|  | long alarmTimeToRtc = timeDelta / SEC_TO_MS + rtcTime;//计算出要唤醒机器的 RTC 时间（单位：ms） |
|  | try { |
|  | IAlarm mProxy = IAlarm.getService(); |
|  | int ret = mProxy.setAlarm(alarmTimeToRtc);//设置 RTC 时间 |
|  | if (ret == SUCCESS) {//设置成功 |
|  | return alarmTimeToRtc; |
|  | } else { |
|  | return FAILURE;//设置失败 |
|  | } |
|  | } catch (Exception e) { |
|  | Log.d(TAG, e.toString()); |
|  | return FAILURE; |
|  | } |
|  | } |


```

对于关机广播的源代码，首先追踪到源代码：`\vendor\qcom\proprietary\PowerOffAlarm\src\com\qualcomm\qti\poweroffalarm.java`



```


|  | public class PowerOffAlarmDialog  extends Activity{ |
| --- | --- |
|  | public static class ShutDownReceiver extends BroadcastReceiver { |
|  | @Override |
|  | public void onReceive(final Context context, Intent intent) { |
|  | PowerOffAlarmUtils.powerOff(context);//关机 |
|  | } |
|  | } |
|  | } |


```

随后追踪到 PowerOffAlarmUtils.java 中：



```


|  | private static final String ACTION_REQUEST_SHUTDOWN = "com.android.internal.intent.action.REQUEST_SHUTDOWN"; |
| --- | --- |
|  |  |
|  | public static void powerOff(Context context) { |
|  | //想要直接使用这个函数的话，app必须是系统app，且具有如下权限： |
|  | // |
|  | Intent requestShutdown = new Intent(ACTION_REQUEST_SHUTDOWN); |
|  | requestShutdown.putExtra(EXTRA_KEY_CONFIRM, false);//true 是弹窗询问，false 是不弹窗直接关机 |
|  | requestShutdown.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK); |
|  | context.startActivity(requestShutdown); |
|  | } |


```

### 项目实战


**设置关机闹钟**



```


|  | public static final String POWER_OFF_ALARM_PACKAGE = "com.qualcomm.qti.poweroffalarm"; |
| --- | --- |
|  | private static final long MS_IN_ONE_MIN = 6000L; |
|  |  |
|  | /** |
|  | * 设置关机闹钟，在到达预定的时间戳时开机 |
|  | * |
|  | * @param mContext 上下文 |
|  | * @param millis 未来的时间戳 |
|  | */ |
|  | public static void startPowerOnAlarm(Context mContext, long millis) {//开机 |
|  | Intent intent = new Intent(Constants.ACTION_SET_POWEROFF_ALARM); |
|  | intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND); |
|  | intent.setPackage(POWER_OFF_ALARM_PACKAGE); |
|  | long time = millis + MS_IN_ONE_MIN; |
|  | intent.putExtra("time", time); |
|  | mContext.sendBroadcast(intent); |
|  | } |


```

**取消关机闹钟**



```


|  | public static final String ACTION_CANCEL_POWEROFF_ALARM = "org.codeaurora.poweroffalarm.action.CANCEL_ALARM"; |
| --- | --- |
|  |  |
|  | /** |
|  | * 取消关机闹钟，取消设定在时间戳 millis 的闹钟 |
|  | * |
|  | * @param mContext 上下文 |
|  | * @param millis 取消的闹钟时间戳 |
|  | */ |
|  | public static void cancelAlarm(Context mContext, long millis) { |
|  | Intent intent = new Intent(ACTION_CANCEL_POWEROFF_ALARM); |
|  | intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND); |
|  | intent.setPackage(POWER_OFF_ALARM_PACKAGE); |
|  | millis += MS_IN_ONE_MIN; |
|  | intent.putExtra("time", millis); |
|  | sendBroadcast(intent); |
|  | } |


```

**关机功能**



```


|  | public static void startShutDown(Context mContext) {//关机 |
| --- | --- |
|  | Intent intent = new Intent("org.codeaurora.poweroffalarm.action.ALARM_POWER_OFF"); |
|  | intent.setPackage(POWER_OFF_ALARM_PACKAGE); |
|  | intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK); |
|  | mContext.sendBroadcast(intent); |
|  | //也可以使用如下代码实现，但必须是系统应用且带有如下权限: |
|  | // |
|  | //        String action = "com.android.internal.intent.action.REQUEST_SHUTDOWN"; |
|  | //        if(Build.VERSION.SDK_INT <= Build.VERSION_CODES.N){ |
|  | //            action = "android.intent.action.ACTION_REQUEST_SHUTDOWN"; |
|  | //        } |
|  | //        Intent intent = new Intent(action); |
|  | //        intent.putExtra("android.intent.extra.KEY_CONFIRM", false); |
|  | //        intent.setFlags(Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS); |
|  | //        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK); |
|  | //        mContext.startActivity(intent); |
|  | } |


```

 本博客参考[milou加速器](https://jiechuangmoxing.com)。转载请注明出处！
