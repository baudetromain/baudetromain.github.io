---
layout: post
title:  "Nahamcon CTF 2023 mobile challenges Writeup, part 3: Red light green light"
date:   2023-06-25
categories: writeup
---
This post is the third of a series of four where I'll be describing how I solved 4 out of the 5 mobile challenges of the [Nahamcon CTF 2023][nahamcon-ctf-website].  
If you haven't read the first and second posts, you can read them [here][first-post] and [here][second-post].

# Tools used

I've already described the tools I used in [the first post][first-post].

# Challenge

The Nahamcon CTF website is down, so I can't take a screenshot of the challenge rules, but it is basically the same than before: we are only given an APK file with no further explanations.
Once again, I downloaded it and install it.  

```console
Natsuko@kali:~/red_light_green_light/$ adb install red_light_green_light.apk
Performing Streamed Install
Success
```

The application looks like this.  

![JNInjaspeak application](/assets/images/nahamcon_ctf/red_light_green_light/app.png)

When I clicked on the button, a toast saying *You cannot move right now, the light is not green!* appeared.

![JNInjaspeak dummy prompt](/assets/images/nahamcon_ctf/red_light_green_light/button_click.png)

Outside of the button, there seemed to be no way of interacting with the application, and the light was of course not turning green. And I was of course not waiting for it to do so.  

In order to find a way to make the app do something, I first opened it in jadx to take a look at the app's logic.

# Decompiling the application

```console
Natsuko@kali:~/red_light_green_light/$ jadx-gui red_light_green_light.apk
```

The `MainActivity` class had a property of type `boolean` called `red` that was set to `true` by default and seemd to be never changed.

```java
private boolean red = true;
```

The class also contained a method called `checkLight()`, which caught my attention. It is most likely the method called when the button is clicked.  
Here are the first lines of the method:

```java
public final void checkLight(View v) {
        Intrinsics.checkNotNullParameter(v, "v");
        Log.w("KEY", getKey());
        if (!this.red) {
            this.decrypt = new Decrypt();
            MainActivity mainActivity = this;
            ImageView imageView = new ImageView(mainActivity);
            setContentView(imageView);
            Decrypt decrypt = this.decrypt;
            Decrypt decrypt2 = null;
            if (decrypt == null) {
                Intrinsics.throwUninitializedPropertyAccessException("decrypt");
                decrypt = null;
            }
            decrypt.decrypt(mainActivity, getKey());
            Decrypt decrypt3 = this.decrypt;
            if (decrypt3 == null) {
                Intrinsics.throwUninitializedPropertyAccessException("decrypt");
            } else {
                decrypt2 = decrypt3;
            }
            imageView.setImageBitmap(BitmapFactory.decodeFile(decrypt2.getOutputFile().getAbsolutePath()));
            return;
        }
        Toast.makeText(this, "You cannot move right now, the light is not green!", 0).show();
    }
```

since `this.red` is true and never changed, `!this.red` is forever false. In case `!this.red` would be true, we see that the application would open an `ImageView`, most likely revealing the flag. We can guess that because the code seems to be decrypting an image before showing it, and to encrypt the flag would make sense, so that we cannot easily statically find it.  
However, the `getKey()` method, which is called in the decryption process, could allow us to easily decrypt the flag image. Sadly, the function is not implemented in Java, but in a shared object bundled in the app, just like in the previous challenge.  

```java
private final native String getKey();

// later...

static {
        System.loadLibrary("redlightgreenlight");
}
```

We could reverse the shared object file where this function is defined, but I wanted to try something else first.  
The `apktool` tool allow us to unpack an APK file into smali bytecode, and rebuild it later. That mean we can edit the smali bytecode of the application to change its behaviour. For example, we could make the `if (!this.red)` condition to `if (this.red)`, so that when we click the button, the flag gets displayed if the traffic light is NOT green.  

# Modifying the app

```console
Natsuko@kali:~/red_light_green_light/$ apktool decode red_light_green_light.apk
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.7.0 on red_light_green_light.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/rouxmarin/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory
```

Once the application has been unpacked, I browsed in the created directory and opened the `smali/com/nahamcon2023/redlightgreenlight/MainActivity.smali` file.  
The smali bytecode contains annotations telling us which smali bytecode instructions correspond to which java source code line, so it was not hard to find the condition I was looking for.

```smali
.line 22
iget-boolean p1, p0, Lcom/nahamcon2023/redlightgreenlight/MainActivity;->red:Z
if-nez p1, :cond_2
```

