---
layout:     post
title:      Android PMS RuntimePermission
subtitle:   App Permissions
date:       2019-07-19
author:     LXG
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - android
    - pms
    - awlog
---

### 参考网址

[运行时权限-AOSP](https://source.android.com/devices/tech/config/runtime_perms)

[APP如何请求运行时权限-Developer](https://developer.android.com/training/permissions/requesting)

[Android O特许权限白名单](https://source.android.com/devices/tech/config/perms-whitelist)

[Android Permissions-简书](https://www.jianshu.com/p/ffd583f720f4)

### 运行时权限和gids

**GIDS**

> gids是由框架在Application安装过程中生成，与Application申请的具体权限相关。如果Application申请的相应的permission被granted，而且有对应的gids，那么这个Application的gids中将包含这个gids


[platform.xml](http://androidxref.com/7.1.2_r36/xref/frameworks/base/data/etc/platform.xml)

```xml

<!-- This file is used to define the mappings between lower-level system
     user and group IDs and the higher-level permission names managed
     by the platform.

     Be VERY careful when editing this file!  Mistakes made here can open
     big security holes.
-->
<permissions>

    <permission name="android.permission.WRITE_MEDIA_STORAGE" >
        <group gid="media_rw" />
        <group gid="sdcard_rw" />
    </permission>

</permissions>

```

### data/media目录访问权限问题

```txt
BE:/ # ls -al data/media/
drwxrwx---  4 media_rw media_rw 3452 1970-01-01 08:00 awlog

```

**修改方案**

```diff

diff --git a/android/frameworks/base/data/etc/platform.xml b/android/frameworks/base/data/etc/platform.xml
index 1c65664893..0bc6349bfe 100644
--- a/android/frameworks/base/data/etc/platform.xml
+++ b/android/frameworks/base/data/etc/platform.xml
@@ -64,6 +64,11 @@
         <group gid="log" />
     </permission>
 
+    <!--lixiaogang add -->
+    <permission name="android.permission.WRITE_MEDIA_STORAGE" >
+        <group gid="media_rw" />
+    </permission>
+
     <permission name="android.permission.ACCESS_MTP" >
         <group gid="mtp" />
     </permission>

```

如此修改后App申请android.permission.WRITE_MEDIA_STORAGE权限后即可访问data/media/awlog目录

### 调试命令

**查看设备支持的运行时权限列表**

adb shell pm list permissions -g -d

**查看进程gids**

adb shell dumpsys activity p com.sunmi.superpermissiontest

**查看应用已经授予的动态权限**

adb shell dumpsys package com.sunmi.superpermissiontest

**权限授予和收回**

pm grant [--user USER_ID] PACKAGE PERMISSION<br>
pm revoke [--user USER_ID] PACKAGE PERMISSION<br>
pm reset-permissions<br>
pm set-permission-enforced PERMISSION [true|false]

### 系统预置应用授权

[DefaultPermissionGrantPolicy.java](http://androidxref.com/7.1.2_r36/xref/frameworks/base/services/core/java/com/android/server/pm/DefaultPermissionGrantPolicy.java)

**PackageManagerService**

```java

    final DefaultPermissionGrantPolicy mDefaultPermissionPolicy;

    public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {

            mDefaultPermissionPolicy = new DefaultPermissionGrantPolicy(this);

    }

    @Override
    public void systemReady() {

        // If we upgraded grant all default permissions before kicking off.
        for (int userId : grantPermissionsUserIds) {
            mDefaultPermissionPolicy.grantDefaultPermissions(userId);
        }

    }

    @Override
    public void grantRuntimePermission(String packageName, String name, final int userId) {}

    @Override
    public void revokeRuntimePermission(String packageName, String name, int userId) {}

```

**DefaultPermissionGrantPolicy**

```java

    public void grantDefaultPermissions(int userId) {
        grantPermissionsToSysComponentsAndPrivApps(userId);
        grantDefaultSystemHandlerPermissions(userId);
        grantDefaultPermissionExceptions(userId);
    }

```

### APP如何请求动态权限

```java

// 请求权限

// Here, thisActivity is the current activity
if (ContextCompat.checkSelfPermission(thisActivity,
        Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

    // Permission is not granted
    // Should we show an explanation?
    if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
            Manifest.permission.READ_CONTACTS)) {
        // Show an explanation to the user *asynchronously* -- don't block
        // this thread waiting for the user's response! After the user
        // sees the explanation, try again to request the permission.
    } else {
        // No explanation needed; request the permission
        ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.READ_CONTACTS},
                MY_PERMISSIONS_REQUEST_READ_CONTACTS);

        // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
        // app-defined int constant. The callback method gets the
        // result of the request.
    }
} else {
    // Permission has already been granted
}

// 请求结果

@Override
public void onRequestPermissionsResult(int requestCode,
        String[] permissions, int[] grantResults) {
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
        // permissions this app might request.
    }
}

```



