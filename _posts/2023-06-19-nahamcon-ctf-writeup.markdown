---
layout: post
title:  "Nahamcon CTF 2023 mobile challenges Writeup"
date:   2023-06-19
categories: writeup
---
A few months ago, I learned at school the basics of reverse-engineering Android applications.  
Last week, I suddenly felt the urge to practice what I learned in a CTF, for no particular reason other than the sudden realization that I never really experienced Android applications reverse-engineering; I then remembered that [John Hammond][john-hammond-website] was advertising a CTF he was organizing on Twitter, so I went to check [his Twitter][john-hammond-twitter] to find the CTF link.  
The CTF was [NahamCon][nahamcon-website] 2023's CTF; [NahamCon][nahamcon-website] is an online and free virtual offensive security conference. It's a shame that I didn't really pay attention to John Hammond's tweet advertising it, I would have liked to watch some of the conferences.  
Anyway, I headed to the [CTF][nahamcon-ctf-website] link, hoping for Android challenges; there were 5 of them, 2 of them rated as easy, and 3 medium, no hard.  
This CTF was supposed to be played in team, even though you can of course participate as a team of one player. I registered a team under the name *So lonely*, because I was playing on my own.

![Nahacom mobile challenges](/assets/nahamcon_mobile_challenges.png)

There was less than 24 hours remaining before the end of the CTF, so I had to be as quick as possible.  

# Setup

There are a few tools that we're gonna need in order to solve these.  

About the operating system, I'm using Kali Linux, but I'm pretty sure that all of the tools I describe here exist on Windows too. You'll just need to adapt a few things.

First, we need an emulator. The one I'm gonna use here is [Android Studio][[android-studio-website]]'s embedded phone emulator, but there are other emulators you can use. The reasom I'm using this one is because it's the only one I've ever used, and I still had it installed.  
However, regardless of the emulator you're using, I'd suggest you to setup a device that is using the Android API in a version that is less or equal than 29. The reason is that I ran into a problem trying to install APKs that I had patched that I wasn't able to solve. The problem is the same as the one described in [this StackOverflow post][stackoverflow-post-version-30], and applying the fix proposed in the best answer didn't work. I had to delete my emulator that was using API 31 to create one using API 29.  
To install the latest version of Android Studio, go to [the Android Studio website][android-studio-website] and click the Download button.  

![Android Studio website](/assets/android_studio_website.png)

Next, we're gonna need some decompiler. The one I'm using is [jadx][jadx-github]; it's a Java decompiler specially made to produce a Java code from an APK file.  
To install jadx, follow the instruction in the [Github README][jadx-github]. You can simply grab the latest release, install it with a package manager or compile it on your machine.  

![jadx website](/assets/jadx-website.png)

We'll also need [Apktool][apktool-website], which is a tool to extract an APK and re-building it.  
Follow the instructions detailed on the [Download page][apktool-download-page] in order to install it.  

![Apktool website](/assets/apktool_website.png)

Last but not least, we need [ADB][adb-website] (Android Debug Bridge); it is the tool that allow us to interract with an Android phone.  
As described in ADB's website, in [the download page][adb-download-page], ADB is shipped within the Android SDK platform tools. If you have installed Android Studio, just like me, you very likely have ADB somewhere in your Android Studio installation. Else, follow the instructions on [the Download page][adb-download-page].  

![ADB website](/assets/adb_website.png)

If you have Android Studio installed, you're looking for a binary named `adb` (on Linux, on Windows it is probably called `adb.exe`).

```console
Natsuko@kali:~$ find / -name "adb" 2>/dev/null
/home/rouxmarin/.local/lib/python3.10/site-packages/pwnlib/adb
/home/rouxmarin/.local/lib/python3.10/site-packages/pwnlib/protocols/adb
/home/rouxmarin/Documents/tools/jadx/jadx-gui/src/main/resources/icons/adb
/home/rouxmarin/Documents/tools/jadx/jadx-gui/build/resources/main/icons/adb
/home/rouxmarin/Android/Sdk/sources/android-33/com/android/server/adb
/home/rouxmarin/Android/Sdk/platform-tools/adb
/usr/share/doc/metasploit-framework/modules/exploit/android/adb
/usr/share/metasploit-framework/modules/exploits/android/adb
/usr/share/metasploit-framework/lib/rex/proto/adb
/usr/local/lib/python3.11/dist-packages/pwnlib/adb
/usr/local/lib/python3.11/dist-packages/pwnlib/protocols/adb
```

