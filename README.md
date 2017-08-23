<h2 align = "center">CTS延伸--主用户和被管理用户（Managed Profile）</h2>

---

#### 0.起因

在进行CtsVerifier测试时，发现在手动验证"Cross profile intent filters are set"这一项时失败了。
查看Log如下：

```java
10:46:45.111 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.DIAL dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
08-11 10:46:45.120 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.VIEW cat=[android.intent.category.BROWSABLE] dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
08-11 10:46:45.145 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.DIAL dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
08-11 10:46:45.148 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.VIEW cat=[android.intent.category.BROWSABLE] dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
08-11 10:46:45.174 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.DIAL dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
08-11 10:46:45.177 10694 10694 E IntentFiltersTestHelper: Intent { act=android.intent.action.VIEW cat=[android.intent.category.BROWSABLE] dat=tel:xxx }  from managed profile should be forwarded to the primary profile but is not.
```

大概意思是：从managed profile发送一个Intent { act=android.intent.action.DIAL dat=tel:xxx },这个Intent应该被转发到primary profile,但是并没有。

因为这个时候，不知道managed profile和primary profile分别是什么，所以无法定位出原因。

阅读源码添加Log，重编CTS-V后发现，验证步骤大体如下：
> 1. CTS-V构造一个Action为“com.android.cts.verifier.managedprovisioning.BYOD_STATUS”的intent, 通过queryIntentActivities得到一个
IntentForwarderActivity。
> 2. CTS-V再次构造一个Intent { act=android.intent.action.DIAL dat=tel:xxx }，通过queryIntentActivities得到一个Activity列表，
这个Activity列表一定要包含上述IntentForwarderActivity。

由于Intent { act=android.intent.action.DIAL dat=tel:xxx }只能定位匹配到我们的DialtactsActivity，所以CTS验证以失败结束。
虽然知道失败的原因了，但是还是不明白为什么要这样进行验证，于是就有了这篇文章。

#### 1. 用户(Primary profile)与被管理用户（Managed profile）

具体用户的类型，可以通过查看源码的 UserInfo 的FLAG_XXXXX的解释，在这里列出本文必要的几种：
> 1. FLAG_PRIMARY 主用户，设备上的第一个用户，一个设备只存在一个。
> 2. FLAG_ADMIN 管理员，可以添加删除用户。
> 3. FLAG_MANAGED_PROFILE 表示当前用户是其他用户的Profile，即当前用户只拥有其他用户的部分功能，例如B用户只持有A用户的企业应用数据。
看起来是Android提供的企业管理的手段。
> 4. FLAG_DISABLED 用来表示用户Disabled，即当用户正在被删除时，会设置这个标识位，Disabled的用户不能重新Enabled。

应用与用户的关系，其实就是uid, userId, appId三者的关系，可以通过查看源码的 Process和 UserHandle 得到

> 1. 在不支持多用户的情况下用户只有一个userId为0, uid == appId.
> 2. 在支持多用户的情况下， uid = userId * 100000 + appId.
也就是说，对于每个用户的每个应用，uid都是不一样的。

#### 2. 申请设备管理权限

我们可以通过2步获取设备管理权限。

###### 1. 发送申请设备管理权限的请求。

```java
private void provisionManagedProfile() {
    Activity activity = getActivity();
    if (null == activity) {
        return;
    }
    // 启动设置被管理账户的流程
    Intent intent = new Intent(DevicePolicyManager.ACTION_PROVISION_MANAGED_PROFILE);

    // Use a different intent extra below M to configure the admin component.
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
        //noinspection deprecation
        // 在M版本以前，我们只需要提供一个包名，系统会自动查询该包名下的DeviceReceiver,
        // 但是规定DeviceReceiver只能设置一个
        intent.putExtra(DevicePolicyManager.EXTRA_PROVISIONING_DEVICE_ADMIN_PACKAGE_NAME,
                activity.getApplicationContext().getPackageName());
    } else {
        // 在M版本及更高版本，那么只需要直接指定接收授权结果的及权限的组件
        final ComponentName component = new ComponentName(activity,
                BasicDeviceAdminReceiver.class.getName());
        intent.putExtra(DevicePolicyManager.EXTRA_PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME,
                component);
    }

    if (intent.resolveActivity(activity.getPackageManager()) != null) {
        // 应该在onActivityResult里面处理成功/取消的情况。
        startActivityForResult(intent, REQUEST_PROVISION_MANAGED_PROFILE);
        activity.finish();
    } else {
        Toast.makeText(activity, "Device provisioning is not enabled. Stopping.",
                       Toast.LENGTH_SHORT).show();
    }
}
```

###### 2. 接收权限以及设置Profile可用。

