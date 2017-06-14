# 弹幕 SDK 及 DEMO 使用说明

## 一 demo

### 流程
1. 首先需要用自己的账户密码登录
2. 选择测试接口, 则进入接口测试界面
3. 选择界面测试, 进入弹幕界面测试
    * 弹幕界面测试仅包含收弹幕, 不包含发弹幕
    * 在操作过程中暂无操作提示
    * 点播channelId可以选择 25535757, 直播可以尝试300451. 
    * 只有点击了"开始播放" 才会开始接收弹幕
    * 20170613 增加了透明度调整和全屏/半屏功能
    * 20170614 增加了弹幕显示半屏, 以及无遮挡功能

## 二 SDK

## 0. 接口返回代码

类型 | 解释
--- | ---
httpStatus | 200 正确，201 创建成功,401 权限验证失败, 422 :请求非法, 404 未找到，500系统错误

body | err有0,401,404,422,500.分别表示成功、权限失败、NotFound、请求非法、系统错误

## 1. 初始化
### 1.1 AndroidManifest
增加如下配置
```xml
<meta-data
  android:name="receiveppaction"
  android:value="com.pptv.ppcloud.danmaku.receiver" />
<!-- yourreceiver action -->

<meta-data
  android:name="pushppid"
  android:value="" />

<meta-data
  android:name="pushppkey"
  android:value="" />

<!-- 弹幕接收器 -->
<receiver android:name="com.pptv.ppcloud.danmaku.DanmakuReceiver" >
  <intent-filter>
      <action android:name="com.pptv.ppcloud.danmaku.receiver" />
  </intent-filter>
</receiver>

<!-- 推送服务 -->
<service
  android:name="com.pplive.pushmsgsdk.SdkMainService"
  android:exported="true">
  <intent-filter>
      <action android:name="com.pplive.pushmsgsdk.SdkMainService" />
  </intent-filter>
</service>

<receiver android:name="com.pplive.pushmsgsdk.SdkReceiver" >
  <intent-filter>
      <action android:name="android.intent.action.BOOT_COMPLETED" />
      <action android:name="android.intent.action.USER_PRESENT" />
      <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
      <action android:name="com.pplive.pushmsgsdk.KEEP_ALIVE"/>
      <action android:name="com.pplive.pushmsgsdk.SYNC_DATA"/>
  </intent-filter>
</receiver>
```

```pushppkey``` 和```pushppid```由后台分配给对应端

接收弹幕的Receiver需要增加接收的Action, 此action需跟 ```receiveppaction```中定义的一致.

如果想要自定义弹幕接收的BroadcastReceiver, 则需要继承```com.pptv.ppcloud.danmaku.DanmakuReceiver```这个类进行复写, 并在AndroidManifest文件中声明.

如:

```JAVA
public class MyReceiver extends DanmakuReceiver {

    @Override
    protected void deliver(ArrayList<DanmakuModel> models) {
        super.deliver(models);
        System.out.println(models);
    }
}
```

### 1.2 代码中初始化
所有与弹幕相关操作的类为```DanmakuSDK```, 而其初始化工具是```DanmakuInitializer```类. 初始化方法如下:

```JAVA
//初始化, context, 当前平台枚举, 当前软件版本号
DanmakuInitializer.begin(this, DanmakuSDK.Platform.Phone, "5.0.0")
                
                //必须设置, 可以是前端的序列化库调用. 比如Gson
                .setJsonSerializer(new IDanmakuJSONSerializer() {
                    @Override
                    public String serializeToString(Object object) {
                        return new Gson().toJson(object);
                    }

                    @Override
                    public <T> T from(String json, Class<T> t) {
                        return new Gson().fromJson(json, t);
                    }
                })
                
                //设置直播弹幕socket的接收器. 可以不设置, 如果不设置, 则直播弹幕在DanmakuReceiver及其子类中接收
                .setDanmakuDeliverListener(new IDanmakuDeliverListener() {
                    @Override
                    public void onReceived(ArrayList<DanmakuModel> danmakus) {
                        System.out.println(danmakus);
                    }
                }).complete();
```

弹幕 SDK 初始化建议放在```Application.onCreate```中.

## 2. 调用方法

### 2.1 发送

接口声明如下:

