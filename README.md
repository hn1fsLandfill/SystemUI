## SystemUI from android-11.0.0_r10  
### Compiling SystemUI in Android Studio Without Source Tree Dependency  

### Support Description  
* Does **not** attempt to change the original project directory structure  
* Adds additional configuration and dependencies to build a Gradle environment  
* Uses scripts to remove properties and fields not supported by Android Studio (AS), and ignores them using Git locally  
* The runtime behavior may slightly differ from the native version — this is due to style differences caused by the failure to reference `private` attributes after detaching from the source tree  

### Pixel 2 Runtime Comparison: Gradle Build vs Android.bp Build  
---  
<img src="images/pixel2_systemui_gradle.jpg" width="225" height="400"/> <img src="images/pixel2_systemui_original.jpg" width="225" height="400"/>  
---

## Execution Steps  
#### Step 1: Run the main function in the `Filter` to execute filtering tasks  
<img src="images/filter_main.png" width="600" height="480"/>

### Step 2: In Android Studio, perform **Build APK**, then push the APK to the SystemUI directory on the device:

```bash
adb push SystemUI.apk /system/system_ext/priv-app/SystemUI/

adb shell killall com.android.systemui
```

###### If SystemUI fails to start, a device reboot is required:

```bash
adb reboot
```

---

## Build Steps  

### Step 1: Add Static Dependencies  

##### @framework.jar:  
```gradle
// AOSP/android-11/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes-header.jar
compileOnly files('libs/framework.jar')
```
![framework](images/framework.png)

##### @core-all.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/libcore/core-all/android_common/javac/core-all.jar
compileOnly files('libs/core-all.jar')
```
![core-all](images/core-all.png)

##### @preference-1.2.0-alpha01.aar:  
```gradle
// AOSP/android-11/prebuilts/sdk/current/androidx/m2repository/androidx/preference/preference/1.2.0-alpha01/preference-1.2.0-alpha01.aar
implementation(name: 'preference-1.2.0-alpha01', ext: 'aar')
```
![preference](images/preference-1.2.0-alpha01.png)

> ps: `androidx.preference` is difficult to reference using the following method, so we use a static import instead:  
```gradle
## implementation 'androidx.preference:preference:1.2.0-alpha01'
```

##### @iconloader_base.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/libs/systemui/iconloaderlib/iconloader_base/android_common/javac/iconloader_base.jar
implementation files('libs/iconloader_base.jar')
```
![iconloader_base](images/iconloader_base.png)

##### @libprotobuf-java-nano.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/external/protobuf/libprotobuf-java-nano/android_common/javac/libprotobuf-java-nano.jar
implementation files('libs/libprotobuf-java-nano.jar')
```
![protobuf](images/libprotobuf-java-nano.png)

##### @SystemUIPluginLib.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/packages/SystemUI/plugin/SystemUIPluginLib/android_common/javac/SystemUIPluginLib.jar
implementation files('libs/SystemUIPluginLib.jar')
```
![pluginlib](images/SystemUIPluginLib.png)

##### @SystemUISharedLib.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/packages/SystemUI/shared/SystemUISharedLib/android_common/javac/SystemUISharedLib.jar
implementation files('libs/SystemUISharedLib.jar')
```
![sharedlib](images/SystemUISharedLib.png)

##### @WindowManager-Shell.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/libs/WindowManager/Shell/WindowManager-Shell/android_common/javac/WindowManager-Shell.jar
implementation files('libs/WindowManager-Shell.jar')
```
![wm-shell](images/WindowManager-Shell.png)

##### @SystemUI-tags.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/packages/SystemUI/SystemUI-tags/android_common/javac/SystemUI-tags.jar
implementation files('libs/SystemUI-tags.jar')
```
![tags](images/SystemUI-tags.png)

##### @SystemUI-proto.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/packages/SystemUI/SystemUI-proto/android_common/javac/SystemUI-proto.jar
implementation files('libs/SystemUI-proto.jar')
```
![proto](images/SystemUI-proto.png)

##### @SystemUI-statsd.jar:  
```gradle
// AOSP/android-11/out/soong/.intermediates/frameworks/base/packages/SystemUI/shared/SystemUI-statsd/android_common/javac/SystemUI-statsd.jar
implementation files('libs/SystemUI-statsd.jar')
```
![statsd](images/SystemUI-statsd.png)

---

### Step 2: Import Modules  
Import the specific source code directories as Gradle modules. You can reference them using `implementation project(...)`, or alternatively build them into AARs and include them in the `libs` folder.

##### @iconloaderlib:  
```gradle
// AOSP/android-11/frameworks/libs/systemui/iconloaderlib
implementation project(':iconloaderlib')
```
![iconloaderlib](images/iconloaderlib.png)

##### @WifiTrackerLib:  
```gradle
// AOSP/android-11/frameworks/opt/net/wifi/libs/WifiTrackerLib
implementation project(':WifiTrackerLib')
```
![wifitracker](images/WifiTrackerLib.png)

##### @SettingsLib:  
```gradle
// AOSP/android-11/frameworks/base/packages/SettingsLib
implementation project(':SettingsLib:RestrictedLockUtils')
implementation project(':SettingsLib:AdaptiveIcon')
implementation project(':SettingsLib:HelpUtils')
implementation project(':SettingsLib:ActionBarShadow')
implementation project(':SettingsLib:AppPreference')
implementation project(':SettingsLib:SearchWidget')
implementation project(':SettingsLib:SettingsSpinner')
implementation project(':SettingsLib:LayoutPreference')
implementation project(':SettingsLib:ActionButtonsPreference')
implementation project(':SettingsLib:EntityHeaderWidgets')
implementation project(':SettingsLib:BarChartPreference')
implementation project(':SettingsLib:ProgressBar')
implementation project(':SettingsLib:RadioButtonPreference')
implementation project(':SettingsLib:DisplayDensityUtils')
implementation project(':SettingsLib:Utils')
```
![settingslib](images/SettingsLib.png)

---

## Generate Default Signature with platform.keystore  

Navigate to `AOSP/android-11/build/target/product/security` to find the signing certificate, then use [`keytool-importkeypair`](https://github.com/getfatday/keytool-importkeypair) to generate the keystore.

Execute the following command:  
```bash
./keytool-importkeypair -k platform.keystore -p 123456 -pk8 platform.pk8 -cert platform.x509.pem -alias platform
```

Then add the following block to your Gradle configuration:

```gradle
signingConfigs {
    platform {
        storeFile file("platform.keystore")
        storePassword '123456'
        keyAlias 'platform'
        keyPassword '123456'
    }
}

buildTypes {
    release {
        debuggable false
        minifyEnabled false
        signingConfig signingConfigs.platform
    }

    debug {
        debuggable true
        minifyEnabled false
        signingConfig signingConfigs.platform
    }
}
```

---

### Notes  

#### View Ignored Files in Git  
```bash
git ls-files -v | grep '^h '
```

#### Ignore or Restore a Specific File  
```bash
git update-index --assume-unchanged $path
git update-index --no-assume-unchanged $path
```

#### Restore All Ignored Files  
```bash
git ls-files -v | grep '^h' | awk '{print $2}' | xargs git update-index --no-assume-unchanged
```

---

### Related Projects  
* [Settings](https://github.com/siren-ocean/Settings)  
* [Launcher3](https://github.com/siren-ocean/Launcher3)  
* [DocumentsUI](https://github.com/siren-ocean/DocumentsUI)  
* [Camera2](https://github.com/siren-ocean/Camera2)  
* [PermissionController](https://github.com/siren-ocean/PermissionController)
