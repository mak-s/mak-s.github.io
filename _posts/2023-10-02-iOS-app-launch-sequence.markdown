---
layout: post
title:  "Sequence of iOS App Launch"
date:   2023-10-02
categories: iOS AppLaunch
---

This post outlines the various stages an application goes through during launch, providing insight into the startup process and helping measure launch time effectively.

## Sequence

Following are the steps involved in an iOS application launch:

# 1. Process Start
This is the time when an Application process starts.

An app can start in many different ways. For example:
- a User clicks on the App icon
- OS launches the App in response to some event (e.g. open url, notification etc)
- OS pre-warms the App in background
  - in this case, a user may click on the app icon much later than the process start time.

# 2. Runtime loading
Frameworks used by the Apps are loaded.
Static initializers, load and constructor methdos are called.

# 3. main()
OS calls the `main()` function which is the entry point for an application.

# 4. UIApplicationMain
Inside the `main()` function, `UIApplicationMain(_: _: _: _: )` is called to create an instance of the application.

# 5. Application Launch events
- Application's UIApplicationDelegate methods are invoked.
{% highlight swift %}
application(_:willFinishLaunchingWithOptions:)
application(_:didFinishLaunchingWithOptions:)
{% endhighlight %}
- Application launch notification is sent:
{% highlight swift %}
UIApplication.didFinishLaunchingNotification
{% endhighlight %}

# 6. Window becomes visible
When the App launches, an initial Window becomes visible.
Application sends out a notification: 
{% highlight swift %}
UIWindow.didBecomeVisible
{% endhighlight %}
 
## Stages

The App Launch can be broadly divided into 2 stages:

# 1. Pre-main:
This is the duration it takes before the main() function is executed.

# 2. Post-main:
This is the duration it takes from the main() function execution start till the first screen becomes visible / interactive.

![AppLaunchSequence](/assets/images/app_launch_sequence.png)