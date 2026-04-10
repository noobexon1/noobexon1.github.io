+++
title = "Frida on Android 0x2 - Hooking Native Libraries"
date = 2026-04-10
draft = false
tags = ["Frida", "Android", "Native", "Hooking", "Frida on Android"]
image = "/images/logos/frida_on_android_logo.png"
+++


In this part of the **Frida on Android** series, I'll show you how to **properly** hook functions from Android native libraries using `Frida`. I created a demo application to demonstrate the issues that can arise from doing things wrong, and I'll explain why and how the method I'm using solves them.

## Demo App Overview

The demo application's source code can be found at the [GitHub repository](https://github.com/noobexon1/NativeExample). You can clone and build the application yourself, or you can simply download the prebuilt `APK` file from the repository's [releases](https://github.com/noobexon1/NativeExample/releases/tag/demo) page.

Install it on your device, run it once or twice, and let's briefly go over the code.

### `nativeexample.cpp`

This is the native code of the application. Compiled into a native library named `"libnativeexample.so"`.

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
2. Native library `libnativeexample.so` is loaded.
3. Boolean native function `should_say_hello()` is called from `libnativeexample.so`.
4. Based on the return value:
    - `true` Returned  -> Shows the message `"Hello from native!"` in a `Toast`.
    - `false` Returned -> Shows the message `"Nope!"` in a `Toast`. The application is also immediately killed.

## Goal

As things stand now, the function `should_say_hello()` always returns `false`, making the app immediately kill itself.

Our goal is to use `Frida` to hook said function, so that it returns `true`. If we manage to do that, a different message will be shown and the app won't kill itself on launch.

## Hook Requirements

To hook a function from a native library, we need to supply `Frida` with two pieces of information:

1. The memory base address where the native library is loaded.
2. The offset of the target function from that base address.

Why? Because `Frida` must know the exact place where it needs to install the hook. That exact place is the absolute memory address of the function. To get the absolute memory address of the function, we can compute the following:

`[func_absolute_addr] = [lib_base_addr] + [func_offset_from_base]`

Lucky for us, `Frida` can enumerate the application's memory map at runtime and give us the base address of the library for free. Therefore, all that remains for us to do is find the offset of the function from the native library base address and we are set.

### Finding the Offset

To find the offset, you can use your preferred disassembler or decompiler to explore `libnativeexample.so`:

1. Open a terminal and navigate to the folder in which you downloaded the `apk` file:

<img src="/images/screenshots/hooking_native_functions/offset_1.png" width="100%">

2. Decompile it using [apktool](https://apktool.org/). You should see an output folder:

<img src="/images/screenshots/hooking_native_functions/offset_2.png" width="100%">
<img src="/images/screenshots/hooking_native_functions/offset_3.png" width="100%">

3. Inside of the output folder, you can find the native library at the following location:

<img src="/images/screenshots/hooking_native_functions/offset_4.png" width="100%">

4. Open the native library in your decompiler (I'll be using [Binary Ninja](https://binary.ninja/)). You should immediately see our target function. Click on it:

<a href="/images/screenshots/hooking_native_functions/offset_5.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/offset_5.png" width="100%">
</a>

<a href="/images/screenshots/hooking_native_functions/offset_6.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/offset_6.png" width="100%">
</a>

5. Make sure the decompiler is configured to show you the offset of the function from the image base:

<a href="/images/screenshots/hooking_native_functions/offset_7.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/offset_7.png" width="100%">
</a>

6. Once verified, the offset is as seen here:

<a href="/images/screenshots/hooking_native_functions/offset_8.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/offset_8.png" width="100%">
</a>


## First Hook Attempt (Fail)

Now that we have the offset of the target function from the base address of the native library (`0x4668`), we can try plugging our results into a generic native hook pattern.

### Generic Native Hook Pattern

The following is my common pattern to hook a native function using `Frida`:

#### Frida Snippet

```Javascript
(function () {
    const moduleName = "libnativeexample.so"; // <-- Native library name
    const functionName = "should_say_hello"; // <-- Name of the function
    const functionOffset = "0x4668"; // <-- The offset we just found

    const module = Process.findModuleByName(moduleName);
    if (module != null) {
        const targetAddr = module.base.add(functionOffset);
        console.log(`[+] Hook installed for [${functionName}]`);
        Interceptor.attach(targetAddr, {
            onEnter: function (args) { 
                console.log(`-> entered [${functionName}] at [${targetAddr}]`);

                // Your code goes here...
            }, 
            onLeave: function (retval) {
                console.log(`<- leaving [${functionName}] at [${this.context.pc}]`);

                console.log(`[+] Original return value was: ${retval.toInt32()}`);
                retval.replace(1);
                console.log(`[+] Modified to: ${retval.toInt32()}`);
            }
        });   
    } else {
        console.log("[!] Failed to find module");
    }
})(); 
```

#### Code Breakdown

Let's understand how this `Frida` script works.

1. We plug in our findings from the previous section into the constant variables:

```Javascript
...
const moduleName = "libnativeexample.so"; // <-- Native library name
const functionName = "should_say_hello"; // <-- Name of the function
const functionOffset = "0x4668"; // <-- The offset we just found
...
```

2. We ask `Frida` to find the native library from the application's memory map at runtime. If successful, we perform the calculation of adding the target function's offset to the module base address to get the function's absolute memory address. 

```Javascript
...
const module = Process.findModuleByName(moduleName);
    if (module != null) {
        const targetAddr = module.base.add(functionOffset);
...
```

3. Finally, we use `Frida's` `Interceptor` to install the hook at the calculated `targetAddr` and provide the callbacks for when we enter the function and for when we leave it. Inside the `onLeave` callback, we modify the return value from `0` to `1`, which corresponds to changing the return value from `false` to `true`.

```Javascript
...
Interceptor.attach(targetAddr, {
    onEnter: function (args) { 
        console.log(`-> entered [${functionName}] at [${targetAddr}]`);

        // Your code goes here...
    }, 
    onLeave: function (retval) {
        console.log(`<- leaving [${functionName}] at [${this.context.pc}]`);
        
        console.log(`[+] Original return value was: ${retval.toInt32()}`);
        retval.replace(1);
        console.log(`[+] Modified to: ${retval.toInt32()}`);
    }
});
...
```

### Testing the Hook

All that is left to do is run the script on the target application and see if it works.

1. In the same folder where you put the `apk`, create a file named `script.js` and paste our `Frida` code snippet in it:

<a href="/images/screenshots/hooking_native_functions/first_hook_1.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/first_hook_1.png" width="100%">
</a>

2. Run `frida-server` if you haven't already:

<img src="/images/screenshots/hooking_native_functions/first_hook_2.png" width="100%">

3. Get the package name of our demo application using `Frida`:

```Shell
frida-ps -Uai
```

<a href="/images/screenshots/hooking_native_functions/first_hook_3.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/first_hook_3.png" width="100%">
</a>

4. Finally, make `Frida` spawn the target application and immediately execute our `script.js` file:

```shell
frida -U -f com.noobexon.nativeexample -l script.js
```
As you may have already guessed from this section's title, our script fails. But why?

<img src="/images/screenshots/hooking_native_functions/first_hook_4.png" width="100%">

## What went wrong?

### The problem

To understand what went wrong, we just need to read the output from `Frida`:

<img src="/images/screenshots/hooking_native_functions/first_hook_5.png" width="100%">

In fact, if we take a closer look at our code in `script.js`, we can see that this message is actually from our logs:

<img src="/images/screenshots/hooking_native_functions/first_hook_6.png" width="100%">

And if we zoom further in on the issue, we can see that it stemmed from this part of the code:

```javascript
...
const module = Process.findModuleByName(moduleName);
if (module != null) {
    ...
} else {
    console.log("[!] Failed to find module");
}
...
```

This means `Frida` couldn't find the native library in memory and therefore returned `null`... But why? After all, if we run the application without `Frida`, then we do see that everything works perfectly, so the native library must have been loaded **at some point**, right?

And that is exactly the problem here. `Frida` is fast. Very fast. The moment we run the command in our shell, it immediately tries to install the hook as fast as possible, even before the application had a chance to load the native library to memory, **and `Frida` can't find the library in memory if it hasn't been loaded yet.**

So what can we do?...

Well, our problem is in fact a **timing issue**. `Frida` attempts to hook the native library before it has been loaded to memory, and therefore fails. If we can somehow tell `Frida` **when** it needs to install the hook, then our problems will be solved. And that is exactly what we are going to do!

### The solution

How can we let Frida know for sure that the library is loaded and that it's now ok to install the hook? There are many solutions out there, but most of them are based on busy-waiting -- constantly polling the memory map every few milliseconds until the target library appears. This works only sometimes, and it's wasteful and fragile.

We will take a different approach. To know **when** our target library has been loaded to memory, we can hook the function that actually loads libraries to memory. This function is called `android_dlopen_ext` and it takes the path of the library to be loaded as its first argument.

Modify the contents of `script.js` with the following and run the same command:

```javascript
(function () {
    const dlopenAddr = Module.getGlobalExportByName("android_dlopen_ext");
    if (dlopenAddr != null) {
        Interceptor.attach(dlopenAddr, {
            onEnter: function (args) {
                const loadPath = args[0].readCString();
                console.log(`[+] loading ${loadPath}!`);
            },
        });
    }
})();
```

This snippet will print every library being dynamically loaded:

<img src="/images/screenshots/hooking_native_functions/explain_1.png" width="100%">

Therefore, we can use the following hook as a mechanism to let `Frida` know **when** to install the hook:

```javascript
(function () {
    const targetLibName = "libnativeexample.so"; 

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

Modify the contents of `script.js` to include the above snippet, and run. The script tells us exactly when the target library has been loaded.

<img src="/images/screenshots/hooking_native_functions/explain_2.png" width="100%">


## Second Attempt (Success)

Now that we figured out the solution to the timing problem, all we have to do is take our original script and compose it over our timing mechanism:

<a href="/images/screenshots/hooking_native_functions/second_hook_1.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/second_hook_1.png" width="100%">
</a>

### Frida Snippet

```javascript
(function () {
    const targetLibName = "libnativeexample.so"; 

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

                    // ------- ORIGINAL HOOK ---------------
                    (function () {
                        const moduleName = "libnativeexample.so"; 
                        const functionName = "should_say_hello"; 
                        const functionOffset = "0x4668"; 

                        const module = Process.findModuleByName(moduleName);
                        if (module != null) {
                            const targetAddr = module.base.add(functionOffset);
                            console.log(`[+] Hook installed for [${functionName}]`);
                            Interceptor.attach(targetAddr, {
                                onEnter: function (args) { 
                                    console.log(`-> entered [${functionName}] at [${targetAddr}]`);

                                    // Your code goes here...
                                }, 
                                onLeave: function (retval) {
                                    console.log(`<- leaving [${functionName}] at [${this.context.pc}]`);

                                    console.log(`[+] Original return value was: ${retval.toInt32()}`);
                                    retval.replace(1);
                                    console.log(`[+] Modified to: ${retval.toInt32()}`);
                                }
                            });   
                        } else {
                            console.log("[!] Failed to find module");
                        }
                    })(); 

                    // ----------------------------------

                    this.isTarget = false;
                }
            }
        });
    }
})();
```

### Testing the Hook

As expected, everything works now and the app no longer crashes!

<a href="/images/screenshots/hooking_native_functions/second_hook_2.png" target="_blank">
  <img src="/images/screenshots/hooking_native_functions/second_hook_2.png" width="100%">
</a>

## Conclusion & Template

### Templates You Should Save

#### `android_dlopen_ext` Hook

```javascript
(function () {
    // ** Change to target library name... **
    const targetLibName = "[MODULE_NAME_PLACEHOLDER]"; 

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

#### Generic Native Hook

```javascript
(function () {
    
    const moduleName = "[MODULE_NAME_PLACEHOLDER]"; // <-- Change to target library name... 
    const functionRelativeAddress = "[FUNCTION_RELATIVE_ADDRESS_PLACEHOLDER]"; // <-- Change to function offset from the library base address... 
    const functionName = "[FUNCTION_NAME_PLACEHOLDER]"; // <-- Change to target function name... 

    const module = Process.findModuleByName(moduleName);
    if (module != null) {
        const targetAddr = module.base.add(functionRelativeAddress);
        console.log(`[+] Hook installed for [${functionName}]`);
        Interceptor.attach(targetAddr, {
            onEnter: function (args) { 
                console.log(`-> entered [${functionName}] at [${targetAddr}]`);

                // Your code goes here...

            }, 
            onLeave: function (retval) {
                console.log(`<- leaving [${functionName}] at [${this.context.pc}]`);
                
                // Your code goes here...

            }
        });   
    } else {
        console.log("[!] Failed to find module");
    }
})(); 
```