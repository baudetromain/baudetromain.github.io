---
layout: post
title:  "Nahamcon CTF 2023 mobile challenges Writeup, part 1: Fortune teller"
date:   2023-06-19
categories: writeup
---
This post is the first of a series of four where I'll be describing how I solved 4 out of the 5 mobile challenges of the [Nahamcon CTF 2023][nahamcon-ctf-website].  
[NahamCon][nahamcon-website] is an online and free virtual offensive security conference. Sadly, I did not attend to any of its conferences, because I heard about it too late. I started playing the CTF when there was a bit less than 24 hours remaining.

# Tools used

This is the tools i've been using for those challenges:  

- [Android studio][android-studio-website], as an Android device emulator. There are other Android emulators out there, but I had this one still installed on my machine.
- [jadx][jadx-github]: Jadx is a Java decompiler especially made to decompile Android applications. We'll need it to understand how the applications work internally.
- [apktool][apktool-website]: Apktool is a tool to decompress or build APK files from Smali bytecode (Smali is the name of the bytecode that Android's JVM is running). 
- [adb][adb-website], which stands for Android Debug Bridge, is the tool that allow us to interract with the Android device we're using.  
- [ghidra][ghidra-website]: Ghidra is a free Software Reverse Engineering suite that is able to analyze compiled binaries, and translate them into an equivalent C code.

# Challenge

We are given a few instructions and an APK file.

![Fortune teller challenge rules](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller.png)

I downloaded the APK, and installed it in my emulator.

```console
Natsuko@kali:~/fortune_teller/$ adb install fortune_teller.apk
Performing Streamed Install
Success
```

The application looks like this.

![Fortune teller application](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_1.png)

There seems to be nothing else than the text area and the submit button.  
I tried some dummy input, and I got a toast saying *Hello toast!*

![Fortune teller dummy prompt submitted](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_2.png)

The output text displayed in the toast is always the same, no matter what my input is.  
I opened the APK file in jadx to start looking at its source code.  

```console
Natsuko@kali:~/fortune_teller/$ jadx-gui fortune_teller.apk
```

![fortune_teller.apk opened in jadx](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_jadx_1.png)

The file `Resources\AndroidManifest.xml` looked like this:

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

An activity called `MainActivity` is declared. Let's go to see how it is implemented.

![MainActivity viewed in jadx](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_jadx_main_activity.png)

After reading the code, the code that caught my attention was the `guess()` method: it seems to contain the code that gets executed when we press the *Guess* button.  

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

Even without knowing where to find those resources, I can search in jadx (Ctrl+Shift+F) for *correct_guess*.

![jadx search checkboxes](/assets/images/nahamcon_ctf/fortune_teller/jadx_search_checkboxes.png)

Be sure to check the *Resource* checkbox, since what we are looking for most likely is a resource.

![jadx search results](/assets/images/nahamcon_ctf/fortune_teller/jadx_search_results.png)

The bottom result here is probably what I am looing for. It's a string named *correct_guess* and located in the `res/values/strings.xml` file. Its value is *you win this ctf*
I went back to the app and tried this prompt.  

![Fortune teller trying prompt](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_3.png)

I clicked the *GUESS* button, and...

![Fortune teller flag](/assets/images/nahamcon_ctf/fortune_teller/fortune_teller_4.png)

I got the flag!  

That's it for this first challenge!  
You can find my writeup of the second one [here][second-post].

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
[second-post]: {{ site.url }}/nahamcon-ctf-writeup-part-2