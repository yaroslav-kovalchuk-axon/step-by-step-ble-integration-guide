# Step-by-Step BLE Integration Guide for Expo Apps Using react-native-ble-plx

This guide walks you through integrating Bluetooth Low Energy (BLE) into your Expo application using the `react-native-ble-plx` library. It covers setup, permissions, scanning, connecting, reading, writing, and best practices, with code examples and Expo-specific notes.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Installation & Setup](#installation--setup)
4. [Configuring Expo for BLE](#configuring-expo-for-ble)
5. [Handling Permissions](#handling-permissions)
6. [Initializing the BLE Manager](#initializing-the-ble-manager)
7. [Scanning for Devices](#scanning-for-devices)
8. [Connecting to Devices](#connecting-to-devices)
9. [Reading & Writing Characteristics](#reading--writing-characteristics)
10. [Monitoring State & Disconnections](#monitoring-state--disconnections)
11. [Troubleshooting & Best Practices](#troubleshooting--best-practices)
12. [References](#references)

---

## Introduction

Bluetooth Low Energy (BLE) enables wireless communication with nearby devices. With Expo and `react-native-ble-plx`, you can build cross-platform apps that interact with BLE peripherals, such as sensors and wearables.

## Prerequisites

- Node.js & npm/yarn
- Expo CLI (`npm install -g expo-cli`)
- Basic knowledge of React Native & Expo
- A BLE peripheral device for testing

## Installation & Setup

1. **Create a new Expo app:**
   ```sh
   expo init MyBLEApp
   cd MyBLEApp
   ```
2. **Install react-native-ble-plx:**
   ```sh
   npx expo install react-native-ble-plx
   ```
3. **Add the BLE plugin to your `app.json` or `app.config.js`:**
   ```json
   {
     "expo": {
       "plugins": ["react-native-ble-plx"]
     }
   }
   ```
   For advanced configuration (background mode, custom permissions):
   ```json
   {
     "expo": {
       "plugins": [
         [
           "react-native-ble-plx",
           {
             "isBackgroundEnabled": true,
             "modes": ["peripheral", "central"],
             "bluetoothAlwaysPermission": "Allow $(PRODUCT_NAME) to connect to bluetooth devices"
           }
         ]
       ]
     }
   }
   ```

## Configuring Expo for BLE

- **Android:**
  - BLE requires minimum SDK version 23. Expo handles this, but if you eject, set `minSdkVersion` in `android/build.gradle`:
    ```groovy
    defaultConfig {
      minSdkVersion 23
    }
    ```
  - BLE permissions are auto-injected by the plugin, but if you customize, ensure these are present:
    ```xml
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
    ```
- **iOS:**
  - No extra setup is needed for managed Expo apps. For bare workflow, ensure Bluetooth permissions are set in `Info.plist`.

## Handling Permissions

BLE requires runtime permissions, especially on Android. Here’s a cross-platform permission handler:

```js
import { Platform, PermissionsAndroid } from "react-native";

async function requestBluetoothPermission() {
  if (Platform.OS === "ios") return true;
  if (
    Platform.OS === "android" &&
    PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION
  ) {
    const apiLevel = parseInt(Platform.Version.toString(), 10);
    if (apiLevel < 31) {
      const granted = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION
      );
      return granted === PermissionsAndroid.RESULTS.GRANTED;
    }
    if (
      PermissionsAndroid.PERMISSIONS.BLUETOOTH_SCAN &&
      PermissionsAndroid.PERMISSIONS.BLUETOOTH_CONNECT
    ) {
      const result = await PermissionsAndroid.requestMultiple([
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_SCAN,
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_CONNECT,
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
      ]);
      return (
        result["android.permission.BLUETOOTH_CONNECT"] ===
          PermissionsAndroid.RESULTS.GRANTED &&
        result["android.permission.BLUETOOTH_SCAN"] ===
          PermissionsAndroid.RESULTS.GRANTED &&
        result["android.permission.ACCESS_FINE_LOCATION"] ===
          PermissionsAndroid.RESULTS.GRANTED
      );
    }
  }
  return false;
}
```

## Initializing the BLE Manager

Create a singleton BLE manager instance to manage BLE operations:

```ts
import { BleManager } from "react-native-ble-plx";

export const manager = new BleManager();
```

Or as a singleton class:

```ts
import { BleManager } from "react-native-ble-plx";

class BLEServiceInstance {
  manager: BleManager;
  constructor() {
    this.manager = new BleManager();
  }
}
export const BLEService = new BLEServiceInstance();
```

## Scanning for Devices

Start scanning after permissions are granted and BLE is powered on:

```js
function scanAndConnect() {
  manager.startDeviceScan(null, null, (error, device) => {
    if (error) {
      // Handle error
      return;
    }
    if (device.name === "MyDeviceName") {
      manager.stopDeviceScan();
      // Proceed with connection
    }
  });
}
```

Monitor BLE state and trigger scanning:

```js
import React from "react";
React.useEffect(() => {
  const subscription = manager.onStateChange((state) => {
    if (state === "PoweredOn") {
      scanAndConnect();
      subscription.remove();
    }
  }, true);
  return () => subscription.remove();
}, [manager]);
```

## Connecting to Devices

Connect and discover services/characteristics:

```js
device
  .connect()
  .then((device) => device.discoverAllServicesAndCharacteristics())
  .then((device) => {
    // Ready to interact with device
  })
  .catch((error) => {
    // Handle connection errors
  });
```

## Reading & Writing Characteristics

**Reading:**

```js
device
  .readCharacteristicForService(serviceUUID, characteristicUUID)
  .then((characteristic) => {
    console.log("Read value:", characteristic.value);
  })
  .catch((error) => {
    console.error("Read error:", error);
  });
```

**Writing:**

```js
device
  .writeCharacteristicWithResponseForService(
    serviceUUID,
    characteristicUUID,
    value
  )
  .then(() => {
    console.log("Write success");
  })
  .catch((error) => {
    console.error("Write error:", error);
  });
```

## Monitoring State & Disconnections

**Monitor device disconnection:**

```ts
const setupOnDeviceDisconnected = (deviceIdToMonitor: string) => {
  manager.onDeviceDisconnected(deviceIdToMonitor, (error, device) => {
    if (error) {
      console.error(error);
    }
    if (device) {
      // Optionally reconnect
      device.connect();
    }
  });
};
```

**Enable verbose logging for debugging:**

```js
manager.setLogLevel("Verbose");
```

## Troubleshooting & Best Practices

- Always check and request permissions before scanning.
- Stop scanning once the target device is found to save battery.
- Use a singleton BLE manager to avoid multiple instances.
- Clean up listeners and subscriptions in `useEffect` cleanup.
- For Android 12+, use `neverForLocation` flag to avoid unnecessary location permissions if not needed.
- Test on real devices; BLE does not work on emulators/simulators.
- Handle errors gracefully and inform users about permission issues or BLE state.

## References

- [Official react-native-ble-plx Docs](https://github.com/dotintent/react-native-ble-plx)
- [Expo BLE Integration Blog](https://expo.dev/blog/how-to-build-a-bluetooth-low-energy-powered-expo-app)
- [BLE Permissions on Android](https://developer.android.com/guide/topics/connectivity/bluetooth/permissions)
- [BLE Permissions on iOS](https://developer.apple.com/documentation/corebluetooth)

---

This guide is based on the latest documentation and best practices for Expo and `react-native-ble-plx`. For advanced use cases, consult the official documentation and community resources.
