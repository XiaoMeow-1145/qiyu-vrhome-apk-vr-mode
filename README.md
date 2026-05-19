# 奇遇VR Home APK VR 模式

## 规则

生成一个修补后的APK文件。除非用户明确要求设备测试，否则不要求adb、连接的头显、`adb install`、`am start`、logcat、截图或设备验证。

主要清单信号是应用级元数据：

```xml
<application ...>
    <meta-data android:name="qiyu-sdk-flag" android:value="true" />
</application>
```

在QIYU Mix / VR700风格的固件上，奇遇Home会调用`PMExt.isVRApplication(packageName)`。PackageManager实现会在以下任一情况下返回true：

- `ApplicationInfo.metaData.getBoolean("qiyu-sdk-flag")`为true。
- `ApplicationInfo.vrType == 2`。

因此，离线APK补丁应该添加`qiyu-sdk-flag=true`。不要将`com.qti.intent.category.SNAPDRAGON_VR`视为决定性的奇遇Home VR应用信号。只有在用户要求时才将其作为兼容性提示添加。

不要将奇遇Home自己的启动器操作`iqiyi.3d.vr.launcher`添加到第三方应用程序中。该操作属于`com.qiyu.vraio.home/.QiyuHomeActivity`，而不是修补的应用程序清单。

## 可选的旧版类别

为了与高通/XR风格的启动模式兼容，脚本还可以将此类别添加到应用程序的`MAIN + LAUNCHER`活动：

```xml
<category android:name="com.qti.intent.category.SNAPDRAGON_VR" />
```

仅在用户也要求旧版类别时使用：

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --add-qti-category \
  --output /path/to/input-qiyu-vrmode.apk
```

如果APK有多个启动器活动且必须添加类别，请指定`--activity`。

## 快速开始

首先使用捆绑的脚本：

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --output /path/to/input-qiyu-vrmode.apk
```

此默认路径仅添加`qiyu-sdk-flag=true`，重建，在可能时对齐，并对输出APK进行签名。

如果还需要可选的QTI类别且APK有多个启动器活动，请明确指定目标：

```bash
python3 ~/.codex/skills/qiyu-vrhome-apk-vr-mode/scripts/patch_qiyu_vrmode_apk.py \
  /path/to/input.apk \
  --add-qti-category \
  --activity com.example.game/.MainActivity \
  --output /path/to/input-qiyu-vrmode.apk
```

如果用户只需要生成修补后的文件，请在脚本写入输出APK后停止。不要要求设备。

## 所需工具

脚本需要：

- `apktool`
- 一个签名工具：来自Android SDK构建工具的`apksigner`，或`jarsigner`
- 如果脚本必须创建调试签名密钥，则需要`keytool`
- `zipalign`是首选，但脚本可以在没有它的情况下继续

如果缺少所需工具，请报告确切的缺失工具并停止。不要通过直接编辑zip字节来发明二进制APK补丁。

## 脚本行为

脚本执行这些确定性步骤：

1. 使用`apktool`解码APK。
2. 解析解码的`AndroidManifest.xml`。
3. 定位清单中的`<application>`。
4. 添加或更新`<meta-data android:name="qiyu-sdk-flag" android:value="true" />`。
5. 如果使用了`--add-qti-category`或`--category`，请定位目标`activity`或`activity-alias`并修补其`MAIN + LAUNCHER` `intent-filter`。
6. 使用`apktool`重新构建APK。
7. 在可用时使用`zipalign`对齐。
8. 对生成的APK进行签名。
9. 打印包名、元数据结果、可选活动/类别和输出路径。

默认签名密钥是在工作目录下生成的调试密钥。这足以生成可安装的APK，但它不会更新已使用不同证书签名的已安装应用。如果用户需要就地更新，请要求原始签名密钥库或告诉他们必须先卸载现有包，然后再安装调试签名的修补APK。

## 手动回退

仅在无法使用脚本但可用外部工具时使用此回退。

解码：

```bash
apktool d -f /path/to/input.apk -o /tmp/qiyu-vrmode-decoded
```

修补`/tmp/qiyu-vrmode-decoded/AndroidManifest.xml`，使`<application>`元素具有：

```xml
<meta-data android:name="qiyu-sdk-flag" android:value="true" />
```

如果还要添加可选的旧版类别，请修补主启动活动，使其具有：

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
    <category android:name="com.qti.intent.category.SNAPDRAGON_VR" />
</intent-filter>
```

重建：

```bash
apktool b /tmp/qiyu-vrmode-decoded -o /tmp/qiyu-vrmode-unsigned.apk
```

如果存在`zipalign`则对齐：

```bash
zipalign -p -f 4 /tmp/qiyu-vrmode-unsigned.apk /tmp/qiyu-vrmode-aligned.apk
```

使用`apksigner`签名：

```bash
apksigner sign \
  --ks /path/to/debug.keystore \
  --ks-key-alias qiyuvrmode \
  --ks-pass pass:android \
  --key-pass pass:android \
  --out /path/to/output-qiyu-vrmode.apk \
  /tmp/qiyu-vrmode-aligned.apk
```

如果使用`jarsigner`，请将对齐的APK签名到输出路径：

```bash
jarsigner \
  -keystore /path/to/debug.keystore \
  -storepass android \
  -keypass android \
  -signedjar /path/to/output-qiyu-vrmode.apk \
  /tmp/qiyu-vrmode-aligned.apk \
  qiyuvrmode
```

## 静态检查

允许静态检查且不需要adb：

```bash
apktool d -f /path/to/output-qiyu-vrmode.apk -o /tmp/qiyu-vrmode-check
rg -n "qiyu-sdk-flag|com.qti.intent.category.SNAPDRAGON_VR|android.intent.action.MAIN|android.intent.category.LAUNCHER" \
  /tmp/qiyu-vrmode-check/AndroidManifest.xml
```

如果有`aapt`或`aapt2`，清单转储也是可以接受的。不要因为不存在adb设备而使任务失败。

可选的设备端验证，仅在用户明确要求时：

```bash
adb shell service call package 211 s16 <package.name>
```

结果中的`00000001`表示固件端的`PMExt.isVRApplication(packageName)`检查返回true。不要默认运行此命令。

## 参考

当您需要历史理由、奇遇Home窗口化启动证据、已知类别模式或不应添加的内容时，请阅读`references/qiyu-vrhome-vr-signals.md`。
