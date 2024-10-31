---
name: iTag
tools: [Android, Bluetooth, GPS, Gradle Artefacts, Java]
image: /assets/images/itag-screen.jpg
description: iTag is an Android application that connects to a Bluetooth gadget for tracking the user’s belongings.
---

# Android iTag

iTag is Bluetooth gadget that helps users track their belongings. The iTag Android application has 2 modes:
 - **Find**: The user can ring the iTag gadget to find their belongings.
 - **Lost**: The user can watch if the gadget is in range or not. If the gadget is out of range, the user will receive
  a notification and the location where the connection was lost is shown on the map.

Besides the main features, the application has scan mode to find all the iTag gadgets in the area, and a settings
to configure up to 4 iTag gadgets with the name and color of each one.

  [GitHub](https://github.com/s4ysolutions/itag)

  [Play Store](https://play.google.com/store/apps/details?id=s4y.itag)

## Technical Details

The BLE layer is implemented using the Android BLE API, but for consistency it is wrapped in iOS CoreBluetooth-like API.
The wrapper is implemented in its own module and is used in the main application.

For the sake of the performance and battery life, the application is written in Java and do not use heavy libraries like
RxJava, instead for this application were developed lightweight reactive frameworks porting [iOS pub/sub library](https://github.com/gokselkoksal/Rasat)

   - [rasat-java](https://github.com/s4ysolutions/rasat-java)
   - [rasat-android](https://github.com/s4ysolutions/rasat-android)

[Sample usage](https://github.com/s4ysolutions/itag/blob/9a68d6093e9ae1a992a6b29873eeb3f6afdb31af/app/src/main/java/s4y/itag/ITagsFragment.java#L480):

```java
   disposableBag.add(connection.observableRSSI().subscribe(rssi -> updateRSSI(id, rssi)));
```

The BLE signal strength is very noisy and raw reading is not very useful. The application uses a 1-dimensional [Kalman
filter](https://github.com/s4ysolutions/itag/blob/master/itagble/src/main/java/s4y/itag/ble/RSSIFilter.java) to report the smoothed signal strength.