There are multiple files called `adb`, but the one we are looking for is the one located in your Android SDK installation directory. If you know where it is in the first place, the search can be faster:

```console
Natsuko@kali:~$ find /home/rouxmarin/Android -name "adb" 2>/dev/null
/home/rouxmarin/Android/Sdk/sources/android-33/com/android/server/adb
/home/rouxmarin/Android/Sdk/platform-tools/adb
```

If you're experienced with Android applications reverse-engineering, you might find it weird that I'm not listing any dynamic instrumentation toolkit, such as [Frida][frida-website]. It is because I ended up not needing it for the challenges I managed to solve.  
For those that don't know what Frida is, it is a tool that allows us to dynamically modify the behaviour of Android applications, by hooking function calls, watching the parameters, modifying them, modifying the functions' return values, etc. It is usually incredibily useful when doing Android applications reverse-engineering, but this time I managed to solve the challenges (well, not all of them, but almost) without using it. If you want to learn it, I strongly encourage you to look for writeups that are using it, or directly try it by yourself with the help of its documentation.  

# First challenge: Fortune teller

![Fortune teller challenge rules](/assets/fortune_teller.png)

*Can you guess what fortune the fortune teller is thinking of?* is the only thing we have to introduce the challenge. We're given the *fortune_teller.apk* file, and that's all.  

## Installing the application

First things first, let's install this app in our emulator. For this, we'll use ADB.  

```console
Natsuko@kali:~$ adb install fortune_teller.apk
Performing Streamed Install
Success
```

You should then see the application appear in the phone's applications list.

![Fortune teller application installed](/assets/fortune_teller_app.png)

Let's run it.

![Fortune teller application](/assets/fortune_teller_1.png)

The application is telling us to enter a guess.  
Let's try some dummy prompt.

![Fortune teller dummy prompt submitted](/assets/fortune_teller_2.png)

Clicking the Guess button displays a *Toast* saying *Hello toast!*.  
If you don't know what a toast is, it is the word that describes the little popup you can see in the bottom of the screenshot: According to Android's website: *A toast provides simple feedback about an operation in a small popup.*  
Regardless of the prompt we try, we always get the same toast. Even an empty prompt leads to the same result.  
We won't achieve anything by trying random prompt. We're gonna have to use jadx to understand how does the application work.

## Decompiling the application

Let's use `jadx-gui` to decompile the APK file:
```console
Natsuko@kali:~$ jadx-gui fortune_teller.apk
```

After a few seconds, a jadx window should open, revealing a directory tree on the left of the screen.  

![fortune_teller.apk opened in jadx](/assets/fortune_teller_jadx_1.png)

The first thing to look at in general is the `AndroidManifest.xml` file; it is located in the `Resources` directory.  

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" android:versionCode="1" android:versionName="1.0" android:compileSdkVersion="33" android:compileSdkVersionCodename="13" package="com.nahamcon2023.fortuneteller" platformBuildVersionCode="33" platformBuildVersionName="13">
    <uses-sdk android:minSdkVersion="24" android:targetSdkVersion="33"/>
    <permission android:name="com.nahamcon2023.fortuneteller.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION" android:protectionLevel="signature"/>
    <uses-permission android:name="com.nahamcon2023.fortuneteller.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"/>
    <application android:theme="@style/Theme.Fortuneteller" android:label="@string/app_name" android:icon="@mipmap/ic_launcher" android:debuggable="true" android:allowBackup="true" android:supportsRtl="true" android:extractNativeLibs="false" android:fullBackupContent="@xml/backup_rules" android:appComponentFactory="androidx.core.app.CoreComponentFactory" android:dataExtractionRules="@xml/data_extraction_rules">
        <activity android:name="com.nahamcon2023.fortuneteller.MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <provider android:name="androidx.startup.InitializationProvider" android:exported="false" android:authorities="com.nahamcon2023.fortuneteller.androidx-startup">
            <meta-data android:name="androidx.emoji2.text.EmojiCompatInitializer" android:value="androidx.startup"/>
            <meta-data android:name="androidx.lifecycle.ProcessLifecycleInitializer" android:value="androidx.startup"/>
        </provider>
    </application>
