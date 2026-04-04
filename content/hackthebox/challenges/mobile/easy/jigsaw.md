---
title: jigsaw
description: Writeup for the Jigsaw mobile challenge on HackTheBox
created: 2025-09-20
tags: hackthebox, mobile, ctf, practice
draft: false
---

# Summary

---

Jigsaw is an easy Mobile challenge on HackTheBox that hinges on understanding the Google Flutter framework. Flutter lets developers build cross-platform mobile apps from a single codebase. Once you understand how Flutter works, you can zero in on the code that derives the flag. The challenge splits the flag across three components: native library code, Dart bytecode, and decompiled Java. Your task is to reverse-engineer the flag-generation algorithm across these segments and stitch the pieces together. Solving all three yields the final flag. Overall, this was one of my favorite Mobile HTB challenges. It’s packed with learning and forces you to draw on multiple areas of domain knowledge.

**Challenge Description**
A secret lies hidden, protected by layers of logic and scattered clues. Your task is to uncover these fragments, piece them together, and solve the mystery. It’s a challenge of patience, creativity, and determination. Can you reveal the secret?

**Difficulty**: Easy

# Challenge

---

## Recon

We start with some basic file enumeration:

```bash
$ file Jigsaw.apk
Jigsaw.apk: Zip archive data, at least v0.0 to extract, compression method=deflate
```

Next, let’s install the APK on an emulated device. I use `avdmanager` to create/manage AVDs and boot an emulator.

After installing Jigsaw.apk on the emulator, we launch the app and see the following:

---

<img src="/assets/hackthebox/challenges/mobile/easy/mobile_jigsaw/login-screen.png" alt="login-screen" width="400" />

---

Submitting random credentials triggers this notification:

---

<img src="/assets/hackthebox/challenges/mobile/easy/mobile_jigsaw/failed-login.png" alt="failed-login-screen" width="400">

---

That seems to be the extent of the in-app functionality for now. Let’s pivot to reverse-engineering the application to see what’s happening behind the scenes.

## Reverse-Engineering

Since an Android Package (APK) is just a ZIP archive, we can extract it with standard tools like unzip. For decompilation, specialized tools such as [jadx](https://github.com/skylot/jadx) and [apktool](https://apktool.org/) to translate DEX bytecode into readable sources.

For this walkthrough, I’ll use jadx:

```bash
$ jadx ./Jigsaw.apk
```

After a short run, jadx produces a Jigsaw/ directory with the decompiled sources.

### AndroidManifest.xml

A solid starting point for Android reverse engineering is the AndroidManifest.xml. Every app includes this file—it’s essentially the app’s blueprint, defining components, permissions, and required features.

Below are the contents of Jigsaw’s AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:versionCode="1"
    android:versionName="1.0.0"
    android:compileSdkVersion="35"
    android:compileSdkVersionCodename="15"
    package="com.example.menascyber"
    platformBuildVersionCode="35"
    platformBuildVersionName="15">
    <uses-sdk
        android:minSdkVersion="21"
        android:targetSdkVersion="35"/>
    <uses-permission android:name="android.permission.INTERNET"/>
    <queries>
        <intent>
            <action android:name="android.intent.action.PROCESS_TEXT"/>
            <data android:mimeType="text/plain"/>
        </intent>
    </queries>
    <permission
        android:name="com.example.menascyber.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"
        android:protectionLevel="signature"/>
    <uses-permission android:name="com.example.menascyber.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"/>
    <application
        android:label="Jigsaw"
        android:icon="@mipmap/ic_launcher"
        android:name="android.app.Application"
        android:debuggable="true"
        android:extractNativeLibs="true"
        android:appComponentFactory="androidx.core.app.CoreComponentFactory">
        <activity
            android:theme="@style/LaunchTheme"
            android:name="com.example.menascyber.MainActivity"
            android:exported="true"
            android:taskAffinity=""
            android:launchMode="singleTop"
            android:configChanges="fontScale|layoutDirection|density|smallestScreenSize|screenSize|uiMode|screenLayout|orientation|keyboardHidden|keyboard|locale"
            android:windowSoftInputMode="adjustResize"
            android:hardwareAccelerated="true">
            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme"/>
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <meta-data
            android:name="flutterEmbedding"
            android:value="2"/>
        <uses-library
            android:name="androidx.window.extensions"
            android:required="false"/>
        <uses-library
            android:name="androidx.window.sidecar"
            android:required="false"/>
        <provider
            android:name="androidx.startup.InitializationProvider"
            android:exported="false"
            android:authorities="com.example.menascyber.androidx-startup">
            <meta-data
                android:name="androidx.lifecycle.ProcessLifecycleInitializer"
                android:value="androidx.startup"/>
            <meta-data
                android:name="androidx.profileinstaller.ProfileInstallerInitializer"
                android:value="androidx.startup"/>
        </provider>
        <receiver
            android:name="androidx.profileinstaller.ProfileInstallReceiver"
            android:permission="android.permission.DUMP"
            android:enabled="true"
            android:exported="true"
            android:directBootAware="false">
            <intent-filter>
                <action android:name="androidx.profileinstaller.action.INSTALL_PROFILE"/>
            </intent-filter>
            <intent-filter>
                <action android:name="androidx.profileinstaller.action.SKIP_FILE"/>
            </intent-filter>
            <intent-filter>
                <action android:name="androidx.profileinstaller.action.SAVE_PROFILE"/>
            </intent-filter>
            <intent-filter>
                <action android:name="androidx.profileinstaller.action.BENCHMARK_OPERATION"/>
            </intent-filter>
        </receiver>
    </application>
</manifest>

```

### Flutter

In most Android apps, a good initial target is MainActivity. For Jigsaw, the decompiled source at mobile_jigsaw/Jigsaw/sources/com/example/menascyber/MainActivity.java shows a Flutter-based entry point that wires up a native Dart bridge via a MethodChannel:

```java
import io.flutter.embedding.android.FlutterActivity;
import io.flutter.embedding.engine.FlutterEngine;
import io.flutter.plugin.common.MethodCall;
import io.flutter.plugin.common.MethodChannel;

// ...

public void configureFlutterEngine(FlutterEngine flutterEngine) {
    Intrinsics.checkNotNullParameter(flutterEngine, "flutterEngine");
    super.configureFlutterEngine(flutterEngine);
    new MethodChannel(flutterEngine.getDartExecutor().getBinaryMessenger(), this.CHANNEL).setMethodCallHandler(new MethodChannel.MethodCallHandler() { // from class: com.example.menascyber.MainActivity$$ExternalSyntheticLambda0
        @Override // io.flutter.plugin.common.MethodChannel.MethodCallHandler
        public final void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
            MainActivity.configureFlutterEngine$lambda$0(methodCall, result);
        }
    });
}

