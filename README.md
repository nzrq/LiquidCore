The LiquidCore Project
----------------------

LiquidCore provides an environment for developers to create native mobile micro apps in Javascript that can in turn be embedded into _other_ apps.  Think: native `<iframe>` for mobile apps.  A LiquidCore micro app is simply a [Node.js] module that can be served from the cloud, and therefore, like in a webpage, it can be modified server-side and instantly updated on all mobile devices.

LiquidCore also provides a convenient way for Android developers to execute raw JavaScript inside of their apps, as iOS developers can already do natively with JavaScriptCore.

LiquidCore is currently only available on Android, but will be ported to iOS.  **EDIT**: LiquidCore now works on iOS, but is still experimental.  You can build the framework using Carthage.  From the `LiquidCoreiOS` directory, run `carthage build --no-skip-current`.  Of course you can also build inside of XCode.  Documentation is forthcoming and will be official in version 0.5.0.

Version
-------
[0.4.6](https://github.com/LiquidPlayer/LiquidCore/releases/tag/0.4.6) - Get it through [JitPack](https://jitpack.io/#LiquidPlayer/LiquidCore/0.4.6)

[![Release](https://jitpack.io/v/LiquidPlayer/LiquidCore.svg)](https://jitpack.io/#LiquidPlayer/LiquidCore)

Javadocs
--------
[Version 0.4.6](https://liquidplayer.github.io/LiquidCoreAndroid/0.4.6/index.html)

# Table of Contents

1. [Use Cases](#use-cases)
2. [Building the LiquidCore Android library](#building-the-liquidcore-android-library)
3. [License](#license)

# Use Cases

This section covers the two major intended use cases of LiquidCore, complete with _Hello, World!_ step-by-step examples.

1. [The Micro Service](#the-micro-service)
2. [The Micro App](#the-micro-app)

You can also use LiquidCore as a raw Native Javascript Engine (i.e. as a replacement for [`AndroidJSCore`](https://github.com/ericwlange/AndroidJSCore)).  That topic is discussed [here](https://github.com/LiquidPlayer/LiquidCore/wiki/LiquidCore-as-a-Native-Javascript-Engine).

## The Micro Service

A *micro app* is built on a *micro service*.  A micro service is nothing more than an independent Node.js instance whose startup code is referenced by a URI.  For example:

```java
MicroService service = new MicroService(androidContext,
    new URI("http://my.server.com/path/to/code.js"));
service.start();
```

The service URI can either refer to a server URL or a local Android resource (e.g. `android.resource://com.example.myapp/raw/some_js_file`, where `some_js_file.js` resides in `res/raw/some_js_file.js` -- note that the `.js` is omitted from the URI when using an Android resource).  LiquidCore is designed to primarily use remote URLs, as dynamic updates are an important value proposition, but local resources are supported for both debugging and/or backup (e.g. as a factory preset if the network is not available).

A micro service can communicate with the host app once the Node.js environment is set up.  This can be determined by adding a `ServiceStartListener` in the `MicroService` constructor:

```java
MicroService service = new MicroService(
    androidContext,
    new URI("http://my.server.com/path/to/code.js"),
    new MicroService.StartServiceListener() {
        @Override
        public void onStart(MicroService service, Synchronizer synchronizer) {
            // .. The environment is live, but the startup JS code (from the URI)
            // has not been executed yet.
        }
    }
);
service.start();
```

A micro service communicates with the host through a simple [`EventEmitter`](https://nodejs.org/api/events.html#events_class_eventemitter) interface, eponymously called `LiquidCore`.  For example, in your JavaScript startup code (code.js in this example):

```javascript
LiquidCore.emit('my_event', {foo: "hello, world", bar: 5, l337 : ['a', 'b'] })
```

On the Java side, the host app can listen for events:

```java
// ... in the StartServiceListener.onStart() method:

service.on("my_event", new MicroService.EventListener() {
    @Override
    public void onEvent(MicroService service, String event, JSONObject payload) {
        try {
            android.util.Log.i("Event:" + event, payload.getString("foo"));
            // logs: I/Event:my_event: hello, world
        } catch (JSONException e) {
            e.printStackTrace();
        }
    }
});
```

Similarly, the micro service can listen for events from the host:

```java
JSONObject payload = new JSONObject();
payload.put("hallo", "die Weld");
service.emit("host_event", payload);
```

Then, in Javascript:

```javascript
LiquidCore.on('host_event', function(msg) {
   console.log('Hallo, ' + msg.hallo)
})
```

LiquidCore creates a convenient virtual file system so that instances of micro services do not unintentionally or maliciously interfere with each other or the rest of the Android filesystem.  The file system is described in detail [here](https://github.com/LiquidPlayer/LiquidCore/wiki/LiquidCore-File-System).

## The Micro App

There are many uses for micro services.  They are really useful for taking advantage of all the work that has been done by the Node community.  But we want to be able to create our own native applications that do not require much, if any, interaction from the host.  To achieve this, we will introduce one more term: **Surface**.  A surface is a UI canvas for micro services.

There are two surfaces so far:

1. [**`ConsoleSurface`**](https://github.com/LiquidPlayer/ConsoleSurface).  A `ConsoleSurface` is simply a Node.js terminal console that displays anything written to `console.log()` and `console.error()`.  It also allows injection of Javascript commands, just like a standard Node console.  See the [ConsoleSurface](https://github.com/LiquidPlayer/ConsoleSurface) project for a tutorial on how to use it.
2. [**`ReactNativeSurface`**](https://github.com/LiquidPlayer/react-native).  You can drive native UI elements using the [React Native](https://facebook.github.io/react-native/) framework from within your micro app.  This is a fork of the React Native project that has modifications to allow it to run on LiquidCore.  It is very experimental at this point and I haven't written any documentation yet.  But it does work.

There are other surfaces under consideration, including:

* **`WebSurface`** - a `WebView` front-end where a micro service can write to the DOM
* **`CardSurface`** - a limited feature set suitable for driving card-like UI elements in a list
* **`OpenGLSurface`** - an [OpenGL](https://www.opengl.org/) canvas

Eventually, we would like to have virtual/augmented reality surfaces, as well as non-graphical canvases such as chat and voice query interfaces.

### "Hallo, die Weld!" Micro Service Tutorial

#### Prerequisites

* A recent version of [Node.js] -- 8.9.3 or newer
* [Android Studio]

(You can find all the code below in a complete example project [here](https://github.com/LiquidPlayer/Examples/tree/master/HelloWorld) if you get stuck).

To use a micro service, you need two things: the micro service code, and a host app.

We will start by creating a very simple micro service, which does nothing more than send a welcome message to the host.  This will be served from a machine on our network.  Start by creating a working directory somewhere.

```
$ mkdir ~/helloworld
$ cd ~/helloworld
```

Then, install the LiquidCore server (aptly named _LiquidServer_) from `npm`:
```
$ npm install -g liquidserver
```

Now let's create a micro service.  Create a file in the `~/helloworld` directory called `service.js` and fill it with the following contents:

```javascript
/* Hello, World! Micro Service */

// A micro service will exit when it has nothing left to do.  So to
// avoid a premature exit, let's set an indefinite timer.  When we
// exit() later, the timer will get invalidated.
setInterval(function() {}, 1000)

// Listen for a request from the host for the 'ping' event
LiquidCore.on( 'ping', function() {
    // When we get the ping from the host, respond with "Hallo, die Weld!"
    // and then exit.
    LiquidCore.emit( 'pong', { message: 'Hallo, die Weld!' } )
    process.exit(0)
})

// Ok, we are all set up.  Let the host know we are ready to talk
LiquidCore.emit( 'ready' )
```

Next, let's set up a manifest file.  Don't worry too much about this right
now.  Basically, the manifest allows us to serve different versions based
on the capabilities/permissions given by the host.  But for our simple example,
we will serve the same file to any requestor.  Create a file in the same
directory named `service.manifest`:

```javascript
{
   "configs": [
       {
           "file": "service.js"
       }
   ]
}
```

This tells LiquidServer that when a request comes in for `service.js`, it should
serve our `service.js` file.  This may seem dumb, but there are other useful attributes
which can be set.  For our purposes, though, they are not yet needed.

You can now run your server.  Choose some port (say, 8080), or LiquidServer will create
one for you:

```
$ liquidserver 8080
```

You should now see the message, `Listening on port 8080`.  Congratulations, you just
created a micro service.  You can test that it is working correctly by navigating to
`http://localhost:8080/service.js` in your browser.  You should see the contents of
`service.js` that you just created with some wrapper code around it.  The wrapper is simply
to allow multiple Node.js modules to be packed into a single file.  If you were to
`require()` some other module, that module and its dependencies would get wrapped into this
single file.

You can leave that running or restart it later.  Now we need to create a host app.

1. In Android Studio, create a new project by selecting `File -> New Project ...`
2. Fill out the basics and press `Next` (Application Name: `HelloWorld`, Company Domain: `liquidplayer.org`, Package name: `org.liquidplayer.examples.helloworld`)
3. Fill in the Target Devices information.  The defaults are fine.  Click `Next`
4. Select `Empty Activity` and then `Next`
5. The default options are ok.  Click `Finish`

You now have a basic app that does very little.  Go ahead and run it in your emulator.
If you hadn't figured out why we are creating a "Hallo, die Weld!" app, you probably can
see why by now.  You already get "Hello, World!" from Android Studio.  We're going to
make it speak German with our micro service.

Next, open the `res/layout/activity_main.xml` file.  We need to make a couple of modifications.  Replace the contents with the following:

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="org.liquidplayer.examples.helloworld.MainActivity">

    <TextView
        android:id="@+id/text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:layout_centerVertical="true"
        android:text="Sprechen Sie Deutsch!"
        />
</RelativeLayout>
```

All we've changed is that we have given our `TextView` a name: `text`, and added a button.  Go ahead and run it again.  You should now see a big button in the middle.

Now it is time to connect LiquidCore.  First, you must add the library.  Go to your **root-level `build.grade`**
file and add the `jitpack` dependency:

```
...

allprojects {
    repositories {
        jcenter()
        maven { url 'https://jitpack.io' }
    }
}

...
```

Then, add the LiquidCore library to your **app's `build.gradle`**:

```
dependencies {
    ...
    implementation 'com.github.LiquidPlayer:LiquidCore:0.4.6'
}

```
Note that `implementation` should be replaced with `compile` if you are using an older buildtools version.

Go ahead and sync to make sure the library downloads and links properly.  Run the app again to ensure that all is good.

Now, let's connect our button to the micro service.  Edit `MainActivity.java` in our app, and replace the contents with the following:

```java
package org.liquidplayer.examples.helloworld;

import android.os.Handler;
import android.os.Looper;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import org.json.JSONException;
import org.json.JSONObject;
import org.liquidplayer.service.MicroService;
import org.liquidplayer.service.MicroService.ServiceStartListener;
import org.liquidplayer.service.MicroService.EventListener;

import java.net.URI;
import java.net.URISyntaxException;

public class MainActivity extends AppCompatActivity {

    // IMPORTANT: Replace this with YOUR server's address or name
    private final String serverAddr = "192.168.1.152:8080";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        final TextView textView = (TextView) findViewById(R.id.text);
        final Button button = (Button) findViewById(R.id.button);

        // Our 'ready' listener will wait for a ready event from the micro service.  Once
        // the micro service is ready, we'll ping it by emitting a "ping" event to the
        // service.
        final EventListener readyListener = new EventListener() {
            @Override
            public void onEvent(MicroService service, String event, JSONObject payload) {
                service.emit("ping");
            }
        };

        // Our micro service will respond to us with a "pong" event.  Embedded in that
        // event is our message.  We'll update the textView with the message from the
        // micro service.
        final EventListener pongListener = new EventListener() {
            @Override
            public void onEvent(MicroService service, String event, final JSONObject payload) {
                // NOTE: This event is typically called inside of the micro service's thread, not
                // the main UI thread.  To update the UI, run this on the main thread.
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            textView.setText(payload.getString("message"));
                        } catch (JSONException e) {
                            e.printStackTrace();
                        }
                    }
                });
            }
        };

        // Our start listener will set up our event listeners once the micro service Node.js
        // environment is set up
        final ServiceStartListener startListener = new ServiceStartListener() {
            @Override
            public void onStart(MicroService service, Synchronizer synchronizer) {
                service.addEventListener("ready", readyListener);
                service.addEventListener("pong", pongListener);
            }
        };

        // When our button is clicked, we will launch a new instance of our micro service.
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    URI uri = new URI("http://"+serverAddr+"/service.js");
                    MicroService service = new MicroService(MainActivity.this, uri, startListener);
                    service.start();
                } catch (URISyntaxException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

Make sure you change `serverAddr` to your server's address.  Now, restart the app and then click the button.  The "Hello World" message should change to German.  You have successfully connected a micro service to a host app!

To demonstrate the instant update feature, leave the app and server running.  Now, edit `service.js` on your server machine to respond with a different message and then save:

```javascript
...
    LiquidCore.emit( 'pong', { message: 'Das ist super!' } )
...
```

Go back to the app and press the button again.  Your message should update.

That's it.  That's all there is to it.  Of course, this is an overly simplified example.  We can do other useful things, like utilizing existing Node.js modules.  To try this, create a new file named `bn.js`, and fill it with the following:

```javascript
var BigNumber = require('bignumber.js')

setInterval(function() {}, 1000)

LiquidCore.on( 'ping', function() {
    var x = new BigNumber(1011, 2)          // "11"
    var y = new BigNumber('zz.9', 36)       // "1295.25"
    var z = x.plus(y)                       // "1306.25"
    LiquidCore.emit( 'pong', { message: '' + x + ' + ' + y + ' = ' + z } )
    process.exit(0)
})

LiquidCore.emit( 'ready' )
```

We will now be using the [BigNumber] module.  Be sure to install it first:

```
% npm install bignumber.js
```

You will also need the manifest file, `bn.manifest` in the same directory:

```javascript
{
   "configs": [
       {
           "file": "bn.js"
       }
   ]
}
```

Now navigate to `http://localhost:8080/bn.js` in your browser and you should now see that the `bignumber.js` module has also been wrapped.

In your Hallo, die Weld app, change the following line:
```java
URI uri = new URI("http://"+serverAddr+"/service.js");
```
to:
```java
URI uri = new URI("http://"+serverAddr+"/bn.js");
```

Then restart the app.  You should now see an equation that utilized the module when you click the button.

Ok, one last little trick.  Let's modify the `bn.manifest` file to include a transform.  Replace with this:

```javascript
{
   "configs": [
       {
           "file": "bn.js",
           "transforms": [ "uglifyify" ]
       }
   ]
}
```

You will need to clear the server cache, so simply delete the `.lib` directory:

```
% rm -rf ~/helloworld/.lib
```

And then restart the server.  Now when you navigate to `http://localhost:8080/bn.js`, you will see that the code has been minified in order to save space.  The manifest file can do a bunch of things, but we'll save that for later as it is still in its infancy.


Building the LiquidCore Android library
---------------------------------------

If you are interested in building the library directly and possibly contributing, you must
do the following:

    % git clone https://github.com/liquidplayer/LiquidCore.git
    % cd LiquidCore/LiquidCoreAndroid
    % echo ndk.dir=$ANDROID_NDK > local.properties
    % echo sdk.dir=$ANDROID_SDK >> local.properties
    % ./gradlew assembleRelease

Your library now sits in `LiquidCoreAndroid/build/outputs/aar/LiquidCore-release.aar`.  To use it, simply
add the following to your app's `build.gradle`:

    repositories {
        flatDir {
            dirs '/path/to/lib'
        }
    }

    dependencies {
        implementation(name:'LiquidCore-release', ext:'aar') // 'compile' on older buildtools versions
    }
    
##### Note

The Node.js library (`libnode.so`) is pre-compiled and included in binary form in
`deps/node-8.9.3/prebuilt`.  All of the modifications required to produce the library are included in `deps/node-8.9.3`.  To build each library (if you so choose), see the instructions [here](https://github.com/LiquidPlayer/LiquidCore/wiki/How-to-build-libnode.so).

License
-------

 Copyright (c) 2014-2018 Eric Lange. All rights reserved.

 Redistribution and use in source and binary forms, with or without
 modification, are permitted provided that the following conditions are met:

 - Redistributions of source code must retain the above copyright notice, this
 list of conditions and the following disclaimer.

 - Redistributions in binary form must reproduce the above copyright notice,
 this list of conditions and the following disclaimer in the documentation
 and/or other materials provided with the distribution.

 THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
 DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
 FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
 CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
 OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[Node.js]:https://nodejs.org/
[Android Studio]:https://developer.android.com/studio/index.html
[BigNumber]:https://github.com/MikeMcl/bignumber.js/