```JAVA
/**
* 发送弹幕
* @param token
* @param username
* @param isLive
* @param channelId 视频或直播流id
* @param content 内容, 50字以内
* @param positionInMilliseconds 位置的毫秒, 精确到100ms即可
* @param callback 成功时, 会返回对应的弹幕id
*/
public static void send(@NonNull String token,
                       @NonNull String username,
                       boolean isLive,
                       @NonNull String channelId,
                       @NonNull String content,
                       long positionInMilliseconds,
                       Callback<String> callback);
```

### 2.2 点播拉取弹幕列表

声明如下

```JAVA
/**
     * 获取列表
     * 对于点播弹幕：第一次和用户跳转请求的pos必须是1000个单位的整数倍，往前最近的一个位置。第二次请求按照我给的end作为pos。
     * 对于直播弹幕：
     * 第一次请求直播弹幕，传入参数f=1和pos=自然时间(play返回)。后台返回（自然时间-10s）到（自然时间）的弹幕，并告诉end和ts=10s。
     * 前端根据上一次请求的时间，在ts之后做第二次请求，传入pos={end}。我返回（pos）到(pos+10s)的弹幕，并告诉新的end和ts。
     * 反复请求。如果出现end时间大于某个产品不能接受的值（一定大于我给的ts）。做第一次请求逻辑—>第二次—>….
     * 直播回看时，保证pos是1000个单位的倍数。
     *
     * @param isLive 是否是直播
     * @param channelId 视频或直播流id
     * @param positionInMilliseconds 默认为0, 从头开始, 单位毫秒, 播放位置
     * @param f 第一次请求传入 0 或者不传，1表示后面的直播请求 2表示直播回看
     */
    public static void getList(boolean isLive, String channelId, long positionInMilliseconds, String f, final Callback<ApiDanmuListBean> callback)
```

获取列表功能 __仅用于点播__ .

接口返回 ```ApiDanmuListBean```类, 声明如下:

```JAVA
public class ApiDanmuListBean {
    public String name;
    public ArrayList<ApiDanmuBean> infos;

    /**
     *  开始时间点, 0.1s
     */
    public int start;

    /**
     * 结束点, 0.1s
     */
    public int end;

    /**
     * 下次拉取的间隔时间。单位0.1s
     */
    public int ts;
}
```

这里由于后端接口定义原因, ts, end, start 返回值单位都是时间戳 0.1秒, 如果要编程毫秒单位, 需要乘100

infos 中的数据可以通过 ```getDanmakuModel()``` 方法转换成接口无关的```DanmakuModel```类

### 2.3 直播收取弹幕
直播弹幕通过TCP Socket和服务器进行通讯. 因此可以通过声明或者继承```DanmakuReceiver```接收弹幕, 或者通过初始化时传入的 ```IDanmakuDeliverListener``` 类来接收.

当需要接收某个直播弹幕时, 需要注册该直播的channelId

```JAVA
DanmakuSDK.getInstance().registerDanmakuSocket(context, true, channelId);
```

当需要停止接收弹幕或者退出activity时, 则调用如下方法:

```JAVA
DanmakuSDK.getInstance().unregisterDanmakuSocket(context);

```

或者
 
```JAVA
DanmakuSDK.getInstance().unregisterDanmakuSocket(context, channelId);
```

进行解除绑定

## 3. 退出 APP
由于弹幕 sdk 当中含有线程池, 长连接等资源, 因此在退出app时, 需要调用 ```DanmakuSDK.getInstance().onAppTerminal(context);``` 进行资源的释放

## 4. 弹幕网络接口文档:

[http://sharepoint/tech/clouddivision/_layouts/15/WopiFrame2.aspx?sourcedoc=/tech/clouddivision/Shared%20Documents/%E9%95%BF%E8%BF%9E%E6%8E%A5%E4%BB%A5%E5%8F%8A%E5%BC%B9%E5%B9%95%E6%A2%B3%E7%90%86/%E5%BC%B9%E5%B9%951.0%E6%8E%A5%E5%8F%A3%E6%96%87%E6%A1%A3.docx&action=default](http://sharepoint/tech/clouddivision/_layouts/15/WopiFrame2.aspx?sourcedoc=/tech/clouddivision/Shared%20Documents/%E9%95%BF%E8%BF%9E%E6%8E%A5%E4%BB%A5%E5%8F%8A%E5%BC%B9%E5%B9%95%E6%A2%B3%E7%90%86/%E5%BC%B9%E5%B9%951.0%E6%8E%A5%E5%8F%A3%E6%96%87%E6%A1%A3.docx&action=default)