// ...
```

When doing a security review, it’s always a good idea to inspect the app’s dependencies. Not only might you find a third-party library with a publicly disclosed vulnerability, but you’ll also develop a better understanding of the application.

In this case, the notable packages are `io.flutter.embedding.android`, `io.flutter.embedding.engine`, and `io.flutter.plugin.common`. The [`io.flutter.embedding.engine`](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/FlutterEngine.html) package sets up the Flutter execution environment—the engine that runs Dart code (via the Dart VM) and enables largely platform-independent logic. In practice, this lets developers target both Android and iOS from one codebase.

The other two packages are part of the Flutter framework as well. [`MethodChannel`](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodChannel.html) creates a named channel for asynchronous calls between platform code and Dart, and [`MethodCall`](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodCall.html) represents an individual call on that channel. Understanding these pieces clarifies the native–Dart boundary we’re reversing.

With that in mind, the code we saw earlier spins up a `FlutterEngine`, creates a `MethodChannel` (named `"parttwo"`), and registers a callback to handle incoming `MethodCall`s by invoking `MainActivity.configureFlutterEngine$lambda$0`. In short, this is the **receiver** for the channel.

A natural research question follows: **where is the sender?** Given the challenge description, and the channel name, it’s reasonable to assume the flag is split into multiple parts and this channel handles only one of them.

Messages are typically sent over a `MethodChannel` using `MethodChannel.invokeMethod`. Let’s search the decompiled sources for that call next.

```bash
$ rg -a -i -l "invokeMethod"
resources/assets/flutter_assets/kernel_blob.bin
sources/io/flutter/plugin/common/MethodChannel.java
sources/androidx/lifecycle/ClassesInfoCache.java
sources/io/flutter/embedding/engine/systemchannels/RestorationChannel.java
sources/io/flutter/embedding/engine/systemchannels/LocalizationChannel.java
sources/io/flutter/embedding/engine/systemchannels/NavigationChannel.java
sources/io/flutter/embedding/engine/systemchannels/BackGestureChannel.java
sources/io/flutter/embedding/engine/systemchannels/PlatformViewsChannel.java
sources/io/flutter/embedding/engine/systemchannels/PlatformChannel.java
sources/io/flutter/embedding/engine/systemchannels/TextInputChannel.java
```

Everything under `sources/io/flutter` belongs to the Flutter framework, so we can ignore those for now. That leaves `resources/assets/flutter_assets/kernel_blob.bin` and `sources/androidx/lifecycle/ClassesInfoCache.java`. [`ClassesInfoCache`](https://android.googlesource.com/platform/frameworks/support/+/a9ac247af2afd4115c3eb6d16c05bc92737d6305/lifecycle/common/src/main/java/androidx/lifecycle/ClassesInfoCache.java) is part of AndroidX, so we can also set it aside.

That points us to `kernel_blob.bin`. Let’s dig into that next.

Let’s figure out what this file is and where it’s used. If we `cat` it, we get a wall of bytes. A quick search shows that [`kernel_blob.bin`](https://stackoverflow.com/questions/53368586/what-is-flutters-kernel-blob-bin) contains the app’s Dart code. We can run `strings` on it and save the output to review much of the embedded Dart:

```bash
strings kernel_blob.bin > kernel_blob-strings.txt
```

I also wanted a deeper understanding of how Flutter wires this up, so I looked for where the Dart bytecode gets loaded by searching for `kernel_blob.bin`.

```bash
$ rg -a -i -l "kernel_blob.bin"
resources/META-INF/CERT.SF
resources/META-INF/MANIFEST.MF
sources/io/flutter/embedding/engine/loader/FlutterLoader.java
```

The first two are flat files containing signatures and metadata for APK contents. The interesting hit is [`FlutterLoader.java`](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/loader/FlutterLoader.html), which is responsible for loading Flutter resources. Notably, `ensureInitializationComplete` blocks the main thread until the Flutter native resources are fully initialized.

```java
public void ensureInitializationComplete(Context applicationContext, String[] args) {
    int oldGenHeapSizeMegaBytes;
    if (this.initialized) {
        return;
    }
    if (Looper.myLooper() != Looper.getMainLooper()) {
        throw new IllegalStateException("ensureInitializationComplete must be called on the main thread");
    }
    if (this.settings == null) {
        throw new IllegalStateException("ensureInitializationComplete must be called after startInitialization");
    }
    try {
        TraceSection e = TraceSection.scoped("FlutterLoader#ensureInitializationComplete");
        try {
            InitResult result = this.initResultFuture.get();
            List<String> shellArgs = new ArrayList<>();
            shellArgs.add("--icu-symbol-prefix=_binary_icudtl_dat");
            shellArgs.add("--icu-native-lib-path=" + this.flutterApplicationInfo.nativeLibraryDir + File.separator + DEFAULT_LIBRARY);
            if (args != null) {
                Collections.addAll(shellArgs, args);
            }
            String snapshotAssetPath = result.dataDirPath + File.separator + this.flutterApplicationInfo.flutterAssetsDir;
            String kernelPath = snapshotAssetPath + File.separator + DEFAULT_KERNEL_BLOB;
            shellArgs.add("--snapshot-asset-path=" + snapshotAssetPath);
            shellArgs.add("--vm-snapshot-data=" + this.flutterApplicationInfo.vmSnapshotData);
            shellArgs.add("--isolate-snapshot-data=" + this.flutterApplicationInfo.isolateSnapshotData);
            shellArgs.add("--cache-dir-path=" + result.engineCachesPath);
            if (this.flutterApplicationInfo.domainNetworkPolicy != null) {
                shellArgs.add("--domain-network-policy=" + this.flutterApplicationInfo.domainNetworkPolicy);
            }
            if (this.settings.getLogTag() != null) {
                shellArgs.add("--log-tag=" + this.settings.getLogTag());
            }
            ApplicationInfo applicationInfo = applicationContext.getPackageManager().getApplicationInfo(applicationContext.getPackageName(), 128);
            Bundle metaData = applicationInfo.metaData;
            int oldGenHeapSizeMegaBytes2 = metaData != null ? metaData.getInt(OLD_GEN_HEAP_SIZE_META_DATA_KEY) : 0;
            if (oldGenHeapSizeMegaBytes2 == 0) {
                ActivityManager activityManager = (ActivityManager) applicationContext.getSystemService("activity");
                ActivityManager.MemoryInfo memInfo = new ActivityManager.MemoryInfo();
                activityManager.getMemoryInfo(memInfo);
                oldGenHeapSizeMegaBytes = (int) ((memInfo.totalMem / 1000000.0d) / 2.0d);
            } else {
                oldGenHeapSizeMegaBytes = oldGenHeapSizeMegaBytes2;
            }
            shellArgs.add("--old-gen-heap-size=" + oldGenHeapSizeMegaBytes);
            DisplayMetrics displayMetrics = applicationContext.getResources().getDisplayMetrics();
            int screenWidth = displayMetrics.widthPixels;
            int screenHeight = displayMetrics.heightPixels;
            int resourceCacheMaxBytesThreshold = screenWidth * screenHeight * 12 * 4;
            shellArgs.add("--resource-cache-max-bytes-threshold=" + resourceCacheMaxBytesThreshold);
            shellArgs.add("--prefetched-default-font-manager");
            if (metaData != null) {
                if (metaData.containsKey(ENABLE_IMPELLER_META_DATA_KEY)) {
                    if (metaData.getBoolean(ENABLE_IMPELLER_META_DATA_KEY)) {
                        shellArgs.add(FlutterShellArgs.ARG_ENABLE_IMPELLER);
                    } else {
                        shellArgs.add(FlutterShellArgs.ARG_DISABLE_IMPELLER);
                    }
                }
                if (metaData.getBoolean(ENABLE_VULKAN_VALIDATION_META_DATA_KEY, false)) {
                    shellArgs.add(FlutterShellArgs.ARG_ENABLE_VULKAN_VALIDATION);
                }
                if (metaData.getBoolean(IMPELLER_OPENGL_GPU_TRACING_DATA_KEY, false)) {
                    shellArgs.add("--enable-opengl-gpu-tracing");
                }
                if (metaData.getBoolean(IMPELLER_VULKAN_GPU_TRACING_DATA_KEY, false)) {
                    shellArgs.add("--enable-vulkan-gpu-tracing");
                }
                if (metaData.getBoolean(DISABLE_SURFACE_CONTROL, false)) {
                    shellArgs.add("--disable-surface-control");
                }
                if (metaData.containsKey(ENABLED_MERGED_PLATFORM_UI_THREAD_KEY)) {
                    if (metaData.getBoolean(ENABLED_MERGED_PLATFORM_UI_THREAD_KEY)) {
                        shellArgs.add("--enable-merged-platform-ui-thread");
                    } else {
                        shellArgs.add("--no-enable-merged-platform-ui-thread");
                    }
                }
                String backend = metaData.getString(IMPELLER_BACKEND_META_DATA_KEY);
                if (backend != null) {
                    shellArgs.add("--impeller-backend=" + backend);
                }
            }
            String leakVM = isLeakVM(metaData) ? "true" : "false";
            shellArgs.add("--leak-vm=" + leakVM);
            long initTimeMillis = SystemClock.uptimeMillis() - this.initStartTimestampMillis;
            this.flutterJNI.init(applicationContext, (String[]) shellArgs.toArray(new String[0]), kernelPath, result.appStoragePath, result.engineCachesPath, initTimeMillis);
            this.initialized = true;
            if (e != null) {
                e.close();
            }
        } catch (Throwable th) {
            if (e == null) {
                throw th;
            }
            try {
                e.close();
                throw th;
            } catch (Throwable th2) {
                th.addSuppressed(th2);
                throw th;
            }
        }
    } catch (Exception e2) {
        Log.e(TAG, "Flutter initialization failed.", e2);
        throw new RuntimeException(e2);
    }
}
```

Enough Java—let’s pivot to the Dart side. We’ll dive into the bytecode next, using the `strings` dump of `kernel_blob.bin` to hunt for channel names, method calls, and any flag-building logic embedded in the Dart code.

### Dart Bytecode

We’ve already dumped the Dart bytecode from `kernel_blob.bin` using `strings`, so now we can start cleaning and analyzing it. There’s quite a bit of output to sift through. Flutter embeds a large amount of generated code. According to Flutter’s documentation, when the Dart bytecode is loaded into the virtual machine, the `main` symbol serves as the entry point. Let’s begin by searching for that symbol to locate where execution starts.

```dart
// ...