</manifest>
```

We can see an activity called `MainActivity` declared here; that is certainly the name of the activity being displayed when the application is started.  
Let's check what it looks like. Open the `Source code` folder in the directory tree, and navigate to the activity's path as declared here (`com.nahamcon2023.fortuneteller.MainActivity`).

```java
package com.nahamcon2023.fortuneteller;

import android.app.Activity;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.os.Bundle;
import android.view.View;
import android.widget.EditText;
import android.widget.ImageView;
import android.widget.Toast;
import kotlin.Metadata;
import kotlin.jvm.internal.DefaultConstructorMarker;
import kotlin.jvm.internal.Intrinsics;

/* compiled from: MainActivity.kt */
@Metadata(d1 = {"\u0000:\n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0010\u000e\n\u0002\b\u0005\n\u0002\u0018\u0002\n\u0002\b\u0005\n\u0002\u0018\u0002\n\u0002\b\u0005\n\u0002\u0010\u0002\n\u0000\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0018\u0002\n\u0002\b\u0002\u0018\u0000 \u001c2\u00020\u0001:\u0001\u001cB\u0005¢\u0006\u0002\u0010\u0002J\u000e\u0010\u0015\u001a\u00020\u00162\u0006\u0010\u0017\u001a\u00020\u0018J\u0012\u0010\u0019\u001a\u00020\u00162\b\u0010\u001a\u001a\u0004\u0018\u00010\u001bH\u0014R\u001a\u0010\u0003\u001a\u00020\u0004X\u0086.¢\u0006\u000e\n\u0000\u001a\u0004\b\u0005\u0010\u0006\"\u0004\b\u0007\u0010\bR\u001a\u0010\t\u001a\u00020\nX\u0086.¢\u0006\u000e\n\u0000\u001a\u0004\b\u000b\u0010\f\"\u0004\b\r\u0010\u000eR\u001a\u0010\u000f\u001a\u00020\u0010X\u0086.¢\u0006\u000e\n\u0000\u001a\u0004\b\u0011\u0010\u0012\"\u0004\b\u0013\u0010\u0014¨\u0006\u001d"}, d2 = {"Lcom/nahamcon2023/fortuneteller/MainActivity;", "Landroid/app/Activity;", "()V", "correctString", "", "getCorrectString", "()Ljava/lang/String;", "setCorrectString", "(Ljava/lang/String;)V", "decrypt", "Lcom/nahamcon2023/fortuneteller/Decrypt;", "getDecrypt", "()Lcom/nahamcon2023/fortuneteller/Decrypt;", "setDecrypt", "(Lcom/nahamcon2023/fortuneteller/Decrypt;)V", "guessInput", "Landroid/widget/EditText;", "getGuessInput", "()Landroid/widget/EditText;", "setGuessInput", "(Landroid/widget/EditText;)V", "guess", "", "v", "Landroid/view/View;", "onCreate", "savedInstanceState", "Landroid/os/Bundle;", "Companion", "app_debug"}, k = 1, mv = {1, 8, 0}, xi = 48)
/* loaded from: classes3.dex */
public final class MainActivity extends Activity {
    public static final Companion Companion = new Companion(null);
    public static String guessString;
    public String correctString;
    public Decrypt decrypt;
    public EditText guessInput;

    public final EditText getGuessInput() {
        EditText editText = this.guessInput;
        if (editText != null) {
            return editText;
        }
        Intrinsics.throwUninitializedPropertyAccessException("guessInput");
        return null;
    }

    public final void setGuessInput(EditText editText) {
        Intrinsics.checkNotNullParameter(editText, "<set-?>");
        this.guessInput = editText;
    }

    public final String getCorrectString() {
        String str = this.correctString;
        if (str != null) {
            return str;
        }
        Intrinsics.throwUninitializedPropertyAccessException("correctString");
        return null;
    }

    public final void setCorrectString(String str) {
        Intrinsics.checkNotNullParameter(str, "<set-?>");
        this.correctString = str;
    }

