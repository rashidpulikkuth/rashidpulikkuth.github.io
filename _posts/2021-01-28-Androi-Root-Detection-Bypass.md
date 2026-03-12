---
layout: post
title: Android Root Detection Bypass
tags: [Android Security]
comments: true
---

This is a way, how i bypassed **Root Detection** while performing retesting of an android application. I hope this will help you on understanding root detection bypass. With less talk lets jump in.

#### Lets start..

I have got a new fixed application which i tested earlier and now i have to check the bug status.

![Crepe](/assets/images/ard1.png)

When i installed application on emulator i found that this application now implemented root detection which was not present before.

![Crepe](/assets/images/ard2.png)

![Crepe](/assets/images/ard3.png)

So i decided to Bypass this root detection. First, i used **enjarify** to reverse enginner the android application, Enjarify is a tool for translating Dalvik bytecode to equivalent Java bytecode. This allows Java analysis tools to analyze Android applications.

Github Repo Link : (https://github.com/Storyyeller/enjarify)

~~~
enjarify <Your App>.apk -o <Output>.jar
~~~

![Crepe](/assets/images/ard4.png)

After this i decompiled the application using apktool.

~~~
apktool d <Application>.apk -o <outputdirectory>
~~~

![Crepe](/assets/images/ard5.png)

Next step is to open the jar file on JD-GUI, which will show source code of the application and i have to find how the application is doing the root detection. 

From AndroidManifest.xml i found that the launcher activity is “com.bestdocapp.bdconnectplus.ui.splash.SplashActivity” , so i checked the source code of that class.

Here is the source code

![Crepe](/assets/images/ard6.png)

So a boolean value is returned from 'isDeviceRooted()' function at RootUtils class.

if that boolean value is 'true' then the device is rooted and the application will not work on that device.

We can bypass this check by simply editing the condition on 'if' statement  on smali code .

But i have to check what is inside 'isDeviceRooted()' function.

* function : isDeviceRooted()

![Crepe](/assets/images/ard8.png)

So this applcation uses 3 functions to check the device is rooted or not.

![Crepe](/assets/images/ard7.png)

So i can bypass this root detection in diffrent ways, but what i did is modified 'isDeviceRooted()' function and it will return "false" everytime.

This way we can bypass this security check.

So lets do it.

~~~
nano smali/com/bestdocapp/bdconnectplus/utils/RootUtils.smali
~~~

* Before editing the file

![Crepe](/assets/images/ard9.png)

* After editing

![Crepe](/assets/images/ard10.png)

So i changed the function now this function will return false (0x0) everytime.

Lets rebuild the application , sign and check the bypass is working or not.

~~~
apktool b Application -o Application-new.apk
~~~

![Crepe](/assets/images/ard12.png)

Now we can install the application, but When i tried to install the apk i got the below error.

![Crepe](/assets/images/ard13.png)

So i edited AndroidManifest.xml file and changed android:extractNativeLibs="false" to android:extractNativeLibs="true"

then again tried to install the app.

![Crepe](/assets/images/ard14.png)

Now opened the app on emulator and yes our bypass worked well.

![Crepe](/assets/images/ard15.png)

Bye..