void main() {
    runApp(MyApp());

    // ...
}
```

This simply calls `runApp()` with `MyApp()` as the root widget. Next step: locate the `MyApp` class and see how it initializes the app (typically via `build()` returning a `MaterialApp` or similar) to uncover where any flag-related logic or channel wiring might begin.

```dart
// ...

class MyApp extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
        return MaterialApp(
            title: 'Login App',
            theme: ThemeData(
                primarySwatch: Colors.blue,
            ),
            home: LoginPage(),
        );
    }

    // ...

}

    // ...
```

`MyApp` sets home to `LoginPage()`, which matches what we saw in the emulator. Let’s jump into the `LoginPage` class next.

```dart
// ...

class LoginPage extends StatefulWidget {
    @override
    _LoginPageState createState() => _LoginPageState();

}

// ...

class _LoginPageState extends State<LoginPage> {
    final TextEditingController _usernameController = TextEditingController();
    final TextEditingController _passwordController = TextEditingController();
    final AESCombinedService _aesCombinedService = AESCombinedService();

    Future<void> fetchAndDecryptFlag() async {
        try {
            final finallyClass = Finally();
            final decryptedFlag = await finallyClass.decryptFlag();

        } catch (e) {

        }
    }

    void _login() {
        // Check if the username and password are correct
        if (_usernameController.text == 'nimda' && _passwordController.text == 'guessme') {

            Navigator.push(
                context,
                MaterialPageRoute(builder: (context) => HomePage(username: _usernameController.text)),
            );
        } else {
            showDialog(
                context: context,
                builder: (context) {
                    return AlertDialog(
                        title: Text('Login Failed'),
                        content: Text('Crack me and find the flag'),
                        actions: [
                            TextButton(
                                onPressed: () {
                                    Navigator.of(context).pop();
                                },
                                child: Text('OK'),
                            ),
                        ],
                    );
                },
            );
        }
    }