    public final Decrypt getDecrypt() {
        Decrypt decrypt = this.decrypt;
        if (decrypt != null) {
            return decrypt;
        }
        Intrinsics.throwUninitializedPropertyAccessException("decrypt");
        return null;
    }

    public final void setDecrypt(Decrypt decrypt) {
        Intrinsics.checkNotNullParameter(decrypt, "<set-?>");
        this.decrypt = decrypt;
    }

    @Override // android.app.Activity
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        View findViewById = findViewById(R.id.guess_input);
        Intrinsics.checkNotNullExpressionValue(findViewById, "findViewById(R.id.guess_input)");
        setGuessInput((EditText) findViewById);
        setDecrypt(new Decrypt());
    }

    public final void guess(View v) {
        Intrinsics.checkNotNullParameter(v, "v");
        Companion companion = Companion;
        companion.setGuessString(getGuessInput().getText().toString());
        String string = getString(R.string.correct_guess);
        Intrinsics.checkNotNullExpressionValue(string, "getString(R.string.correct_guess)");
        setCorrectString(string);
        if (Intrinsics.areEqual(companion.getGuessString(), getCorrectString())) {
            ImageView imageView = new ImageView(this);
            setContentView(imageView);
            getDecrypt().decrypt(this);
            Bitmap bitmap = BitmapFactory.decodeFile(getDecrypt().getOutputFile().getAbsolutePath());
            imageView.setImageBitmap(bitmap);
            return;
        }
        Toast toast = Toast.makeText(this, "Hello toast!", 0);
        toast.show();
    }

    /* compiled from: MainActivity.kt */
    @Metadata(d1 = {"\u0000\u0014\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u000e\n\u0002\b\u0005\b\u0086\u0003\u0018\u00002\u00020\u0001B\u0007\b\u0002¢\u0006\u0002\u0010\u0002R\u001a\u0010\u0003\u001a\u00020\u0004X\u0086.¢\u0006\u000e\n\u0000\u001a\u0004\b\u0005\u0010\u0006\"\u0004\b\u0007\u0010\b¨\u0006\t"}, d2 = {"Lcom/nahamcon2023/fortuneteller/MainActivity$Companion;", "", "()V", "guessString", "", "getGuessString", "()Ljava/lang/String;", "setGuessString", "(Ljava/lang/String;)V", "app_debug"}, k = 1, mv = {1, 8, 0}, xi = 48)
    /* loaded from: classes3.dex */
    public static final class Companion {
        public /* synthetic */ Companion(DefaultConstructorMarker defaultConstructorMarker) {
            this();
        }

        private Companion() {
        }

        public final String getGuessString() {
            String str = MainActivity.guessString;
            if (str != null) {
                return str;
            }
            Intrinsics.throwUninitializedPropertyAccessException("guessString");
            return null;
        }

        public final void setGuessString(String str) {
            Intrinsics.checkNotNullParameter(str, "<set-?>");
            MainActivity.guessString = str;
        }
    }
}
```

After reading the code, we can see that the `guess` method is particularily interesting: it seems to contain the code that gets executed when we press the *Guess* button.  

```java
public final void guess(View v) {
    Intrinsics.checkNotNullParameter(v, "v");
    Companion companion = Companion;
    companion.setGuessString(getGuessInput().getText().toString());
    String string = getString(R.string.correct_guess);
    Intrinsics.checkNotNullExpressionValue(string, "getString(R.string.correct_guess)");
    setCorrectString(string);
    if (Intrinsics.areEqual(companion.getGuessString(), getCorrectString())) {
        ImageView imageView = new ImageView(this);
        setContentView(imageView);
        getDecrypt().decrypt(this);
        Bitmap bitmap = BitmapFactory.decodeFile(getDecrypt().getOutputFile().getAbsolutePath());
        imageView.setImageBitmap(bitmap);
        return;
    }
    Toast toast = Toast.makeText(this, "Hello toast!", 0);
    toast.show();
}
```

The `if` statement is probably the one testing if the guess is correct.  
It is comparing two values: `companion.getGuessString()` and `getCorrectString()`. The first one is probably the user input, and the second one, the answer.  
The `getCorrectString` method looks like this:  
```java
public final String getCorrectString() {
    String str = this.correctString;
    if (str != null) {
        return str;
// The rest is not important
```

The answer is stored in the `correctString` property of the class we're in. This property is uninitialized, but there's also a setter for this property: `setCorrectString()`. In jadx, selecting a symbol and pressing the `x` key allows us to see its usages.  
The `setCorrectString()` method is called once, in the `guess()` method:  
```java
    String string = getString(R.string.correct_guess);
    Intrinsics.checkNotNullExpressionValue(string, "getString(R.string.correct_guess)");
    setCorrectString(string);
```

The correct string is the result of `getString(R.string.correct_guess);`.  
>R.java is the dynamically generated class, created during build process to dynamically identify all assets (from strings to android widgets to layouts), for usage in java classes in Android app. (https://stackoverflow.com/questions/6804053/understand-the-r-class-in-android)

Even without knowing where to find those resources, we can search (Ctrl+Shift+F) for *correct_guess*.

![jadx search checkboxes](/assets/jadx_search_checkboxes.png)

Be sure to check the *Resource* checkbox, since what we are looking for most likely is a resource.

![jadx search results](/assets/jadx_search_results.png)

The bottom result here is probably what we are looing for. It's a string named *correct_guess* and located in the `res/values/strings.xml` file. Its value is *you win this ctf*
Let's go back to the app and try this prompt.  

![Fortune teller trying prompt](/assets/fortune_teller_3.png)

![Fortune teller flag](/assets/fortune_teller_4.png)

We got the flag!  

# Second challenge: JNInjaspeak

![JNInjaspeak challenge rules](/assets/jninjaspeak.png)

Once again, we are only given an APK file.  
And once again, let's dwonload it and install it.

## Installing the application

```console
Natsuko@kali:~$ adb install jninjaspeak.apk
```

![JNInjaspeak application](/assets/jninjaspeak_1.png)

The application looks like this.  
Let's try a dummy prompt.  

![JNInjaspeak dummy prompt](/assets/jninjaspeak_2.png)

We get a toast of weird text.  
I played a bit with the prompt, and I found out a few interesting properties about the toast's text:
- The output is deterministic (inputting the same text twice will produce the same output)
- The output is as long as the input
- let f be the function that takes an input A and produce an output A_O; let A[i] be the ith character of the A string (and A_O[i] the ith character of A_O, etc); let A and B two strings. If A[i] = B[i], then A_O[i] = B_O[i]
- If the input contains a single character that is repeated a huge amount of times, then you'll see a pattern being repeated in the output (see screenshot below).

![JNInjaspeak single character input](/assets/jninjaspeak_3.png)

This looks like the input is XORed with a key.  
I'm gonna write down the repeated pattern in the output: `vlqwk!v%#)u$q'&u&!(tqu)tr#vqt&q'(v!m`.
Supposing that output = input ^ key ("^" here being the XOR of two strings, character by character), then it means that key = output ^ input. Let's XOR the input and the output in a Python REPL.

```python
>>> output = "vlqwk!v%#)u$q'&u&!(tqu)tr#vqt&q'(v!m"
>>> input = ['0' for _ in range(len(output))]
>>> "".join([chr(ord(output[i]) ^ ord(input[i])) for i in range(len(output))])
'F\\AG[\x11F\x15\x13\x19E\x14A\x17\x16E\x16\x11\x18DAE\x19DB\x13FAD\x16A\x17\x18F\x11]'
```

Hmmm... Looks like we are on the right way, but it's not quite good yet... We can clearly see the word *FLAG* at the start of the result, but some characters are not right.  
Let's decompile the app to try to understand what's happening!  

## Decompiling the application

```console
Natsuko@kali:~$ jadx-gui jninjaspeak.apk
```

In jadx, we can quickly figure out that the code responsible for making the output from the given input is this one:
```java
public final void translatePress(View v) {
    Intrinsics.checkNotNullParameter(v, "v");
    Companion companion = Companion;
    companion.setTranslateString(getTranslateInput().getText().toString());
    translate(companion.getTranslateString());
    Toast toast = Toast.makeText(this, translate(companion.getTranslateString()), 0);
    toast.show();
}
```

The `translate()` function seems to be the one we've gotta look at. Unfortunately, we're gonna face a problem here, because the function's signature looks like this:  

```java
private final native String translate(String str);
```

This means that the function is not written in Java, but in a compiled language, and is exported into a shared object library (`.so` file) bundled within the application. We can confirm it by looking at the `Resources/lib` directory in jadx:

![.so files in jadx](/assets/jninjaspeak_so_files.png)

There are 4 `.so` files, one for each possible combination between x86 or ARM and 32 bits or 64 bits.  
We're gonna have to extract those and analyze one with Ghidra!  
[Ghidra][ghidra-website] is a free Software Reverse Engineering suite that is able to analyze compiled binaries, and translate them into an equivalent C code.  
I haven't put Ghidra in the tools we need to Download, because it is installed by default on Kali. If you don't have it, download the latest version for free in [the GitHub release page][ghidra-download].  

## Extracting the files from the APK

Before launching Ghidra, we need to get one of these shared objects libraries.  
We're gonna use `apktool` to extract the contents of the APK file.

```console
Natsuko@kali:~$ decode jninjaspeak.apk
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.7.0 on jninjaspeak.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/rouxmarin/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Baksmaling classes4.dex...
I: Baksmaling classes3.dex...
I: Baksmaling classes2.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory

Natsuko@kali:~$ ll jninjaspeak/lib/x86_64
total 300
-rw-r--r-- 1 rouxmarin rouxmarin 304680 Jun 19 23:23 libjninjaspeak.so
```

Nice.

## Reversing the shared library

Launch Ghidra, create a new project, and then press `I` to import files, and import the shared objects library of the assembly language you like the most. I'm bad at reading any assembly anyway, so by default I'm gonna go with the `x86_64` one. Once the import is done, double-click on the file.

![Ghidra first impressions](/assets/jninjaspeak_ghidra.png)

Here's what the shared objects library looks like once opened in Ghidra.

In the symbols tree on the left, we can look for the `translate` function in the *Exports* section, because in order to be used by other pieces of code, the function has to be exported by the shared objects library.

![Symbols table](/assets/jninjaspeak_symbol_tree.png)

![translate function in symbols table](/assets/jninjaspeak_symbol_tree_translate_function.png)

Let's double click it to see what it looks like.

![translate function assembly code](/assets/jninjaspeak_translate_assembly.png)

While looking at the code, something immediately caught my attention:

![a flag string ??](/assets/jninjaspeak_flag_maybe.png)

A variable called *flag{somethingthatlooksliketheflag}* ?  
Let's double click it:

![Oh my god it's the flag](/assets/jninjaspeak_flag_confirmed.png)

The flag is here in plaintext! Submitting it in NahamCon CTF's website confirms that it is indeed the flag!  

## Beyond flag

In the end, we didn't really have to read any assembly code in details, what a relief.  
I still tried to understand what's going on in the code. As I suspected, there's a XOR happening, but there's also some weird bit shifting and function call.

![what is going on here ?](/assets/jninjaspeak_for_loop.png)

Also, there's a call to `_JNIEnv::GetStringUTFChars`, maybe that could explain why we got a weird result while naively XORing the input and the output.  

![UTF encoding or something](/assets/jninjaspeak_encoding.png)

Since I was running out of time for the CTF, I didn't take the time to analyze deeper than that and headed to the next challenge.

[john-hammond-website]: https://j-h.io/links
[john-hammond-twitter]: https://twitter.com/_JohnHammond
[nahamcon-website]: https://www.nahamcon.com/
[nahamcon-ctf-website]: https://ctf.nahamcon.com/
[stackoverflow-post-version-30]: https://stackoverflow.com/questions/69667830/targeting-r-version-30-and-above-requires-the-resources-arsc-of-installed-apk
[android-studio-website]: https://developer.android.com/studio
[jadx-github]: https://github.com/skylot/jadx
[apktool-website]: https://ibotpeaches.github.io/Apktool/
[apktool-download-page]: https://ibotpeaches.github.io/Apktool/install/
[adb-website]: https://developer.android.com/tools/adb
[adb-download-page]: https://developer.android.com/studio/releases/platform-tools
[frida-website]: https://frida.re/
[ghidra-website]: https://ghidra-sre.org/
[ghidra-download]: https://github.com/NationalSecurityAgency/ghidra/releases