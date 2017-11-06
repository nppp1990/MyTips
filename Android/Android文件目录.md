Android文件目录

### 问题

之前碰到线上有人投诉离线缓存没用，但是本地测试没有问题。发现OPPO手机能复现：用oppo的杀进程从recent里杀掉app，然后离线缓存就失效了，查看设置里的存储信息，发现下图的缓存数据为0

### 问题原因

okhttp配置的缓存目录如下

```java
builder.cache(new Cache(new File(context.getCacheDir(), "responses"), CONST_10 * CONST_1024 * CONST_1024));
```

```
/**
 * Returns the absolute path to the application specific cache directory on
 * the filesystem. These files will be ones that get deleted first when the
 * device runs low on storage. There is no guarantee when these files will
 * be deleted.
 *
 * @return The path of the directory holding application cache files.
 * @see #openFileOutput
 * @see #getFileStreamPath
 * @see #getDir
 * @see #getExternalCacheDir
 */
public abstract File getCacheDir();
```

getCacheDir是应用的缓存目录，这个目录下的文件会在手机内存不够时被删掉，所以会有可能缓存失效

### 解决办法

getCacheDir改成getFilesDir

### 各种目录

把能获取的几个目录都打印出来如下

```java
Ln.e("----getFilesDir==" + getFilesDir());
Ln.e("----getCacheDir==" + getCacheDir());
Ln.e("----Environment.getExternalStorageDirectory()==" + Environment.getExternalStorageDirectory());
Ln.e("----getExternalFilesDir==" + getExternalFilesDir(null));
Ln.e("----getExternalCacheDir==" + getExternalCacheDir());
Ln.e("----getObbDir==" + getObbDir());
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    Ln.e("----getCodeCacheDir==" + getCodeCacheDir());
}
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    Ln.e("----getDataDir==" + getDataDir());
}
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    Ln.e("----getNoBackupFilesDir==" + getNoBackupFilesDir());
}
```

结果如下：

```java
----getFilesDir==/data/user/0/package/files
----getCacheDir==/data/user/0/package/cache
----Environment.getExternalStorageDirectory()==/storage/emulated/0
----getExternalFilesDir==/storage/emulated/0/Android/data/package/files
----getExternalCacheDir==/storage/emulated/0/Android/data/package/cache
----getObbDir==/storage/emulated/0/Android/obb/package
----getCodeCacheDir==/data/user/0/package/code_cache
----getDataDir==/data/user/0/package
----getNoBackupFilesDir==/data/user/0/package/no_backup
```