    @override
    Widget build(BuildContext context) {
        // ...
    }
}
// ...
```

There’s a `_login()` function that hardcodes credentials. Using `nimda` / `guessme` yields:

---

<img src="/assets/hackthebox/challenges/mobile/easy/mobile_jigsaw/no-flag.png" alt="login-screen" width="400" />

---

No luck. :\

More interesting is `fetchAndDecryptFlag()`, which invokes `decryptFlag()` in the `Finally` class. Let’s extract that class from the `kernel_blob.bin` strings dump and inspect the implementation.

```dart
// ...

import 'dart:convert';
import 'dart:typed_data';
import 'package:encrypt/encrypt.dart' as encrypt;
import 'package:jigsaw/services.dart'; // Import the file containing AESCombinedService
class Finally {
    final String _encryptedFlagBase64 = 'aZ/KF0GsnN81j5XStQyKz3vXtktTVN5zFqy5lwTmub6fx5w70c+p08O0OWcn/9nh';
    Future<String> decryptFlag() async {
        final aesService = AESCombinedService();
        final flagData = await aesService.getflag();
        final encryptedFlagBytes = _base64ToBytes(_encryptedFlagBase64);
        final key = Uint8List.fromList(flagData['key']!);
        final iv = Uint8List.fromList(flagData['iv']!);
        final encryptKey = encrypt.Key(key);
        final encryptIV = encrypt.IV(iv);
        final encrypter = encrypt.Encrypter(encrypt.AES(encryptKey, mode: encrypt.AESMode.cbc));
        final decrypted = encrypter.decrypt(
            encrypt.Encrypted.fromBase64(_encryptedFlagBase64),
            iv: encryptIV,
        );

        //return decrypted;
        String message = "Developer forgot to uncomment";
        return  message;
    }