The value of  this.red  is stored into the `p1` register, and then the `if-nez p1 :cond_2` instruction mean that if the value inside `p1` is not equal to 0 (*nez*), then the execution will jump to the `cond_2` label.  
We can make this check doing the opposite by replacing the `if-nez` by `if-eqz` (if equals zero).  

```smali
.line 22
iget-boolean p1, p0, Lcom/nahamcon2023/redlightgreenlight/MainActivity;->red:Z
# if-nez p1, :cond_2
if-eqz p1, :cond_2
```

Once done, I repacked the APK file with `apktool`:

```console
Natsuko@kali:~/red_light_green_light/$ apktool build red_light_green_light -o red_light_green_light_modified.apk
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.7.0
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Copying libs... (/lib)
I: Copying libs... (/kotlin)
I: Copying libs... (/META-INF/services)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk into: red_light_green_light_modified.apk
```

Before being able to install our modified APK, we need to sign it.  
I always forget the command and look it up on Google. [Here][stack-overflow-sign-apk]'s a StackOverflow post showing the commands to use.  

Creating a keystore:  

```console
Natsuko@kali:~/red_light_green_light/$ keytool -genkey -v -keystore my-release-key.keystore -alias my-keystore -keyalg RSA -keysize 2048 -validity 10000
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Enter keystore password:
Re-enter new password:
What is your first and last name?
  [Unknown]:
What is the name of your organizational unit?
  [Unknown]:
What is the name of your organization?
  [Unknown]:
What is the name of your City or Locality?
  [Unknown]:
What is the name of your State or Province?
  [Unknown]:
What is the two-letter country code for this unit?
  [Unknown]:
Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
  [no]:  y

Generating 2,048 bit RSA key pair and self-signed certificate (SHA256withRSA) with a validity of 10,000 days
        for: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
[Storing my-release-key.keystore]
```

Signing the APK file:  

```console
Natsuko@kali:~/red_light_green_light/$ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore red_light_green_light_modified.apk my-keystore
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
Enter Passphrase for keystore:
   adding: META-INF/MANIFEST.MF
   adding: META-INF/MY-KEYST.SF
   adding: META-INF/MY-KEYST.RSA
  signing: META-INF/services/kotlinx.coroutines.internal.MainDispatcherFactory
  signing: META-INF/services/kotlinx.coroutines.CoroutineExceptionHandler
  signing: kotlin/internal/internal.kotlin_builtins
  
  [...]

  signing: assets/dexopt/baseline.prof
  signing: DebugProbesKt.bin
  signing: kotlin-tooling-metadata.json

>>> Signer
    X.509, CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
    Signature algorithm: SHA256withRSA, 2048-bit key
    [trusted certificate]

jar signed.

Warning:
The signer's certificate is self-signed.
The SHA1 algorithm specified for the -digestalg option is considered a security risk and is disabled.
The SHA1withRSA algorithm specified for the -sigalg option is considered a security risk and is disabled.
```

While trying to install the app, I got an error:

```console
Natsuko@kali:~/red_light_green_light/$ adb install red_light_green_light_modified.apk
Performing Streamed Install
adb: failed to install red_light_green_light_modified.apk: Failure [INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2]
```

[This stackoverflow post][stack-overflow-align] helped me resolve it.  

```console
Natsuko@kali:~/red_light_green_light/$ zipalign -p 4 red_light_green_light_modified.apk red_light_green_light_modified_aligned.apk
```

After that, I was able to install the app.

```console
Natsuko@kali:~/red_light_green_light/$ adb install -t red_light_green_light_modified_aligned.apk
Performing Streamed Install
Success
```

I ran the app, clicked the button, and the flag appeared!  

![flag](/assets/images/nahamcon_ctf/red_light_green_light/flag.png)  

That's it for this third challenge!  
You can find my writeup of the fourth one [here][fourth-post].

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
[stack-overflow-sign-apk]: https://stackoverflow.com/questions/10930331/how-to-sign-an-already-compiled-apk
[stack-overflow-align]: https://stackoverflow.com/questions/55173004/targeting-sdk-android-q-results-in-failed-to-finalize-session-install-failed-i
[first-post]: {{ site.url }}/nahamcon-ctf-writeup-part-1
[second-post]: {{ site.url }}/nahamcon-ctf-writeup-part-2
[fourth-post]: {{ site.url }}/nahamcon-ctf-writeup-part-4
