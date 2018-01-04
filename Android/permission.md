permission

自定义权限如下

```xml
<permission
    android:name="test"
    android:permissionGroup="xx"
    android:protectionLevel="signature"/>
```

系统的例子

~~~xml
<!-- Used for permissions that can be used to make the user spend money  
     without their direct involvement.  For example, this is the group  
     for permissions that allow you to directly place phone calls,  
     directly send SMS messages, etc. -->  
<permission-group android:name="android.permission-group.COST_MONEY"  
    android:label="@string/permgrouplab_costMoney"  
    android:description="@string/permgroupdesc_costMoney" />  
<!-- Allows an application to send SMS messages. -->  
<permission android:name="android.permission.SEND_SMS"  
    android:permissionGroup="android.permission-group.COST_MONEY"  
    android:protectionLevel="dangerous"  
    android:label="@string/permlab_sendSms"  
    android:description="@string/permdesc_sendSms" />  
<!-- Allows an application to initiate a phone call without going through  
     the Dialer user interface for the user to confirm the call  
     being placed. -->  
<permission android:name="android.permission.CALL_PHONE"  
    android:permissionGroup="android.permission-group.COST_MONEY"  
    android:protectionLevel="dangerous"  
    android:label="@string/permlab_callPhone"  
    android:description="@string/permdesc_callPhone" />  
~~~

`protectionLevel` 属性是必要属性，用于指示系统如何向用户告知需要权限的应用，或者谁可以拥有该权限，具体如链接的文档中所述。

[`android:permissionGroup`](https://developer.android.com/guide/topics/manifest/permission-group-element.html?hl=zh-cn) 属性是可选属性，只是用于帮助系统向用户显示权限。大多数情况下，您要将此设为标准系统组（列在 `android.Manifest.permission_group` 中），但您也可以自己定义一个组。建议使用现有的组，因为这样可简化向用户显示的权限 UI。



- 如果应用请求其清单中列出的危险权限，而应用目前在权限组中没有任何权限，则系统会向用户显示一个对话框，描述应用要访问的权限组。对话框不描述该组内的具体权限。例如，如果应用请求 `READ_CONTACTS` 权限，系统对话框只说明该应用需要访问设备的联系信息。如果用户批准，系统将向应用授予其请求的权限。
- 如果应用请求其清单中列出的危险权限，而应用在同一权限组中已有另一项危险权限，则系统会立即授予该权限，而无需与用户进行任何交互。例如，如果某应用已经请求并且被授予了 `READ_CONTACTS` 权限，然后它又请求 `WRITE_CONTACTS`，系统将立即授予该权限。