        // Helper function to convert base64 string to bytes
    List<int> _base64ToBytes(String base64) {
        final bytes = base64Decode(base64);
        return bytes;
    }

    String bytesToHex(Uint8List bytes) {
        final buffer = StringBuffer();
        for (var byte in bytes) {
            buffer.write(byte.toRadixString(16).padLeft(2, '0'));
        }
        return buffer.toString();
    }
}

// ...
```

`decryptFlag()` delegates to `AESCombinedService.getFlag()`, which embeds a key and IV. That method base64-decodes the hardcoded `_encryptedFlagBase64` and then decrypts it. To understand where the key and IV originate—and whether they’re static or derived at runtime, we need to examine `AESCombinedService.getFlag()` next.

```dart
class AESCombinedService {
    final AESNativeLib _nativeLib = AESNativeLib.instance;
    final AESService _aesService = AESService();

    // Hardcoded key and IV
    final List<int> _hardcodedKey = List<int>.generate(32, (i) => (i + 1) % 256);
    final List<int> _hardcodedIV = List<int>.generate(16, (i) => (i + 10) % 256);

    // Deterministic shuffling method
    List<int> _deterministicShuffle(List<int> input, int shift) {
        return List<int>.generate(input.length, (i) {
            return input[(i + shift) % input.length];
        });

    }

    Future<Map<String, List<int>>> partone() async {
        // Shuffle the key and IV deterministically
        final shuffledKey = _deterministicShuffle(_hardcodedKey, 5);
        final shuffledIV = _deterministicShuffle(_hardcodedIV, 3);
        return {
            "key": shuffledKey,
            "iv": shuffledIV,
        };
    }

    Future<Map<String, List<int>>> getflag() async {
        final partoneData = await partone();
        final parttwoData = await _aesService.getparttwo();
        final partthreeKey = _nativeLib.getAESKey();
        final partthreeIV = _nativeLib.getAESIV();
        // Combine and slice the key and IV from each part
        final combinedKey = [
            ...partoneData['key']!.sublist(0, 8),
            ...parttwoData['key']!.sublist(0, 8),
            ...partthreeKey.sublist(0, 16)
        ];
        final combinedIV = [
            ...partoneData['iv']!.sublist(0, 4),
            ...parttwoData['iv']!.sublist(0, 4),
            ...partthreeIV.sublist(0, 8)
        ];
        return {
            "key": combinedKey,
            "iv": combinedIV,
        };
    }
}
```

`getFlag()` calls four helpers `partone()`, `_aesService.getparttwo()`, `_nativeLib.getAESKey()`, and `_nativeLib.getAESIV()`. The values returned by these functions are combined and massaged to produce the final AES key and IV used to decrypt the flag. Let’s examine each piece in turn.

### partone

We’ll start with part one of the key. The logic lives alongside `AESCombinedService` in the Dart code:

```dart
// ...

// Hardcoded key and IV
final List<int> _hardcodedKey = List<int>.generate(32, (i) => (i + 1) % 256);
final List<int> _hardcodedIV  = List<int>.generate(16, (i) => (i + 10) % 256);

// Deterministic shuffling method
List<int> _deterministicShuffle(List<int> input, int shift) {
    return List<int>.generate(input.length, (i) {
        return input[(i + shift) % input.length];
    });
}

Future<Map<String, List<int>>> partone() async {
    // Shuffle the key and IV deterministically
    final shuffledKey = _deterministicShuffle(_hardcodedKey, 5);
    final shuffledIV  = _deterministicShuffle(_hardcodedIV, 3);
    return {
        "key": shuffledKey,
        "iv": shuffledIV,
    };
}

