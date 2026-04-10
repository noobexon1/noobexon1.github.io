+++
title = "Frida on Android 0x2 - Hooking Native Libraries"
date = 2026-04-10
draft = true
tags = ["Frida", "Android", "Native", "Hooking", "Frida on Android"]
image = "/images/logos/frida_on_android_logo.png"
+++


In this part of the **Frida on Android** series, I'll show you how to **properly** hook functions from Android native libraries using `Frida`. I created a demo application to demonstrate the issues that can arise from doing things wrong, and explain why and how the method I'm using solves them.

## Demo App Overview

The demo application's source code can be found at the [GitHub repository](https://github.com/noobexon1/NativeExample). You can clone and build the application yourself, or you can simply download the prebuilt `APK` file from the repository's [releases](https://github.com/noobexon1/NativeExample/releases/tag/demo) page.

Install it on your device, and let's breifly go over the code.

### `nativeexample.cpp`

This is the native code of the application. Compiled into a native library named `"nativeexample.so"`.

#### Code

```C
#include <jni.h>

extern "C" JNIEXPORT jboolean JNICALL
Java_com_noobexon_nativeexample_MainActivity_should_1say_1hello(
    JNIEnv *env,
    jobject thiz
) {
    return JNI_FALSE;
}
```

#### Flow

1. Declares `should_say_hello()`: a simple boolean native function that always returns `false`.

### `MainActivity.java`

This is the code for the main activity of the application.

#### Code

```Java
package com.noobexon.nativeexample;

import android.os.Bundle;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {

    static {
        System.loadLibrary("nativeexample");
    }

    private native boolean should_say_hello();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (should_say_hello()) {
            Toast.makeText(this, "Hello from native!", Toast.LENGTH_LONG).show();
        } else {
            Toast.makeText(this, "Nope!", Toast.LENGTH_LONG).show();
            finishAffinity();
            android.os.Process.killProcess(android.os.Process.myPid());
        }
    }
}
```

#### Flow

1. App starts.
2. Native library `nativeexample.so` is loaded.
3. Boolean native function `should_say_hello()` is called from `nativeexample.so`.
4. Based on the return value:
- `true` Returned  -> Shows the message `"Hello from native!"` in a `Toast`.
- `false` Returned -> Shows the message `"Nope!"` in a `Toast`. The application is also immediately killed.

## Target

As things stands now, the function `should_say_hello()` always returns `false`, making the app immediately kill itself.

Our goal is to use `Frida` to hook said function, so that it returns `true`. If we manage to do that, a different message will be shown and the app won't kill itself on launch.

## First Attempt (Fail)

## Second Attempt (Success)

## Conclusion & Template

### Frida Snippet

Here is the complete `Frida` snippet:

```javascript
(function () {
    // ** Change to target library name... **
    const targetLibName = "MODULE_NAME_PLACEHOLDER"; 

    const dlopenAddr = Module.getGlobalExportByName("android_dlopen_ext");
    if (dlopenAddr != null) {
        Interceptor.attach(dlopenAddr, {
            onEnter: function (args) {
                const loadPath = args[0].readCString();
                if (loadPath != null && loadPath.includes(targetLibName)) {
                    this.isTarget = true;
                }
            },
            onLeave: function () {
                if (this.isTarget) {
                    console.log(`[+] ${targetLibName} loaded!`);

                    // ** Your code goes in here... **

                    this.isTarget = false;
                }
            }
        });
    }
})();
```

To use it, simply modify the values as shown in the comments

### Code Breakdown