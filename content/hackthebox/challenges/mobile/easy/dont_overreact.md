---
title: dont_overreact
description: Writeup for the Don't Overreact mobile challenge on HackTheBox
created: 2025-10-11
tags: hackthebox, mobile, ctf, practice
draft: false
---

## Introduction

---

**Challenge Summary**: Don’t Overreact is a very easy Mobile challenge on HackTheBox. A quick code review reveals it’s a React Native Android application, which shifts our focus to the `index.android.bundle` file that contains the JavaScript. If you’re a meticulous code-reviewer, you can solve this entirely via static analysis without loading the APK on a device. That said, installing the APK can give useful hints about what to look for in the JavaScript.

**Challenge Description**: Some web developers wrote this fancy new app! It's really cool, isn't it?

**Challenge Difficulty**: Very Easy

**Challenge Category**: Mobile

## Enumeration

---

We begin like most Mobile challenges: decompiling the APK with jadx.

```bash
$ jadx ./app-release.apk
INFO  - loading ...
INFO  - processing ...
INFO  - done
```

Next, review AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    android:compileSdkVersion="29"
    android:compileSdkVersionCodename="10"
    package="com.awesomeproject"
    platformBuildVersionCode="29"
    platformBuildVersionName="10">
    <uses-sdk
        android:minSdkVersion="21"
        android:targetSdkVersion="29"/>
    <uses-permission android:name="android.permission.INTERNET"/>
    <application
        android:theme="@style/AppTheme"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:name="com.awesomeproject.MainApplication"
        android:allowBackup="false"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:appComponentFactory="androidx.core.app.CoreComponentFactory">
        <activity
            android:label="@string/app_name"
            android:name="com.awesomeproject.MainActivity"
            android:launchMode="singleTask"
            android:configChanges="screenSize|uiMode|orientation|keyboardHidden|keyboard"
            android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

The package is `com.awesomeproject` with a single Activity: `com.awesomeproject.MainActivity`. It requests only `android.permission.INTERNET`, suggesting some network usage. Nothing else stands out. Let’s pivot into the app code.

## Reverse-Engineering

---

Open `com/awesomeproject/MainApplication.java` and review its contents:

```java
package com.awesomeproject;

import android.app.Application;
import android.content.Context;
import com.facebook.react.PackageList;
import com.facebook.react.ReactApplication;
import com.facebook.react.ReactInstanceManager;
import com.facebook.react.ReactNativeHost;
import com.facebook.react.ReactPackage;
import com.facebook.soloader.SoLoader;
import java.util.List;

/* loaded from: classes.dex */
public class MainApplication extends Application implements ReactApplication {
    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) { // from class: com.awesomeproject.MainApplication.1
        @Override // com.facebook.react.ReactNativeHost
        protected String getJSMainModuleName() {
            return "index";
        }

        @Override // com.facebook.react.ReactNativeHost
        public boolean getUseDeveloperSupport() {
            return false;
        }

        @Override // com.facebook.react.ReactNativeHost
        protected List<ReactPackage> getPackages() {
            return new PackageList(this).getPackages();
        }
    };

    private static void initializeFlipper(Context context, ReactInstanceManager reactInstanceManager) {
    }

    @Override // com.facebook.react.ReactApplication
    public ReactNativeHost getReactNativeHost() {
        return this.mReactNativeHost;
    }

    @Override // android.app.Application
    public void onCreate() {
        super.onCreate();
        SoLoader.init((Context) this, false);
        initializeFlipper(this, getReactNativeHost().getReactInstanceManager());
    }
}
```

At app startup, `onCreate()` initializes `SoLoader` and (optionally) `Flipper`. The imports and the ReactApplication interface strongly indicate this is a React Native app.

[React Native](https://reactnative.dev/) is a UI framework for building mobile apps for iOS and Android using JavaScript. That means most application logic will be in JavaScript, not Java/Kotlin. So our next step is to locate and analyze the JS bundle.

After a quick search, we find `app-release/resources/assets/index.android.bundle`. The [`index.android.bundle`](https://medium.com/@naandalist/what-is-index-android-bundle-in-react-native-and-why-you-shouldnt-commit-to-repository-be5774b26652) file contains the packed/minified React Native JavaScript. Dump it and run it through a beautifier like [beautifier.io](https://beautifier.io/) for readability.

Before diving into minified code (a task I wouldn’t wish on my worst enemy), install the APK on a device or emulator to see if the UI offers any hints.

---

<img src="/assets/hackthebox/challenges/mobile/easy/mobile_dont_overreact/launch.png" alt="launch" width="400" />

---

Nothing but the HackTheBox logo. Not much, but a small breadcrumb.

Now, search the beautified JS for obvious strings: “htb”, “hackthebox”, etc. Eventually, we hit the following snippet with a base64-encoded value:

```js
// ...

__d(
  function (g, r, i, a, m, e, d) {
    ;(Object.defineProperty(e, "__esModule", {
      value: !0,
    }),
      (e.myConfig = void 0))
    var t = {
      importantData: "baNaNa".toLowerCase(),
      apiUrl: "https://www.hackthebox.eu/",
      debug: "SFRCezIzbTQxbl9jNDFtXzRuZF9kMG43XzB2MzIyMzRjN30=",
    }
    e.myConfig = t
  },
  400,
  [],
)

// ...
```

Base64-decode the string to get the flag.

```bash
$ echo 'SFRCezIzbTQxbl9jNDFtXzRuZF9kMG43XzB2MzIyMzRjN30=' | base64 -d
```

Cheers, Mate :D

# References

---

- [GitHub: facebook/react-native](https://github.com/facebook/react-native)
- [reactnative.dev](https://reactnative.dev/)
- [Medium: What is index.android.bundle in React Native and Why You Shouldn’t Commit to Repository](https://medium.com/@naandalist/what-is-index-android-bundle-in-react-native-and-why-you-shouldnt-commit-to-repository-be5774b26652)
- [beautifier.io](https://beautifier.io/)
- [pilfer.github.io: Reverse Engineering and Instrumenting React Native Apps](https://pilfer.github.io/mobile-reverse-engineering/react-native/reverse-engineering-and-instrumenting-react-native-apps/)