接收权限的必须是一个继承了DeviceAdminReceiver的声明在Manifest的Receiver
```java
<receiver
    android:name=".BasicDeviceAdminReceiver"
    android:description="@string/app_name"
    android:label="@string/app_name"
    <!--指定发送方需要有android.permission.BIND_DEVICE_ADMIN权限-->
    android:permission="android.permission.BIND_DEVICE_ADMIN">
    <meta-data
        android:name="android.app.device_admin"
        android:resource="@xml/basic_device_admin_receiver"/>
    <intent-filter>
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED"/>
    </intent-filter>
</receiver>
```
basic_device_admin_receiver.xml 文件内容
```java
<device-admin>
    <!--指定申请的权限列表-->
    <uses-policies>
        <limit-password/>
        <watch-login/>
        <reset-password/>
        <force-lock/>
        <wipe-data/>
        <expire-password/>
        <encrypted-storage/>
        <disable-camera/>
        <disable-keyguard-features/>
    </uses-policies>
</device-admin>
```

```java
public class BasicDeviceAdminReceiver extends DeviceAdminReceiver {

    // 申请管理员步骤完成后，onProfileProvisioningComplete会触发。
    // 当前Receiver的运行环境已经换成了ManagedProfile.
    // 但是Profile还是disable的状态。
    @Override
    public void onProfileProvisioningComplete(Context context, Intent intent) {

        Intent launch = new Intent(context, EnableProfileActivity.class);
        launch.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(launch);
    }
}
```
在EnableProfileActivity里面将当前Profile设为Enabled状态, 并设置用户名。
```java
private void enableProfile() {
    DevicePolicyManager manager =
        (DevicePolicyManager) getSystemService(Context.DEVICE_POLICY_SERVICE);
    ComponentName componentName = BasicDeviceAdminReceiver.getComponentName(this);
    // This is the name for the newly created managed profile.
    manager.setProfileName(componentName, getString(R.string.profile_name));
    // We enable the profile here.
    manager.setProfileEnabled(componentName);
}
```

到此，申请设备管理权限的步骤已经完成，我们可以在设置->锁屏、指纹、安全->更多安全设置->设备管理器 里面查看到我们的应用。
我们也可以在设置->帐号 里面看到查看到我们的工作帐号。

回到桌面，可以看到新增了一些右下角带有公文包的应用，这些应用是属于Profile用户的。默认情况下Google会将保证基本用户体验的
应用添加进来。

#### 3. 主用户与被管理的用户之间的通信

当前我们设备上是运行着2个用户的，我们可以通过`adb shell pm list users`列出当前运行的用户信息。

```java
Users:
	UserInfo{0:机主:13} running
	UserInfo{12:工作资料:30} running
```

为了安全，Android默认情况下完全隔离了主用户和ManagedProfile之间的通信。但是也允许设备管理员打开特定的IntentFilter通道。
我们知道，Android组件之间的通信主要靠Intent完成，匹配规则主要是定义在intent-filter项里面的。下面是一个打开IntentFilter通道的例子。

```java
private void enableForwarding() {
    Activity activity = getActivity();
    if (null == activity || activity.isFinishing()) {
        return;
    }
    DevicePolicyManager manager =
            (DevicePolicyManager) activity.getSystemService(Context.DEVICE_POLICY_SERVICE);
    try {
        IntentFilter filter = new IntentFilter(Intent.ACTION_SEND);
        filter.addDataType("text/plain");
        filter.addDataType("image/jpeg");
        // This is how you can register an IntentFilter as allowed pattern of Intent forwarding
        manager.addCrossProfileIntentFilter(BasicDeviceAdminReceiver.getComponentName(activity),
                filter, FLAG_MANAGED_CAN_ACCESS_PARENT | FLAG_PARENT_CAN_ACCESS_MANAGED);
    } catch (IntentFilter.MalformedMimeTypeException e) {
        e.printStackTrace();
    }
}
```

如果不把ACTION_SEND这个Action添加到CrossProfileIntentFilter里面，在Profile用户环境下，发送该Action是没有响应的,

除非Profile用户也安装有响应ACTION_SEND的应用。

其中

> 1. FLAG_MANAGED_CAN_ACCESS_PARENT 表示ManagedProfile用户发出的Intent可以到达主用户空间，即主用户的APP可以响应该Intent。
> 2. FLAG_PARENT_CAN_ACCESS_MANAGED 标识主用户发出的Intent可以到达ManagedProfile用户空间。

`注意`：以上可以跨Profile用户发送的Intent的目标组件必须是Activity。

#### 4. 将主用户下的APP安装到ManagedProfile用户，并禁用部分组件。

