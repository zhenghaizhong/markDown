# Android N Intent传递(Uri)
* ## 序言
   Android N不再支持通过Intent传递'file://' scheme,官方并推荐使用[FileProvider]('https://developer.android.com/reference/android/support/v4/content/FileProvider.html)的方式去解决.但本人使用过程中遇到了不少问题，借此总结一下，下面的内容是建立在已经了解如何使用FileProvider的基础上的。

* ## 分享不崩溃但不成功
  使用了FileProvider后，应用进行分享操作虽然不崩溃但还是失败，会有如下的日志：'System.err: java.lang.SecurityException: Permission Denial: opening provider android.support.v4.content.FileProvider from ProcessRecord'(使用FileProvider时，已在AndroidManifest赋予了权限：android:grantUriPermissions="true")      ,        [解决办法](http://stackoverflow.com/questions/18249007/how-to-use-support-fileprovider-for-sharing-content-to-other-apps):赋予权限给所有符合条件应用。猜测原因：应用分享过程中，经过了分享应用后才达到真正分享到的应用，Manifest赋予分享这个应用权限，而真正分享到的应用就没权限了。
*个人猜测：Manifest的android:grantUriPermissions="true" 是否理解为Context.grantUriPermission(package, Uri, mode_flags)的自动识别package版*


***
* ## 崩溃源码
应用之所以崩溃查看源码可得：
*`  startActivity(Intent)` 中：*
```java
Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
```
*` Instrumentation的execStartActivity(Context、IBinder、IBinder、String、Intent、int、Bundle)` 中：*
```java
try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
```
*` intent.prepareToLeaveProcess() `中：*
```java
public void prepareToLeaveProcess(boolean leavingPackage) {
        setAllowFds(false);

        if (mSelector != null) {
            mSelector.prepareToLeaveProcess(leavingPackage);
        }
        if (mClipData != null) {
            mClipData.prepareToLeaveProcess(leavingPackage);
        }

        if (mAction != null && mData != null && StrictMode.vmFileUriExposureEnabled()
                && leavingPackage) {
            switch (mAction) {
                case ACTION_MEDIA_REMOVED:
                case ACTION_MEDIA_UNMOUNTED:
                case ACTION_MEDIA_CHECKING:
                case ACTION_MEDIA_NOFS:
                case ACTION_MEDIA_MOUNTED:
                case ACTION_MEDIA_SHARED:
                case ACTION_MEDIA_UNSHARED:
                case ACTION_MEDIA_BAD_REMOVAL:
                case ACTION_MEDIA_UNMOUNTABLE:
                case ACTION_MEDIA_EJECT:
                case ACTION_MEDIA_SCANNER_STARTED:
                case ACTION_MEDIA_SCANNER_FINISHED:
                case ACTION_MEDIA_SCANNER_SCAN_FILE:
                case ACTION_PACKAGE_NEEDS_VERIFICATION:
                case ACTION_PACKAGE_VERIFIED:
                    // Ignore legacy actions
                    break;
                default:
                    mData.checkFileUriExposed("Intent.getData()");
            }
        }
    }
```
```java
    public void checkFileUriExposed(String location) {
        if ("file".equals(getScheme()) && !getPath().startsWith("/system/")) {
            StrictMode.onFileUriExposed(this, location);
        }
    }
```
* ## 小结
* 1.setResult然后finish这种Intent传递仍可使用以前的Uri（file：//）。
* 2 .ClipData的情况网上的FileProvider教程举例有的是系统拍照（action：MediaStore.ACTION_IMAGE_CAPTURE）或者系统裁剪(action:com.android.camera.action.CROP)，查源码得出MediaStore.ACTION_IMAGE_CAPTURE的action注释中有写到会使用setClipData，上述的`mClipData.prepareToLeaveProcess() `中：
```java
    public void prepareToLeaveProcess() {
        final int size = mItems.size();
        for (int i = 0; i < size; i++) {
            final Item item = mItems.get(i);
            if (item.mIntent != null) {
                item.mIntent.prepareToLeaveProcess();
            }
            if (item.mUri != null && StrictMode.vmFileUriExposureEnabled()) {
                item.mUri.checkFileUriExposed("ClipData.Item.getUri()");
            }
        }
    }
```
 
* ## FileProvider的使用理解
源码摘自android.support.v4.content.FileProvider
先解析一下使用FileProvider中，必须的xml中的含义：
```java
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path path="Android" name="files" />
</paths>
```
` external-path`下面称为节点，文章开头的官方链接有其详细的介绍.还有其他的节点：files-path、cache-path 等

`path `字段配合节点主要的功能是作传递前的过滤（getUriForFile源码中）：
```java
 // Find the most-specific root path
            Map.Entry<String, File> mostSpecific = null;
            for (Map.Entry<String, File> root : mRoots.entrySet()) {
                final String rootPath = root.getValue().getPath();
                if (path.startsWith(rootPath) && (mostSpecific == null
                        || rootPath.length() > mostSpecific.getValue().getPath().length())) {
                    mostSpecific = root;
                }
            }

            if (mostSpecific == null) {
                throw new IllegalArgumentException(
                        "Failed to find configured root that contains " + path);
            }
```
就上述的xml，rootPath=Environment.getExternalStorageDirectory()+“/Android”.大概就是你要传递的uri的path路径如果不符合此rootpath下的，将不会传递成功，算是一种安全性的加强吧。

`name `字段，作用与键值中的键的含义一样，一个xml中可能存在多个节点，以此来标识，而它恰好是作为一个键来使用，mRoot的键

`mRoot `是一个HashMap，它的键是xml中的name字段，值就是路径是节点+path字段生成的file，理解为你要传递的file的父目录.mRoot生成过程：
```java
private static PathStrategy parsePathStrategy(Context context, String authority)
            throws IOException, XmlPullParserException {
        final SimplePathStrategy strat = new SimplePathStrategy(authority);

        final ProviderInfo info = context.getPackageManager()
                .resolveContentProvider(authority, PackageManager.GET_META_DATA);
        final XmlResourceParser in = info.loadXmlMetaData(
                context.getPackageManager(), META_DATA_FILE_PROVIDER_PATHS);
        if (in == null) {
            throw new IllegalArgumentException(
                    "Missing " + META_DATA_FILE_PROVIDER_PATHS + " meta-data");
        }

        int type;
        while ((type = in.next()) != END_DOCUMENT) {
            if (type == START_TAG) {
                final String tag = in.getName();

                final String name = in.getAttributeValue(null, ATTR_NAME);
                String path = in.getAttributeValue(null, ATTR_PATH);

                File target = null;
                if (TAG_ROOT_PATH.equals(tag)) {
                    target = DEVICE_ROOT;
                } else if (TAG_FILES_PATH.equals(tag)) {
                    target = context.getFilesDir();
                } else if (TAG_CACHE_PATH.equals(tag)) {
                    target = context.getCacheDir();
                } else if (TAG_EXTERNAL.equals(tag)) {
                    target = Environment.getExternalStorageDirectory();
                } else if (TAG_EXTERNAL_FILES.equals(tag)) {
                    File[] externalFilesDirs = ContextCompat.getExternalFilesDirs(context, null);
                    if (externalFilesDirs.length > 0) {
                        target = externalFilesDirs[0];
                    }
                } else if (TAG_EXTERNAL_CACHE.equals(tag)) {
                    File[] externalCacheDirs = ContextCompat.getExternalCacheDirs(context);
                    if (externalCacheDirs.length > 0) {
                        target = externalCacheDirs[0];
                    }
                }

                if (target != null) {
                    strat.addRoot(name, buildPath(target, path));
                }
            }
        }

        return strat;
    }
```
备注：` TAG_ROOT_PATH = "root-path"`，官方文档未提到到节点，如果你传递的是SD卡或别的分区的文件，就得使用这个节点了，对应的是文件系统的根目录，也就是“/"。

* ##解析Uri中的Path

    由FileProvider.getUriForFile(Context context, String authority, File file) 生成的Uri格式是：
    ` Content:// authority/xml中name字段/path`
    
    
之所以要解析是因为在工作中遇到：
* 1.接收方是通过ContentResolver.query或者ContentResolver.openInputStream的方式处理Uri.可能造成解析失败(FileNotFoundException)或者应用崩溃。
* 2.生成Uri需要File，生成File需要Path，但Path有可能是相对路径可能是绝对路径，加上万能节点root-path，情况如下：
      * `相对路径+非root-path` 自身崩溃，因为传递前的过滤机制过不了
      * `相对路径+root-path` 成功
      * `绝对路径+root-path` 成功 ，uri的字段不会被截取
      * `绝对路径+对应的节点` 成功 ，uri的字段会被截取
        在getUriForFile中：
```java
// Start at first char of path under root
            final String rootPath = mostSpecific.getValue().getPath();
            if (rootPath.endsWith("/")) {
                path = path.substring(rootPath.length());
            } else {
                path = path.substring(rootPath.length() + 1);
            }
```
也就是节点是root-path的话，路径是什么，uri中的路径就是什么。
对于最后一种情况`绝对路径+对应的节点` 
举个例子：
File的path = Environment.getExternalStorageDirectory()+“/abc.txt”.

节点 = "external-path"

 接收方的Uri=    ` Content:// authority/xml中name字段/abc.txt`
 ***
 其实FileProvider中有一个很好的方法**getFileForUri**去解析Uri，但google了一下这方法并未开放。
    **方法一：通过反射调用此方法获取file，就能获取到path**.
    _备注_:此方法未实验过，理论上可行，是和一位同事交流后他写的:)
```java
public static File getFileForUri(Context context, String authority, Uri uri) {
        try {
            final Method getPathStrategyMethod = ReflectHelper.getDeclaredMethod(FileProvider.class, "getPathStrategy", Context.class, String.class);
            getPathStrategyMethod.setAccessible(true);
            final Object strategy = getPathStrategyMethod.invoke(null, context, authority);
            final Method getFileForUriMethod = ReflectHelper.getDeclaredMethod(strategy.getClass(), "getFileForUri", Uri.class);
            getFileForUriMethod.setAccessible(true);
            return (File) getFileForUriMethod.invoke(strategy, uri);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;
    }
```
  
**    方法二：修改getFileForUri**
```java
public static String getAbsolutePath(String path, String authority) {
        Context context = FileManagerApplication.getContext();
        String target = null;
        final ProviderInfo info = context.getPackageManager()
                .resolveContentProvider(authority, PackageManager.GET_META_DATA);
        final XmlResourceParser in = info.loadXmlMetaData(
                context.getPackageManager(), META_DATA_FILE_PROVIDER_PATHS);
        if (in == null) {
            throw new IllegalArgumentException(
                    "Missing " + META_DATA_FILE_PROVIDER_PATHS + " meta-data");
        }
        int type;
        try {
            while ((type = in.next()) != END_DOCUMENT) {
                if (type == START_TAG) {
                    final String tag = in.getName();
                    if (TAG_ROOT_PATH.equals(tag)) {
                        target = "/" + path;
                    } else if (TAG_FILES_PATH.equals(tag)) {
                        target = context.getFilesDir() + "/" + path;
                    } else if (TAG_CACHE_PATH.equals(tag)) {
                        target = context.getCacheDir() + "/" + path;
                    } else if (TAG_EXTERNAL.equals(tag)) {
                        target = Environment.getExternalStorageDirectory() + "/" + path;
                    } else if (TAG_EXTERNAL_FILES.equals(tag)) {
                        File[] externalFilesDirs = ContextCompat.getExternalFilesDirs(context, null);
                        if (externalFilesDirs.length > 0) {
                            target = externalFilesDirs[0].getPath() + "/" + path;
                        }
                    } else if (TAG_EXTERNAL_CACHE.equals(tag)) {
                        File[] externalCacheDirs = ContextCompat.getExternalCacheDirs(context);
                        if (externalCacheDirs.length > 0) {
                            target = externalCacheDirs[0].getPath() + "/" + path;
                        }
                    }

                }
            }
        } catch (XmlPullParserException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return target;
    }
```
 _备注_：可能官方有解析的代码，但暂时没找到。
 
* ## VmPolicy方式规避
其实这都是Andorid N执行了StrictMode API，将如下方法写到Application中就可以规避，成功传递出file://，亲测成功:
```java
        StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
        StrictMode.setVmPolicy(builder.build());
```
* ## 最后
    如果文中有分析错误或者啥的，欢迎指出.
