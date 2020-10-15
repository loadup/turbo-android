# Turbolinks Android

Turbolinks Android is a native adapter for any [Turbolinks 6](https://github.com/turbolinks/turbolinks#readme) enabled web app. It's built entirely using standard Android tools and conventions.

This library has been in use and tested in the wild since June 2020 in the all-new [HEY Android](https://play.google.com/store/apps/details?id=com.basecamp.hey&hl=en_US) app.

Our goal for this library is that it not only be straightforward to get started, but that it aligns with and uses 2020 (and beyond) Android conventions as much as possible.

## Contents

1. [Installation](#installation)
1. [Getting Started](#getting)
1. [Advanced Configuration](#advanced-configuration)
1. [Running the Demo App](#running-the-demo-app)
1. [Contributing](#contributing)

## Installation
Add the dependency from jCenter to your module's (not top-level) `build.gradle` file.	

```groovy
repositories {
    jcenter()
}

dependencies {
    implementation 'com.basecamp:turbolinks:2.0.0'
}
```

## Getting Started

### Prerequisites

1. Android API 24+ is required as the `minSdkVersion` in your build.gradle.
1. In order for a  [WebView](https://developer.android.com/reference/android/webkit/WebView.html) to access the Internet and load web pages, your application must have the `INTERNET` permission. Make sure you have `<uses-permission android:name="android.permission.INTERNET" />` in your Android manifest.

### Create a configuration
A configuration file is what determines a variety of rules Turbolinks will follow to navigate and present views. It consists of both global settings and path specific rules that determine how and when a particular fragment will be navigated to.

At very minimum, you will need a `src/main/assets/json/configuration.json` file that Turbolinks can read, with at least a single path configuration.

Example:

```json
{
  "rules": [
    {
      "patterns": [
        ".*"
      ],
      "properties": {
        "context": "default",
        "uri": "turbolinks://fragment/web",
        "pull_to_refresh_enabled": true
      }
    }
  ]
}
```

The `pattern` attribute defines a standard Regex pattern that will be compared against any `location` provided.

There are a few `properties` that Turbolinks supports out of the box.

* `uri`: Required. Must map to a Fragment that has implemented the `TurbolinksNavGraphDestination` annotation with a matching `uri` value.
* `context`: Optional. Possible values are `default` or `modal`. Describes the presentation context under which the view should be displayed. Allows Turbolinks to determine what the navigation behavior should be. Unless you are specifically showing a modal-style view (often forms are modal), `default` is usually sufficient. Defaults to `default`.
* `presentation`: Optional. Possible values are `default`, `push`, `pop`, `replace`, `replace_root`, `clear_all`, `refresh`, or `none`. Allows Turbolinks to determine (along with the `context`) what the navigation behavior should be. In most cases `default` should be sufficient, but you may find cases where your app needs specific beahvior. Defaults to `default`.
* `fallback_uri`: Optional. Provides a fallback in case a destination cannot be found that maps to the `uri`. Can be useful in cases when pointing to a new `uri` that may not be deployed yet.
* `pull_to_refresh_enabled`: Optional. Whether or not pull to refresh should be enabled for a given path. Defaults to `false`.

Note that the configuration file is processed in order and cascade downward, similar to CSS. The top most declaration should establish the default behavior for all path patterns, and then each subsequent rule can override for specific behavior. That is, the last rule in the file will override anything above it.

### Create a layout
You'll need a basic layout for your fragment to inflate. Take a look at `fragment_web.xml` and feel free to copy that as your baseline.

The key element here is to include a reference to `turbolinks_default`, which automatically adds the necessary view hierarchy that Turbolinks expects for attaching a WebView, progress view, and error view.

```xml
<include
    layout="@layout/turbolinks_default"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    app:layout_constraintBottom_toBottomOf="parent"
    app:layout_constraintTop_toBottomOf="@+id/app_bar" />
```

### Create a custom destination interface
This step isn't completely necessary, but can provide some flexibility in navigation rules that your app will likely need. The standard `TurbolinksNavDestination` offers most of what you need, but extending this interface and creating your own can offer some benefits.

A good starting point is to look at (and copy) `NavDestination`, where we show some common additional logic you may to help determine how to handle certain navigational scenarios.

### Create a fragment
You'll need at least one fragment to represent the main body of your view. A `WebFragment` would be a good option since you're here and are probably interested in WebViews. :)

This fragment should implement your custom destination interface as mentioned above, or, if you haven't created one, simply implement `TurbolinksNavDestination`.

Your fragment must extend one of the base fragments provided by Turbolinks. In this case, as a web fragment, you should extend `TurbolinksWebFragment`.

A good starting point is to look at (and copy) `WebFragment`. It outlines the very basics of what every fragment will need to do — annotating it appropriately as a nav graph destination, inflating a view, setting up progress and error views, and setting up a toolbar.

### Create an activity
Turbolinks assumes a single-activity per Turbolinks session architecture. So generally you'll have one activity and many fragments that swap into that activity's nav host.

Turbolinks activities are fairly straightforward to begin with, and simply need to extend `TurbolinksActivity` in order to provide a `TurbolinksDelegate` that can interact with Turbolinks.

Take a look at `MainActivity` and feel free to copy that as a starting point.

### Create a session nav host fragment
A session nav host fragment is ultimately an extension of the Android Navigation component's `NavHostFragment`, and as such is primarily responsible for providing "an area in your layout for self-contained navigation to occurr." 

In the Turbolinks version, this nav host fragment is bound to a given Turbolinks session, and you will need to implement just a few things.

* The name of the Turbolinks session
* The start location
* A list of activities where Turbolinks will look for the destinations
* A list of fragments that Turbolinks will register as destinations
* A path configuration to provide navigation configuration
* Any additional setup upon creating the session

Refer to `MainSessionnavHostFragment` for an example.

🎉**Congratulations, you're using Turbolinks on Android!** 👏

## Additional Configuration

### Disabling transitional screenshots
When the Turbolinks `WebView` is detached from an activity, a bitmap screenshot of the view is automatically created and later displayed when the activity is resumed. When a cached version of the page is restored, the screenshot is removed. This makes transitions appear very fast while the cached item is loading.

If you don't want this behavior, it can be disabled with a global setting in the configuration file:

```json
{
  "settings": {
    "screenshots_enabled": false
  },
  "rules": [
    ...
  ]
}
```

### Multiple instances of TurbolinksSession

There is a 1-1-1 relationship between an activity, its nav host fragment, and its Turbolinks session. The nav host fragment automatically creates a `TurbolinksSession` for you.

You may encounter situations where a truly single activity app may not be feasible — that is, you may need an activity, for example, for logged out state vs. logged in state. Or perhaps it's safer to send inactive users to an entirely different activity to guard access controls.

In such cases you, simply need to create a new activity + nav host fragment, which will in turn create its own Turbolinks session. You will need to be sure to register all these activities as `TurbolinksSessionNavHostFragment().registeredActivities` so that you can navigate between them.

### Custom WebView WebSettings

By default the library sets a few WebSettings on the shared WebView for a given session. Some are required while others serve as reasonable defaults for modern web applications. They are:

- `setJavaScriptEnabled(true)` (required)
- `setDomStorageEnabled(true)`

If however these are not to your liking, you can always override them from your `TurbolinksWebFragment`. The `WebView` is always available to you via `TurbolinksWebFragmentDelegate.webView()`, and you can update the `WebSettings` to your liking.

```kotlin
delegate.webView.settings.domStorageEnabled = false
```

**If you do update the WebView settings, be sure not to override `setJavaScriptEnabled(false)`. Doing so would break Turbolinks, which relies heavily on JavaScript.**

### Custom JavascriptInterfaces

If you have custom JavaScript on your pages that you want to access as JavascriptInterfaces, you can add them like so:

```java
delegate.webView.addJavascriptInterface(this, "MyCustomJavascriptInterface");
```

The Java object being passed in can be anything, as long as it has at least one method annotated with `@android.webkit.JavascriptInterface`. Names of interfaces must be unique, or they will be overwritten in the library's map.

**Do not use the reserved name `TurbolinksSession` for your JavaScriptInterface, as that is used by the library.**

## Running the Demo App

A demo app is bundled with the library, and works in two parts:

1. A tiny Turbolinks-enabled Sinatra web app that you can run locally.
1. A tiny Android app that connects to the Sinatra web app.

### Prerequisites

- [Ruby installed](https://www.ruby-lang.org/en/downloads/)
- [RubyGems installed](https://rubygems.org/pages/download)
- [Bundler installed](http://bundler.io/)
- [Sinatra installed](http://www.sinatrarb.com/)
- [Rack installed](https://github.com/rack/rack#installing-with-rubygems)
- Your IP address

### Start the Demo Sinatra Web App

- From the command line, change directories to `<project-root>/demoapp/server`.
- Run `bundle`
- Run `rackup --host 0.0.0.0`

You should see a message saying what port the demo web app is running on. It usually looks like:

`Listening on 0.0.0.0:9292`

### Start the Demo Android App

- Ensure your web app and Android device are on the same network.
- Go to [`MainActivity`](/demoapp/src/main/java/com/basecamp/turbolinks/demo/MainActivity.java).
- Find the `BASE_URL` String at the top of the class. Change the IP and port of that string to match your IP and the port the Sinatra web app is running on.
- Build/run the app to your device.

## Contributing

Turbolinks Android is open-source software, freely distributable under the terms of an [MIT-style license](LICENSE). The [source code is hosted on GitHub](https://github.com/turbolinks/turbolinks-android).

We welcome contributions in the form of bug reports, pull requests, or thoughtful discussions in the [GitHub issue tracker](https://github.com/turbolinks/turbolinks-android/issues). Please see the [Code of Conduct](CONDUCT.md) for our pledge to contributors.

Turbolinks Android was created by [Dan Kim](https://twitter.com/dankim) and [Jay Ohms](https://twitter.com/jayohms), with guidance and help from [Sam Stephenson](https://twitter.com/sstephenson) and [Jeffrey Hardy](https://twitter.com/packagethief). Development is sponsored by [Basecamp](https://basecamp.com/).

### Building from Source

#### From Android Studio:

- Open the [project's Gradle file](build.gradle).
- In the menu, choose Build --> Rebuild project.

#### From command line:

- Change directories to the project's root directory.
- Run `./gradlew clean assemble -p turbolinks`.

The .aar's will be built at `<project-root>/turbolinks/build/outputs/aar`.

### Running Tests

**From command line:**

- Change directories to the project's root directory.
- Run `./gradlew clean testRelease -p turbolinks`