// ...
```

`partone()` calls `_deterministicShuffle()` on `_hardcodedKey` and `_hardcodedIV`, effectively rotating each list by a fixed offset (`shift`). We can mirror this logic in Python to reproduce the same values:

```python
# ...

def deterministic_shuffle(arr, shift):
    n = len(arr)
    return [arr[(i + shift) % n] for i in range(n)]

def partone():
    key = [(i + 1) % 256 for i in range(32)]
    iv  = [(i + 10) % 256 for i in range(16)]
    return deterministic_shuffle(key, 5), deterministic_shuffle(iv, 3)

# ...
```

### parttwo

Next up is `_aesService.getparttwo()`, defined in the `AESService` class:

```dart
class AESService {
    static const platform = MethodChannel('parttwo');

    Future<Map<String, List<int>>> getparttwo() async {
        try {
            final Map<dynamic, dynamic> result = await platform.invokeMethod('parttwo');
            return {
                "key": List<int>.from(result['key']),
                "iv":  List<int>.from(result['iv']),
            };
        } on PlatformException catch (e) {
            print("Failed to get parttwo: '${e.message}'.");
            return {};
        }
    }
}
```

This uses `invokeMethod()` to send a `MethodCall` over the `"parttwo"` `MethodChannel`, expecting a map with `key` and `iv`. Earlier we saw that the receiving end lives in `mobile_jigsaw/Jigsaw/sources/com/example/menascyber/MainActivity.java`, where `configureFlutterEngine` wires a handler that dispatches to `MainActivity.configureFlutterEngine$lambda$0`:

```java
// ...

public static final void configureFlutterEngine$lambda$0(MethodCall call, MethodChannel.Result result) {
    Intrinsics.checkNotNullParameter(call, "call");
    Intrinsics.checkNotNullParameter(result, "result");
    if (Intrinsics.areEqual(call.method, "parttwo")) {
        Map response = MainActivityKt.get_parttwo();
        result.success(response);
    } else {
        result.notImplemented();
    }
}

// ...
```

If `call.method` equals `"parttwo"`, it calls `MainActivityKt.get_parttwo()` and returns the result.

I’ll admit I didn’t fully read the native implementation here. Instead, I took a shortcut: since a `Map<String, byte[]>` is returned on success, I wrote a Frida script to invoke and capture the return value directly.

```javascript
// name:        parttwo.js
// description: Hook MainActivityKt.get_parttwo() and print the key/iv pair

Java.perform(() => {
  function bytes_to_hex(jBarr) {
    const a = Java.array("byte", jBarr)
    let out = ""
    for (let i = 0; i < a.length; i++) {
      out += ("0" + (a[i] & 0xff).toString(16)).slice(-2)
    }
    return out
  }

  const Map = Java.use("java.util.Map")
  const Entry = Java.use("java.util.Map$Entry")
  const MainActivityKt = Java.use("com.example.menascyber.MainActivityKt").$new()

  // invoke and capture the retval from MainActivityKt.get_parttwo()
  const map = MainActivityKt.get_parttwo()
  if (!map) {
    console.error("Could not get key/iv map")
    return
  }

  // convert the Java Map<String, byte[]> to a JS object and print
  const m = Java.cast(map, Map)
  const it = m.entrySet().iterator()
  while (it.hasNext()) {
    const e = Java.cast(it.next(), Entry)
    const k = e.getKey().toString()
    const v = e.getValue()
    console.log(`${k} => ${bytes_to_hex(v)}`)
  }
})
```

Run it:

```
$ frida -U -f com.example.menascyber -l ./parttwo.js

...

key => 47caf2f3f1acf8effff24bfef24be9f9a8cb
iv  => f745c7c94b4dce
```

Now we can drop this key/IV pair into the Python script:

```python
# ...

def hex_str_to_array(hex_str, chunk_size=2):
    arr = []
    for i in range(0, len(hex_str), chunk_size):
        hex_chunk = hex_str[i:i+chunk_size]
        val = int(hex_chunk, 16)
        arr.append(val)
    return arr

print("********** part two **********")
part_two_key = hex_str_to_array("47caf2f3f1acf8effff24bfef24be9f9a8cb")
part_two_iv  = hex_str_to_array("f745c7c94b4dce")
print(f"key: {part_two_key}")
print(f"iv:  {part_two_iv}")

# ...
```

### partthree

Last, we tackle `partthree`. This piece uses `_nativeLib.getAESKey()` and `_nativeLib.getAESIV()` to pull the key/IV pair from native code:

```dart
class AESNativeLib {
  final DynamicLibrary _dylib;

  // Private constructor
  AESNativeLib._(this._dylib);

  // Singleton instance
  static AESNativeLib? _instance;

  // Load the shared library and return the instance
  static AESNativeLib get instance {
    _instance ??= AESNativeLib._(DynamicLibrary.open('libmenascyber.so'));
    return _instance!;
  }

  List<int> getAESKey() {
    final getAESKey =
        _dylib.lookupFunction<GetAESKeyNative, GetAESKeyNative>('partthree_1');
    final keyPointer = getAESKey();
    return keyPointer.asTypedList(32);
  }