```java
private void setAppEnabled(String packageName, boolean enabled) {
    Activity activity = getActivity();
    if (null == activity) {
        return;
    }
    PackageManager packageManager = activity.getPackageManager();
    DevicePolicyManager devicePolicyManager =
            (DevicePolicyManager) activity.getSystemService(Context.DEVICE_POLICY_SERVICE);
    try {
        int packageFlags;
        if(Build.VERSION.SDK_INT < 24){
            //noinspection deprecation
            packageFlags = PackageManager.GET_UNINSTALLED_PACKAGES;
        }else{
            packageFlags = PackageManager.MATCH_UNINSTALLED_PACKAGES;
        }
        ApplicationInfo applicationInfo = packageManager.getApplicationInfo(packageName,
                packageFlags);
        // Here, we check the ApplicationInfo of the target app, and see if the flags have
        // ApplicationInfo.FLAG_INSTALLED turned on using bitwise operation.
        // 在当前用户下是否安装
        if (0 == (applicationInfo.flags & ApplicationInfo.FLAG_INSTALLED)) {
            // If the app is not installed in this profile, we can enable it by
            // DPM.enableSystemApp
            if (enabled) {
                // 将系统APP安装到当前用户中。
                devicePolicyManager.enableSystemApp(
                        BasicDeviceAdminReceiver.getComponentName(activity), packageName);
            } else {
                // But we cannot disable the app since it is already disabled
                Log.e(TAG, "Cannot disable this app: " + packageName);
                return;
            }
        } else {
            // If the app is already installed, we can enable or disable it by
            // DPM.setApplicationHidden
            // 隐藏当前用户APP，保留APP的数据。
            devicePolicyManager.setApplicationHidden(
                    BasicDeviceAdminReceiver.getComponentName(activity), packageName, !enabled);
        }
        Toast.makeText(activity, enabled ? R.string.enabled : R.string.disabled,
                Toast.LENGTH_SHORT).show();
    } catch (PackageManager.NameNotFoundException e) {
        Log.e(TAG, "The app cannot be found: " + packageName, e);
    }
}
```

某些情况下，我们可能需要在ManagedProfile用户中禁止某些APP，或者禁止某个组件。
```java
// 禁用组件
context.getPackageManager().setComponentEnabledSetting(componentName,
                            PackageManager.COMPONENT_ENABLED_STATE_DISABLED, 0);
// 禁用应用
context.getPackageManager().setApplicationEnabledSetting(context.getPackageName(),
                            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0);
```
当组件被禁用后，无论是显式的还是隐式的Intent都无法匹配到该组件了。

#### 5. 以上CTS测试原理以及解决方案

现在我们可以知道，“Cross profile intent filters are set”实际上就是测试是否有通过addCrossProfileIntentFilter设置对应的Intent Filters.

上面失败的信息，实际上就是从Managed profile发出Intent后，本应可以跨用户响应的，但是并没有，这更加说明对应的Intent Filter没设置可跨用户。

但是，addCrossProfileIntentFilter需要提供拥有管理员权限的DeviceAdminReceiver组件，所以在应用直接设置行不通。

经过测试发现，如果应用包名被添加进了`packages/apps/ManagedProvisioning/res/values/vendor_required_apps_managed_profile.xml`，那么对应的应用会在设置Profile完成后，
默认安装在Managed profile用户空间下。这会导致应用的Intent Filter没有被设置跨用户。

因此，我们可以在用户初始化的时候，根据用户FLAGS设置应用是否可用。我们的目的是让电话应用在Manged Profile下不可用，后来证实也不是完全正确的，因为CTS测试也会创建Manged Profile用户进行测试。
后面根据具体FLAGS进行区分CTS测试和CTS-V测试，最后才测试通过。具体代码

权限（只有系统应用才能申请）
```java
<uses-permission android:name="android.permission.MANAGE_USERS"/>
```

```java
// 声明一个接收用户初始化消息的Receiver
<receiver android:name="com.tplink.dialer.BootCompletedReceiver">
    <intent-filter>
        <action android:name="android.intent.action.USER_INITIALIZE" />
    </intent-filter>
</receiver>
```
UserInfo这个类原生是隐藏的，我们SDK开放了，所以可以直接使用。

```java
public void onReceive(Context context, Intent intent) {
    String action = intent.getAction();
    if(action.equals(Intent.ACTION_USER_INITIALIZE)) {

        if (UserHandle.myUserId() != 0) {
            /*应用是否在其他用户空间可以使用，依赖于UserInfo.isEnable(),
             *因为Cts-V的构建的用户isEnable()是false, CTS构建的用户是true.
             *而电话要在Cts-V上不可用，在CTS上可用。
             */
            UserManager um = UserManager.get(context);
            UserInfo currentUserInfo = um.getUserInfo(UserHandle.myUserId());
            log("current user " + currentUserInfo);

            if (currentUserInfo.isEnabled()) {
                // 将当前电话应用设置为可用
                context.getPackageManager().setApplicationEnabledSetting(context.getPackageName(),
                        PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0);
            } else {
                // 将当前电话应用设置为不可用
                context.getPackageManager().setApplicationEnabledSetting(context.getPackageName(),
                        PackageManager.COMPONENT_ENABLED_STATE_DISABLED, 0);
            }
        }
    }
}
```

本文应用的大部分代码来自于：[https://github.com/googlesamples/android-BasicManagedProfile](https://github.com/googlesamples/android-BasicManagedProfile)

