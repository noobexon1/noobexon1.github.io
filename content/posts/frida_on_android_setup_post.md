+++
title = "Frida on Android - Initial Setup"
date = 2026-04-10
draft = true
tags = ["Frida", "Android", "frida-server", "Frida on Android", ".zshrc"]
#image = "/images/post-cover.png"   # optional cover image
+++

# Introduction

In this first part of the **Frida on Android** series, we learn how to perform a basic setup to use `Frida` on our Android device. We start by understanding what is `Frida` and what are the components required used by `Frida` to properly run on Android. Then, we proceed to perform the actual setup of said components. Finally, we make our workflow faster by embedding custom shell commands in our `.zshrc` (or `.bashrc`) file.

## What is Frida?

[Frida](https://frida.re/) is a dynamic instrumentation toolkit that lets you hook into and modify apps logic at runtime. It is especially popular in Android security research because of its straightsorward Java method hooking. Of course, native code hooking is also quite simple. 

It allows researchers to bypass SSL pinning, root detection, and authentication logic, intercept network traffic, and inspect app behavior live, all making it the go-to tool for mobile penetration testing and reverse engineering.

## Frida for Android

There is more then one way to use `Frida` on Android, but in this guide I'll show the most common setup which uses the following:

1. A host machine running `Frida CLI` (I'll be using a [kali-linux](https://www.kali.org/) VM as the host).
2. A rooted Android device running a `Frida-Server`.

![frida_architecture_diagram](/images/diagrams/frida_architecture.svg)

As seen in the diagram, `Frida-Server` is running on the Android device itself. Its main responsibilities are to install hooks into the target app, as well as communicate and transfer data to the `Frida CLI` which runs on the host machine.

# Setting Up

## Host Machine

We need to install `Frida CLI` on our host machine.

1. Open a terminal on you host machine and install `Frida CLI` using `pip`:

```Shell
pip install frida-tools
```

2. After installation, check your installation version by running:

```Shell
frida --version
```

![frida_version](/images/screenshots/frida_version.png)

3. Write this version number down. We will need it in the next step.

## Android Device

We need to download `Frida-Server` from the GitHub repository of `Frida` and transfer it to the rooted Android device.

1. Download `Frida-Server` for your android device. The `Frida-Server` version **must** match the version of your `Frida CLI` on your host. You can get it in the [release page](). For example, my version is `17.5.2` so I need:

![frida_server_1](/images/screenshots/frida_server_1.png)
![frida_server_2](/images/screenshots/frida_server_2.png)

2. Extract the binary from the archive and change its name to `frida-server`:

<img src="/images/screenshots/frida_server_3.png" width="100%">

3. Push the file into `/data/local/tmp/` on your Android device with:

```Shell
adb push frida-server /data/local/tmp
```

![frida_server_4](/images/screenshots/frida_server_4.png)

4. Get into `adb shell`, navigate to `/data/local/tmp` and give execution permissions to the `frida-server` file:

![frida_server_5](/images/screenshots/frida_server_5.png)

5. I recommend reading through the `--help` output just to get yourself familiar with it:

<img src="/images/screenshots/frida_server_6.png" width="100%">


6. When you are ready, start `Fride-Server` with:

```Shell
./frida-server
```


## Testing the Setup

To test that everything works do the following:

1. Make sure `Frida-Server` is running (If your terminal is still up from the last stage, then it is).
2. Make sure your Android device is connected to your host machine via USB cable (Again, this should already be the case).
3. On you Android device, open any app, and then from your host's terminal run the following to attach `Frida` to the frontmost app:

```Shell
frida -U -F
```

<a href="/images/screenshots/testing_1.png" target="_blank">
  <img src="/images/screenshots/testing_1.png" width="100%">
</a>