  List<int> getAESIV() {
    final getAESIV =
        _dylib.lookupFunction<GetAESIVNative, GetAESIVNative>('partthree_2');
    final ivPointer = getAESIV();
    return ivPointer.asTypedList(16);
  }
}
```

This class opens a handle to the native shared object `libmenascyber.so`. When `getAESKey()` is called, it looks up and invokes the exported native function `partthree_1()`, returning a 32-byte slice. Likewise, `getAESIV()` calls `partthree_2()`, returning 16 bytes.

I took the fast path here: rather than reverse-engineering the native library, I used Frida to call the exports directly and read their results.

```js
// name: partthree.js
// description: Invoke partthree_1 and partthree_2 and capture the return values

setTimeout(function () {
  function find_library_base(library_name) {
    const base_address = Module.findBaseAddress(library_name)
    if (!base_address) return null
    return base_address
  }

  function bytes_to_hex(bArr) {
    const u8 = new Uint8Array(bArr)
    let s = ""
    for (let i = 0; i < u8.length; i++) s += u8[i].toString(16).padStart(2, "0")
    return s
  }

  Java.perform(() => {
    const module_name = "libmenascyber.so"

    const lib_base = find_library_base(module_name)
    if (!lib_base) {
      console.error(`Could not find base of ${module_name}`)
      return
    }

    console.log(`${module_name} @ ${lib_base}`)

    // resolve the addresses of the native symbols we need.
    const partthree_1_addr = Module.findExportByName(module_name, "partthree_1")
    const partthree_2_addr = Module.findExportByName(module_name, "partthree_2")

    console.log(`partthree_1 @ ${partthree_1_addr}`)
    console.log(`partthree_2 @ ${partthree_2_addr}`)

    // wrap them as NativeFunction so we can invoke them.
    const partthree_1_nf = new NativeFunction(partthree_1_addr, "pointer", [])
    const partthree_2_nf = new NativeFunction(partthree_2_addr, "pointer", [])

    // invoke partthree_1 (returns a pointer in .data)
    const partthree_1_ret = partthree_1_nf()
    if (partthree_1_ret.isNull()) {
      console.error("partthree_1_nf returned NULL")
      return
    }
    console.log(`partthree_1() -> ${partthree_1_ret}`)

    const key_bArr = Memory.readByteArray(partthree_1_ret, 32)
    const key = bytes_to_hex(key_bArr)

    // invoke partthree_2
    const partthree_2_ret = partthree_2_nf()
    if (partthree_2_ret.isNull()) {
      console.error("partthree_2_nf returned NULL")
      return
    }
    console.log(`partthree_2() -> ${partthree_2_ret}`)

    const iv_bArr = Memory.readByteArray(partthree_2_ret, 16)
    const iv = bytes_to_hex(iv_bArr)

    console.log(`key: ${key}`)
    console.log(`iv: ${iv}`)
  })
}, 5000)
```

Run the script and capture the results:

```
frida -U -f com.example.menascyber -l ./partthree.js

...

libmenascyber.so @ 0x726018a000
partthree_1 @ 0x726018a864
partthree_2 @ 0x726018a8a4
partthree_1() -> 0x726018cb60
partthree_2() -> 0x726018cb80
key: 0b4dcfd15355d6d85b5ddfc14345c6c8494fcdd35157d4da0323436383a3c3e3
iv: 5751d3dd5f59dac44741c3cd4f49cad4
```

Now insert those into the Python script:

```python
# ...

print("********** part three **********")
part_three_key = hex_str_to_array("0b4dcfd15355d6d85b5ddfc14345c6c8494fcdd35157d4da0323436383a3c3e3")
part_three_iv  = hex_str_to_array("5751d3dd5f59dac44741c3cd4f49cad4")
print(f"key: {part_three_key}")
print(f"iv:  {part_three_iv}")

# ...
```

## Putting Everything Together

The final step is to stitch everything together exactly as the Dart code does. In the Dart bytecode, `getflag()` concatenates slices from each part to build the AES key and IV:

```dart
// ...

Future<Map<String, List<int>>> getflag() async {
  final partoneData  = await partone();
  final parttwoData  = await _aesService.getparttwo();
  final partthreeKey = _nativeLib.getAESKey();
  final partthreeIV  = _nativeLib.getAESIV();

  // Combine and slice the key and IV from each part
  final combinedKey = [
    ...partoneData['key']!.sublist(0, 8),
    ...parttwoData['key']!.sublist(0, 8),
    ...partthreeKey.sublist(0, 16),
  ];
  final combinedIV = [
    ...partoneData['iv']!.sublist(0, 4),
    ...parttwoData['iv']!.sublist(0, 4),
    ...partthreeIV.sublist(0, 8),
  ];

  return {
    "key": combinedKey,
    "iv":  combinedIV,
  };
}

// ...
```

We can mirror that logic in Python:

```python
# ...

