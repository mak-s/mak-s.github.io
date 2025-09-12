---
layout: post
title:  "Instrument App Launch"
date:   2023-10-07
categories: iOS AppLaunch Instrumentation Performance
---

This post describes the metrics that you can use to measure your app's launch performance, how to categorize these launch metrics and handle prewarming scenarios.

## What metrics can be used to measure App launch?

For measuring App Launch performance, following metrics can be collected.

**1. AppStart**

This is the duration an application takes to display first screen.

This metric is measured between two fixed points in the app launch sequence and is measured in the same way across different apps.

**2. AppStartInteractive**

This is the duration an application takes to enter a state where the user can start interacting with the application. 

This metric's end time is custom for each app. Every app has a different business logic during app launch, therefore the App specifies the end time for this metric

### Calulating App launch duration

`AppStat` metric measures how fast the first view appears on screen.
- **AppStart:** T<sub>AppStart</sub>= T2 - T1


`AppStartIntearctive` metric measures how fast your application becomes available to the user for use.
- **AppStart Interactive:** T<sub>AppStartInteractive</sub> = T3 - T1

where :
- T1 = time when the App Process started
- T2 = time when the Window became visible
- T3 = time when the App became ready for a User interaction.

![App Launch Metric](/assets/images/app_launch_metric.png)

## Categorize your App launch metrics

Use a category to track the startup type. The startup type can be categorized as:

**Cold Launch**
- When an app launches for the first time after an app install/update or system reboot.
In this case, the system takes additional time to set up resources required by the app and observed duration is more.

**Warm Launch**
- When an app has already been launched once. Warm startup duration tends to be less than Cold Startup duration.

## Track the reason for your App's launch

Identify the reason for your App's launch and associate it with the app launch metric.
Use the launch reason to understand how your App's launch performance is impacted under different launch scenarios. 


## Special Consideration: Prewarming

From the Apple documentation: [about the app launch sequence](https://developer.apple.com/documentation/uikit/app_and_environment/responding_to_the_launch_of_your_app/about_the_app_launch_sequence)

*In iOS 15 and later, the system may, depending on device conditions, pre-warm an app — launch non-running application processes to reduce the amount of time the user waits before the app is usable. Prewarming executes an app’s launch sequence up until, but not including, when main() calls UIApplicationMain(_:_:_:_:).*

*Prewarming an app results in an indeterminate amount of time between when the prewarming phase completes and when the user, or system, fully launches the app. Because of this, use MetricKit to accurately measure user-driven launch and resume times instead of manually signposting various points of the launch sequence.*

**Prewarming affect on AppLaunch metrics**

Due to prewarming, measuring app launch from the process launch time gives elevated value compared to normal launch by a user, since the process is already launched, much ahead of the actual application launch by the user.


**Handling Prewarming cases**

- Check if the app is launched dur to prewarming and include a flag in metric metadata to identify prewarmed launches

```
[[NSProcessInfo processInfo].environment[@"ActivePrewarm"] isEqual:@"1"]
```

- For prewarmed app launches, depending on your app's requirement, either drop the prewarmed launch metrics or measure AppStart from a different start point (e.g. runtime init, main() etc).

- Add a threshold for AppLaunch metric and drop the metrics beyond the threshold. This can ensure that high latency metrics are not captured.

- Gather AppLaunch Metric from Apple framework for device reports: [MetricKit](https://developer.apple.com/documentation/metrickit)