---
layout: post
title:  "Nahamcon CTF 2023 mobile challenges Writeup, part 2: JNInjaspeak"
date:   2023-06-20
categories: writeup
---
This post is the second of a series of four where I'll be describing how I solved 4 out of the 5 mobile challenges of the [Nahamcon CTF 2023][nahamcon-ctf-website].  
If you haven't read the first, you can read it [here][first-post].

# Tools used

I've already described the tools I used in [the first post][first-post].

# Challenge

![JNInjaspeak challenge rules](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak.png)

Once again, we are only given an APK file.  
And once again, I downloaded it and install it.

```console
Natsuko@kali:~/jninjaspeak/$ adb install jninjaspeak.apk
Performing Streamed Install
Success
```

The application looks like this.  

![JNInjaspeak application](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_1.png)

I first tried a dummy prompt.  

![JNInjaspeak dummy prompt](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_2.png)

I got a toast containing a weird text.  

I played a bit with the prompt, and I found out a few interesting properties about the toast's text:
- The output is deterministic (inputting the same text twice will produce the same output)
- The output is as long as the input
- By submitting two different inputs, if their respective ith character are the same, then the ith characters of the respective outputs will also be the same
- If the input contains a single character that is repeated a huge amount of times, then you'll see a pattern being repeated in the output (see screenshot below).

![JNInjaspeak single character input](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_3.png)

Since the challenge was rated as easy, I thought that the input could simply be XORed with a key.  
I wrote down the repeated pattern in the output: `vlqwk!v%#)u$q'&u&!(tqu)tr#vqt&q'(v!m`.
Supposing that output = input ^ key ("^" here being the XOR of two strings, character by character), then it means that key = output ^ input. I XORed the input and the output in a Python REPL.

```python
>>> output = "vlqwk!v%#)u$q'&u&!(tqu)tr#vqt&q'(v!m"
>>> input = ['0' for _ in range(len(output))]
>>> "".join([chr(ord(output[i]) ^ ord(input[i])) for i in range(len(output))])
'F\\AG[\x11F\x15\x13\x19E\x14A\x17\x16E\x16\x11\x18DAE\x19DB\x13FAD\x16A\x17\x18F\x11]'
```

Looks like it isn't as simple as that.  
Rather than doing some more educated guessing on what could be happening behind the scenes, I opened the APK file in jadx.

# Decompiling the application

```console
Natsuko@kali:~$ jadx-gui jninjaspeak.apk
```

In jadx, I quickly figured out that the code responsible for making the output from the given input is this one:
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

The `translate()` function seems to be the one I've gotta look at. Unfortunately, I faced a problem here, because the function's signature looks like this:  

```java
private final native String translate(String str);
```

The `native` keyword means that the function is not written in Java, but in a compiled language, and is exported into a shared object library (`.so` file) bundled within the application. I can confirm this by looking at the `Resources/lib` directory in jadx:

![.so files in jadx](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_so_files.png)

There are 4 `.so` files, one for each possible combination between x86 or ARM and 32 bits or 64 bits.  
I had to extract those and analyze one of them with Ghidra!

# Extracting the files from the APK

Before launching Ghidra, I needed to get one of these shared objects libraries.  
I used `apktool` to extract the contents of the APK file.

```console
Natsuko@kali:~/jninjaspeak/$ apktool decode jninjaspeak.apk
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

Natsuko@kali:~/jninjaspeak/$ ll jninjaspeak/lib/x86_64
total 300
-rw-r--r-- 1 rouxmarin rouxmarin 304680 Jun 19 23:23 libjninjaspeak.so
```

# Reversing the shared library

The `libjninjaspeak.so` looked like this once opened in Ghidra:

![Ghidra first impressions](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_ghidra.png)

In the symbols tree on the left, I looked for the `translate` function in the *Exports* section, because in order to be used by other pieces of code, the function has to be exported by the shared objects library.

![Symbols table](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_symbol_tree.png)

![translate function in symbols table](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_symbol_tree_translate_function.png)

The function looks like this:

![translate function assembly code](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_translate_assembly.png)

While looking at the code, something immediately caught my attention:

![a flag string ??](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_flag_maybe.png)

A variable called *flag{somethingthatlooksliketheflag}* ?  
I double clicked it:

![Oh my god it's the flag](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_flag_confirmed.png)

The flag was here in plaintext! Submitting it in NahamCon CTF's website confirmed that it is indeed the flag!  

## Beyond flag

In the end, I didn't really have to read any assembly code in details, what a relief.  
I still tried to understand what's going on in the code. As I suspected, there's a XOR happening, but there's also some weird bit shifting and function call.

![what is going on here ?](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_for_loop.png)

Also, there's a call to `_JNIEnv::GetStringUTFChars`, maybe that could explain why I got a weird result while naively XORing the input and the output.  

![UTF encoding or something](/assets/images/nahamcon_ctf/jninjaspeak/jninjaspeak_encoding.png)

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
[first-post]: {{ site.url }}/nahamcon-ctf-writeup-part-1