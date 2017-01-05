>临近放假了时间比较充裕,就琢磨着干点啥,想来想去6.0运行时权限一直没弄过而且项目也总有一天要升级的,既然这样,那就趁热来一发吧

那么为了学到原滋原味的6.0权限,官方文档是第一选择,这里附上链接https://developer.android.com/training/permissions/index.html

关于6.0如何选择compileSdkVersion, minSdkVersion 和 targetSdkVersion可以看我另一篇博文
[如何选择compileSdkVersion, minSdkVersion 和 targetSdkVersion](http://blog.csdn.net/zly921112/article/details/52648486)

# 6.0权限介绍
在文档上可以看到6.0在权限上有几处改变

1. 系统权限分为两类

  1. 普通权限

	正常的权限不涉及用户的隐私风险。如果你的应用程序清单列出了一个正常的权限,系统自动授予许可。

  2. 危险权限

    危险的权限可以让应用程序访问用户的机密数据,如果列出一个危险的权限,用户必须授权许可

2. 如果设备运行Android 5.1或更低,或应用程序的targetSdk是22或更低:你在清单列表申明的危险权限,用户授予权限许可是在**安装**应用程序时.(测试时候发现部分国产rom的手机在低于6.0的版本依旧是运行时授予确实强大^_^)
如果设备运行Android 6.0或更高版本,并且你的应用程序的targetSdk是23或更高:应用中危险的权限,将在应用程序**运行**时用户授予。(我用的6.0系统的华为v8测试的,如果程序的targetSdk是23或者更高,当需要使用权限的时候,系统并未自己弹起权限请求,需要自己写代码请求才行)

这里列出危险权限

![](http://img.blog.csdn.net/20160928223436914)



# API介绍

## 1.检查权限是否获取

```
// Assume thisActivity is the current activity
int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,
        Manifest.permission.WRITE_CALENDAR);
```

![](http://img.blog.csdn.net/20160929214633461)

如果返回值 PackageManager.PERMISSION_GRANTED表示权限已经获取,可以执行操作.如果返回值 PackageManager.PERMISSION_DENIED表示权限未授予,需要请求权限

## 2.是否需要解释为什么请求该权限

```
ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,Manifest.permission.WRITE_CALENDAR)
```

![](http://img.blog.csdn.net/20160929223305074)

在用户拒绝过该权限并且未勾选不在提示,再次请求该权限的时候shouldShowRequestPermissionRationale方法将会返回true,这个时候可以给用户一个获取该权限的解释,比如弹出一个说明.

## 3.请求权限

```
ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);
```

![](http://img.blog.csdn.net/20160929215134656)

调用该方法后.系统会显示一个标准的对话框给用户,应用程序不能配置或者改变对话框的内容,即请求权限弹窗**不能自定义**.举个栗子,如果你请求READ_CONTACTS许可,该系统对话框只是说你的应用程序需要访问设备的联系人。一旦用户同意。那么**同权限组的其他权限,系统自动赋予他们,不在弹窗提示**。

此外,权限组的划分在未来的安卓版本可能会改变,所以每一个需要权限的地方都需要请求权限,即使用户已经允许另一个同组的权限。



## 4.结果回调

```
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {


}
```
当系统要求用户授予权限,用户可以选择不在询问或者不在提醒,这种情况下系统不会弹出请求权限弹窗,而是直接回调onRequestPermissionsResult方法并且结果为PackageManager.PERMISSION_DENIED。

用户拒绝了权限并且勾选了不在询问这种情况,可以在onRequestPermissionsResult回调方法中判断,即返回结果为PackageManager.PERMISSION_DENIED并且该权限的shouldShowRequestPermissionRationale方法返回值为false

```
@Override
    public void onRequestPermissionsResult(int requestCode,
                                           String permissions[], int[] grantResults) {
        if(grantResults[0] == PackageManager.PERMISSION_DENIED &&
                !ActivityCompat.shouldShowRequestPermissionRationale(this,permissions[0])){

        }
    }
```
因为这种情况下没法弹出系统请求权限弹窗,所以我们需要手动的写一个dialog去引导用户到设置界面打开权限.

这里附上文档中的例子

```
// Here, thisActivity is the current activity
if (ContextCompat.checkSelfPermission(thisActivity,
                Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {

        // Show an expanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.

    } else {

        // No explanation needed, we can request the permission.

        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
}

@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
            // If request is cancelled, the result arrays are empty.
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // permission was granted, yay! Do the
                // contacts-related task you need to do.

            } else {

                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return;
        }

        // other 'case' lines to check for other
        // permissions this app might request
    }
}
```

 api是不是很简单很方便.但是实际项目中我还是使用第三方库 哈哈哈

--------------------------------------------------------------

关于6.0权限的库还是很多的..这里我使用的是google官方提供的 [easypermissions](https://github.com/googlesamples/easypermissions)

使用比较简单这里我稍微讲解下.
1.流程图

![](http://img.blog.csdn.net/20161001093028106)

2.sample中代码稍加注释提示下
```
public class MainActivity extends AppCompatActivity implements
        EasyPermissions.PermissionCallbacks {

    private static final String TAG = "MainActivity";

    private static final int RC_CAMERA_PERM = 123;
    private static final int RC_SETTINGS_SCREEN = 125;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Button click listener that will request one permission.
        findViewById(R.id.button_camera).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                cameraTask();
            }
        });

    }

	//用户手动修改权限请求回调
    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode == RC_SETTINGS_SCREEN) {
            // Do something after user returned from app settings screen, like showing a Toast.
            Toast.makeText(this, R.string.returned_from_app_settings_to_activity, Toast.LENGTH_SHORT)
                    .show();
        }
    }

    @AfterPermissionGranted(RC_CAMERA_PERM)
    public void cameraTask() {
	    //判断权限是否授予
        if (EasyPermissions.hasPermissions(this, Manifest.permission.CAMERA)) {//已经授予
            // Have permission, do the thing!
            Toast.makeText(this, "TODO: Camera things", Toast.LENGTH_LONG).show();
        } else {//未授予
            // Ask for one permission
            EasyPermissions.requestPermissions(this, getString(R.string.rationale_camera),
                    RC_CAMERA_PERM, Manifest.permission.CAMERA);
        }
    }



    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);

        // EasyPermissions handles the request result.
        EasyPermissions.onRequestPermissionsResult(requestCode, permissions, grantResults, this);//权限回调传给EasyPermissions.onRequestPermissionsResult(),他会根据权限获取成功与否回调onPermissionsGranted()与onPermissionsDenied
    }

	//获取权限成功回调
    @Override
    public void onPermissionsGranted(int requestCode, List<String> perms) {
        Log.d(TAG, "onPermissionsGranted:" + requestCode + ":" + perms.size());
    }

	//获取权限失败回调
    @Override
    public void onPermissionsDenied(int requestCode, List<String> perms) {
        Log.d(TAG, "onPermissionsDenied:" + requestCode + ":" + perms.size());

        // 判断是否拒绝了权限请求并且勾选了不在提示,如果是将会弹出引导手动设置权限的dialog,引导用户到设置中心手动修改权限
        if (EasyPermissions.somePermissionPermanentlyDenied(this, perms)) {
            new AppSettingsDialog.Builder(this, getString(R.string.rationale_ask_again))
                    .setTitle(getString(R.string.title_settings_dialog))
                    .setPositiveButton(getString(R.string.setting))
                    .setNegativeButton(getString(R.string.cancel), null /* click listener */)
                    .setRequestCode(RC_SETTINGS_SCREEN)
                    .build()
                    .show();
        }
    }
}
```

到此基本就介绍的差不多啦.有什么疑问欢迎留言,今天是祖国母亲生日的第一天,祝大家国庆快乐
