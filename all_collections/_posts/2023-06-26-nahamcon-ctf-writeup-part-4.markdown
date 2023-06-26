---
layout: post
title:  "Nahamcon CTF 2023 mobile challenges Writeup, part 4: Where's Waldo ?"
date:   2023-06-26
categories: writeup
---
This post is the fourth and last of a series of four where I'll be describing how I solved 4 out of the 5 mobile challenges of the [Nahamcon CTF 2023][nahamcon-ctf-website].  
If you haven't read the first and second posts, you can read them [here][first-post] and [here][second-post].  

This challenge relied on an API located at http://challenge.nahamcon.com:30001, but the API is not working anymore since the CTF is over.  Hence, I won't be able to show lots of screenshots in this post, but I will still try to explain everything as clearly as possible. 

# Tools used

I've already described the tools I used in [the first post][first-post].

# Challenge

The Nahamcon CTF website is down, so I can't take a screenshot of the challenge rules, but it is basically the same than before: we are only given an APK file with no further explanations.
Once again, I downloaded it and install it.  

```console
Natsuko@kali:~/wheres_waldo/$ adb install wheres_waldo.apk
Performing Streamed Install
Success
```

The application displays a map of the world, with a pin of my location on the map.  
Every few seconds, a toast appears, saying *Waldo is ~4000 miles away!*.  
There's not much things to interact with inside the app, so it seemed clear to me that the goal was to look for waldo around the world. But how is Waldo's position known by the app ? Let's investigate by looking at the application's code.

# Decompiling the application

```console
Natsuko@kali:~/wheres_waldo/$ jadx-gui wheres_waldo.apk
```

By looking at the code, it quickly appeared that the application was making calls to a web API. This is the part of the code that executes the request and uses the response:

```java
Response execute = new OkHttpClient().newCall(new Request.Builder().url("http://challenge.nahamcon.com:30001/location?lat=" + lastLocation.getLatitude() + "&long=" + lastLocation.getLongitude()).build()).execute();
if (execute.isSuccessful()) {
    ResponseBody body = execute.body();
    JsonObject jsonObject = (JsonObject) new Gson().fromJson(body != null ? body.string() : null, (Class<Object>) JsonObject.class);
    String asString = jsonObject.get("message").getAsString();
    int asInt = jsonObject.get("off_by").getAsInt();
    int pow = (int) Math.pow(10.0d, String.valueOf(asInt).length() - 1);
    long round = Math.round(asInt / pow) * pow;
    if (asInt == 0) {
        Toast.makeText(mainActivity.getApplicationContext(), asString, 1).show();
        
        // [...]

    }
    Toast.makeText(mainActivity.getApplicationContext(), asString + " Waldo is ~" + round + " miles away", 1).show();
    return;
```

Here we can see the web API's URL (http://challenge.nahamcon.com:30001/), as well as what does the response look like: there's a *message* field, and an *off_by* one. If the *off_by* one is 0, then the toast will display the content of *message*, most likely the flag. Else, the toast will prompt the distance separating us from Waldo.  

The first thing I tried is to modify the smali bytecode (just like in [the previous post][third-post]) to make the condition inside the `if` statement be the opposite. That did work, and the toast displayed the content of the *message* field of the API response, but this field did not contain the flag, since I did not find Waldo.  

This made me realize that we have no choice but to find Waldo for real.  
I actually knew that would be possible with my setup, because Android Studio's phone emulator has a feature called *Extended controls* where you can do a bunch of things, including virtually changing the phone's location.  

![extended controls feature of android studio's phone emulator](/assets/images/nahamcon_ctf/wheres_waldo/emulator_extended_controls.png)

I had no idea where to look for Waldo, the only clue that I had being that he was approximately 4000 miles away from France (my location). I tried to change my location to New York City, USA (the first generic location that came to my mind), and the toast said that waldo is approximately 20 miles away!!!  
I couldn't believe how lucky I was. I was afraid that finding Waldo could take a certain amount of time, and I got super lucky on my first guess.  

After a few minutes of playing with the phone's location around New York City, I found Waldo, and the app displayed the flag! (Since the app is not working anymore, I can't show you where Waldo was exactly)

# Bonus: an elegant way to solve the challenge ?

Before deciding to look for Waldo within the emulator, I tried to solve the challenge in a more elegant way.  
I tried making requests to the web API outside of the app, and it responded. From there, I thought that I could find Waldo algebraically, using [trilateration][trilateration-wikipedia].  
Unfortunately, I am not very good at math, and I didn't manage to find a way to solve the challenge this way. I still think it is possible though! While looking for utilities to compute distance from earth coordinates, I found [Geopy][geopy-website], a Python library to make various calculations from earth coordinates. Doing those calculations by hand would have been hard, since the earth is not flat, and not a sphere either, but an ellipsoid, making computations unaccurate if you assume the Earth is a sphere.  

That's it for this fourth challenge!  
I ended up solving 4 out of 5 mobile challenges, and I regret not being able to solve the 5th one. This really motivated me to play more CTFs!

![my nahamcon CTF certificate](/assets/images/nahamcon_ctf/nahamcon_certificate.png)

(Yes, my team was named *So lonely*. I was participating alone and had no idea how to name it. A friend suggested me to name it that way.)  
I was glad to see that solving only the mobile challenges (and not even all of them) got me a nice ranking. I wish I had knew about this CTF earlier, so that I could have tried other categories, or join a team.

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
[trilateration-wikipedia]: https://www.wikiwand.com/en/Trilateration
[first-post]: {{ site.url }}/nahamcon-ctf-writeup-part-1
[second-post]: {{ site.url }}/nahamcon-ctf-writeup-part-2
[third-post]: {{ site.url }}/nahamcon-ctf-writeup-part-3
