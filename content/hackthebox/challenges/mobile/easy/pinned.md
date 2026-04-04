---
title: pinned
description: Writeup for the pinned challenge on HackTheBox
created: 2025-10-12
tags: mobile, ctf, practice, hackthebox
draft: true
---

## Introduction

---

**Challenge Summary**:
**Challenge Description**: This app has stored my credentials and I can only login automatically. I tried to intercept the login request and restore my password, but this seems to be a secure connection. Can you help bypass this security restriction and intercept the password in plaintext?
**Challenge Category**: Mobile
**Challenge Difficulty**: Easy

## Challenge

---

### Setup

Based on the challenge description we'll want to intercept the HTTP request containing the password. This will require us to setup an Android Man-in-the-Middle SSL pinning bypass rig (what a mouthful).

Contained within the `README.txt` file:

> 1. Install this application in an API Level 29 or earlier (i.e. Android 10.0 (Google APIs)).

Lets install and boot an Android emulator which meets these requirements

```bash
# list potential system images
$ sdkmanager --list 2>&1 | grep "system-images;android-29"
  system-images;android-29;google_apis_playstore;arm64-v8a | 9       | Google Play ARM 64 v8a System Image | system-images/android-29/google_apis_playstore/arm64-v8a
  system-images;android-29;android-tv;x86                                         | 3                 | Android TV Intel x86 Atom System Image
  system-images;android-29;default;arm64-v8a                                      | 8                 | ARM 64 v8a System Image
  system-images;android-29;default;x86                                            | 8                 | Intel x86 Atom System Image
  system-images;android-29;default;x86_64                                         | 8                 | Intel x86_64 Atom System Image
  system-images;android-29;google_apis;arm64-v8a                                  | 13                | Google APIs ARM 64 v8a System Image
  system-images;android-29;google_apis;x86                                        | 13                | Google APIs Intel x86 Atom System Image
  system-images;android-29;google_apis;x86_64                                     | 13                | Google APIs Intel x86_64 Atom System Image
  system-images;android-29;google_apis_playstore;arm64-v8a                        | 9                 | Google Play ARM 64 v8a System Image
  system-images;android-29;google_apis_playstore;x86                              | 9                 | Google Play Intel x86 Atom System Image
  system-images;android-29;google_apis_playstore;x86_64                           | 9                 | Google Play Intel x86_64 Atom System Image

# download the system image
$ sdkmanager "system-images;android-29;google_apis;arm64-v8a"

# creating an Android emulator with android-29
$ avdmanager create avd -n pinned_emu -k "system-images;android-29;google_apis;arm64-v8a" --device "pixel_c"

# boot the Android emulator
$ emulator -avd pinned_emu

# install the APK onto the device
adb install pinned.apk
```

### Enumeration

As with the other mobile challenges, lets decompile the APK with `jadx`:

```bash
$ jadx ./pinned.apk
INFO  - loading ...
INFO  - processing ...
ERROR - finished with errors, count: 2
```

We'll navigate to and review the `AndroidManifest.xml` file.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0"
    android:compileSdkVersion="30"
    android:compileSdkVersionCodename="11"
    package="com.example.pinned"
    platformBuildVersionCode="30"
    platformBuildVersionName="11">
    <uses-sdk
        android:minSdkVersion="16"
        android:targetSdkVersion="30"/>
    <uses-permission android:name="android.permission.INTERNET"/>
    <application
        android:theme="@style/Theme.Pinned"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:allowBackup="true"
        android:supportsRtl="true"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:appComponentFactory="androidx.core.app.CoreComponentFactory">
        <activity android:name="com.example.pinned.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

The manifest is short and sweet. The name of the main package is `com.example.pinned`. It only uses a single permission - `android.permission.INTERNET`. We an assume then that there might be some network traffic leaving the application. Additionally, there is a single Activity - `com.example.pinned.MainActivity` with an intent-filter associated with it.

Lets navigate to the MainActivity and review that next.

### Reverse-Engineering

Most `MainActivity.java` classes have an `onCreate()` callback function. This is the first function invoked upon loading the class into the JVM.

```java
// ...

public void onCreate(Bundle bundle) {
    super.onCreate(bundle);
    setContentView(R.layout.activity_main);
    StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().permitAll().build());
    this.s = (TextView) findViewById(R.id.editTextTextPersonName);
    this.t = (TextView) findViewById(R.id.editTextTextPassword2);
    this.r = (TextView) findViewById(R.id.textView3);
    Button button = (Button) findViewById(R.id.button);
    this.u = button;
    button.setOnClickListener(new a());
}

// ...
```

The `onCreate()` function just creates handlers for the various text box's in the UI as well as the button. A class `a()` is set as a callback with `setOnClickListener()`. The logic will execute as soon as we hit the purple "Login" button.

### Defeating Certificate Pinning

### Alternative Solution

## References

---