combined_key = []
combined_key += part_one_key[0:8]
combined_key += part_two_key[0:8]
combined_key += part_three_key[0:16]
key = bytes(combined_key)

combined_iv = []
combined_iv += part_one_iv[0:4]
combined_iv += part_two_iv[0:4]
combined_iv += part_three_iv[0:8]
iv = bytes(combined_iv)

print("********** combined **********")
print(f"key: {key}")
print(f"iv:  {iv}")

# ...
```

With the key and IV assembled, decode and decrypt the flag:

```python
# ...

ciphertext_flag = b"aZ/KF0GsnN81j5XStQyKz3vXtktTVN5zFqy5lwTmub6fx5w70c+p08O0OWcn/9nh"
ciphertext_flag = base64.b64decode(ciphertext_flag)

cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext_flag = unpad(cipher.decrypt(ciphertext_flag), AES.block_size)
print(f"Flag: {plaintext_flag.decode()}")

# ...
```

That reproduces the Dart logic end-to-end: assemble the key/IV from the three parts, base64-decode the ciphertext, then decrypt (AES-CBC) and unpad to reveal the flag.

<details>
    <summary>solve.py</summary>

```python
#!/usr/bin/env python3

import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

def deterministic_shuffle(input, shift):
    input_length = len(input)
    shuffled = [input[(i + shift) % input_length] for i in range(input_length)]
    return shuffled

def partone():
    key = [((i + 1) % 256) for i in range(32)]
    iv = [((i + 10) % 256) for i in range(16)]

    return deterministic_shuffle(key, 5), deterministic_shuffle(iv, 3)

def hex_str_to_array(hex_str, chunk_size=2):
    arr = []
    for i in range(0, len(hex_str), chunk_size):
        hex_chunk = hex_str[i:i+chunk_size]
        val = int(hex_chunk, 16)
        arr.append(val)
    return arr

def main()-> None:
    print("********** part one **********")
    part_one_key, part_one_iv = partone()
    print(f"key: {part_one_key}")
    print(f"iv: {part_one_iv}")

    print("********** part two **********")
    part_two_key = hex_str_to_array("47caf2f3f1acf8effff24bfef24be9f9a8cb")
    part_two_iv = hex_str_to_array("f745c7c94b4dce")
    print(f"key {part_two_key}")
    print(f"iv: {part_two_iv}")

    print("********** part three **********")
    part_three_key = hex_str_to_array("0b4dcfd15355d6d85b5ddfc14345c6c8494fcdd35157d4da0323436383a3c3e3")
    part_three_iv = hex_str_to_array("5751d3dd5f59dac44741c3cd4f49cad4")
    print(f"key: {part_three_key}")
    print(f"iv: {part_three_iv}")

    combined_key = []
    combined_key += part_one_key[0:8]
    combined_key += part_two_key[0:8]
    combined_key += part_three_key[0:16]
    key = bytes(combined_key)

    combined_iv = []
    combined_iv += part_one_iv[0:4]
    combined_iv += part_two_iv[0:4]
    combined_iv += part_three_iv[0:8]
    iv = bytes(combined_iv)

    print("********** combined **********")
    print(f"key: {key}")
    print(f"iv: {iv}")

    ciphertext_flag = b"aZ/KF0GsnN81j5XStQyKz3vXtktTVN5zFqy5lwTmub6fx5w70c+p08O0OWcn/9nh"
    ciphertext_flag = base64.b64decode(ciphertext_flag)
    cipher = AES.new(key, AES.MODE_CBC, iv)
    plaintext_flag = unpad(cipher.decrypt(ciphertext_flag), AES.block_size)
    print(f"Flag: {plaintext_flag.decode()}")

if __name__ == "__main__":
    main()
```

</details>

# References

---

1. **jadx** [https://github.com/skylot/jadx](https://github.com/skylot/jadx)
2. **Apktool** [https://apktool.org/](https://apktool.org/)
3. **`FlutterEngine`** [https://api.flutter.dev/javadoc/io/flutter/embedding/engine/FlutterEngine.html](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/FlutterEngine.html)
4. **`MethodChannel`** [https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodChannel.html](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodChannel.html)
5. **`MethodCall`** [https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodCall.html](https://api.flutter.dev/javadoc/io/flutter/plugin/common/MethodCall.html)
6. **`FlutterLoader`** [https://api.flutter.dev/javadoc/io/flutter/embedding/engine/loader/FlutterLoader.html](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/loader/FlutterLoader.html)
7. **`ClassesInfoCache.java`** [https://android.googlesource.com/platform/frameworks/support/+/a9ac247af2afd4115c3eb6d16c05bc92737d6305/lifecycle/common/src/main/java/androidx/lifecycle/ClassesInfoCache.java](https://android.googlesource.com/platform/frameworks/support/+/a9ac247af2afd4115c3eb6d16c05bc92737d6305/lifecycle/common/src/main/java/androidx/lifecycle/ClassesInfoCache.java)
