>- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
>- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
>- 作者：[Melo](https://itsmelo.github.io/)
>- 审阅者：[]()

**写在前面：**

Android M 中有一个比较重要的知识点就是运行时权限了，关于运行时权限的文章和封装库目前也出现了不少，在开发的过程中始终觉得运行时权限这块的代码可以进一步封装一下，让其使用起来能再简洁一点，并且还不容易出错。关于 Android 运行时权限的知识就不多讲解了，可以参考下面的一些资料：

[Android Runtime Permissions](https://developer.android.com/training/permissions/requesting.html)

[Android 6.0 运行时权限处理完全解析](http://blog.csdn.net/lmj623565791/article/details/50709663)

知识点比较简单，大致了解和使用过后，就来开始我的这次封装之旅吧。

现在直观的看看封装之后的使用，是不是清爽很多？

```
    @OnClick(R.id.tv_toolbar_right)
    public void onClick() {
        performRequestPermissions(getString(R.string.permission_desc), new String[]{Manifest.permission.READ_PHONE_STATE, Manifest.permission.ACCESS_COARSE_LOCATION}
                , PER_REQUEST_CODE, new PermissionsResultListener() {
                    @Override
                    public void onPermissionGranted() {
                        Toast.makeText(MainActivity.this, "已申请权限", Toast.LENGTH_LONG).show();
                    }

                    @Override
                    public void onPermissionDenied() {
                        Toast.makeText(MainActivity.this, "拒绝申请权限", Toast.LENGTH_LONG).show();
                    }
                });
    }
```
我将运行时权限封装到 BaseActivity 中，MainActivty 继承BaseActivity，调用 performRequestPermissions 方法，此时我不用再去做一些判断，只需要穿入四个参数即可。当然你为了在 6.0 以下保证应用正常运行，你依然需要像以前一样，在**清单文件**中申明要使用的权限。

首先我需要一个接口做通信：

```
public interface PermissionsResultListener {

    void onPermissionGranted();

    void onPermissionDenied();

}
```

然后把逻辑代码封装在 BaseActivity 里面：

```
public class BaseActivity extends AppCompatActivity {

    private PermissionsResultListener mListener;

    private int mRequestCode;

    /**
     * 其他 Activity 继承 BaseActivity 调用 performRequestPermissions 方法
     *
     * @param desc        首次申请权限被拒绝后再次申请给用户的描述提示
     * @param permissions 要申请的权限数组
     * @param requestCode 申请标记值
     * @param listener    实现的接口
     */
    protected void performRequestPermissions(String desc, String[] permissions, int requestCode, PermissionsResultListener listener) {
        if (permissions == null || permissions.length == 0) {
            return;
        }
        mRequestCode = requestCode;
        mListener = listener;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (checkEachSelfPermission(permissions)) {// 检查是否声明了权限
                requestEachPermissions(desc, permissions, requestCode);
            } else {// 已经申请权限
                if (mListener != null) {
                    mListener.onPermissionGranted();
                }
            }
        } else {
            if (mListener != null) {
                mListener.onPermissionGranted();
            }
        }
    }

    /**
     * 申请权限前判断是否需要声明
     *
     * @param desc
     * @param permissions
     * @param requestCode
     */
    private void requestEachPermissions(String desc, String[] permissions, int requestCode) {
        if (shouldShowRequestPermissionRationale(permissions)) {// 需要再次声明
            showRationaleDialog(desc, permissions, requestCode);
        } else {
            ActivityCompat.requestPermissions(BaseActivity.this, permissions, requestCode);
        }
    }

    /**
     * 弹出声明的 Dialog
     *
     * @param desc
     * @param permissions
     * @param requestCode
     */
    private void showRationaleDialog(String desc, final String[] permissions, final int requestCode) {
        final AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle(getString(R.string.tips))
                .setMessage(desc)
                .setPositiveButton(getResources().getString(R.string.confrim), new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i) {
                        ActivityCompat.requestPermissions(BaseActivity.this, permissions, requestCode);
                    }
                })
                .setNegativeButton(getResources().getString(R.string.cancle), new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialogInterface, int i) {
                        dialogInterface.dismiss();
                    }
                })
                .setCancelable(false)
                .show();
    }


    /**
     * 再次申请权限时，是否需要声明
     *
     * @param permissions
     * @return
     */
    private boolean shouldShowRequestPermissionRationale(String[] permissions) {
        for (String permission : permissions) {
            if (ActivityCompat.shouldShowRequestPermissionRationale(this, permission)) {
                return true;
            }
        }
        return false;
    }


    /**
     * 检察每个权限是否申请
     *
     * @param permissions
     * @return true 需要申请权限,false 已申请权限
     */
    private boolean checkEachSelfPermission(String[] permissions) {
        for (String permission : permissions) {
            if (ContextCompat.checkSelfPermission(this, permission) != PackageManager.PERMISSION_GRANTED) {
                return true;
            }
        }
        return false;
    }

    /**
     * 申请权限结果的回调
     *
     * @param requestCode
     * @param permissions
     * @param grantResults
     */
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == mRequestCode) {
            if (checkEachPermissionsGranted(grantResults)) {
                if (mListener != null) {
                    mListener.onPermissionGranted();
                }
            } else {// 用户拒绝申请权限
                if (mListener != null) {
                    mListener.onPermissionDenied();
                }
            }
        }
    }

    /**
     * 检查回调结果
     *
     * @param grantResults
     * @return
     */
    private boolean checkEachPermissionsGranted(int[] grantResults) {
        for (int result : grantResults) {
            if (result != PackageManager.PERMISSION_GRANTED) {
                return false;
            }
        }
        return true;
    }
}
```

每一个方法都有注释，逻辑也相对简单，就是我会遍历每个数组中申请的权限，如果需要申请就去申请，然后再处理一下回调的结果，其中还有对用户拒绝，然后再次申请弹出 Dialog 的处理。


如果在 Fragment 中使用，只是改变了一下参数封到 BaseFragment 中，这里就不贴代码了，都已经上传 **github**

BaseActivity：

[BuzzerBeater BaseActivity](https://github.com/itsMelo/BuzzerBeater/blob/master/app/src/main/java/com/blog/melo/buzzerbeater/activity/BaseActivity.java)

BaseFragment：

[BuzzerBeater BaseFragment](https://github.com/itsMelo/BuzzerBeater/blob/master/app/src/main/java/com/blog/melo/buzzerbeater/fragment/BaseFragment.java)

如有问题，继续交流~