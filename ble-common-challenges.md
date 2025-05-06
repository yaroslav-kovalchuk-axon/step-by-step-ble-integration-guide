# React Native BLE: Common Challenges and Solutions

## react-native-ble-plx

### 1. Permissions and Platform Differences

- **Android 12+**: Requires `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` permissions. For older Android, you need `ACCESS_FINE_LOCATION`.
- **Solution**: Always check and request the correct permissions at runtime. Use the `neverForLocation` flag if you don't need location for BLE scanning.
- **Code Example:**
  ```js
  if (Platform.OS === "android") {
    const apiLevel = parseInt(Platform.Version.toString(), 10);
    if (apiLevel < 31) {
      await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION
      );
    } else {
      await PermissionsAndroid.requestMultiple([
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_SCAN,
        PermissionsAndroid.PERMISSIONS.BLUETOOTH_CONNECT,
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
      ]);
    }
  }
  ```

### 2. Device Scanning and Connection

- **Challenge**: Scanning may not return devices if Bluetooth or location is off, or if permissions are missing.
- **Solution**: Always check Bluetooth state and permissions before scanning. Use `onStateChange` to wait for Bluetooth to be powered on.
- **Code Example:**
  ```js
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

### 3. Service/Characteristic Discovery

- **Challenge**: Must call `discoverAllServicesAndCharacteristics()` after connecting before reading/writing.
- **Solution**: Chain `.connect().then(device => device.discoverAllServicesAndCharacteristics())`.

### 4. Disconnection Handling

- **Challenge**: Unexpected disconnects.
- **Solution**: Use `onDeviceDisconnected` to listen for disconnects and implement auto-reconnect logic.
- **Code Example:**
  ```js
  bleManagerInstance.onDeviceDisconnected(deviceId, (error, device) => {
    if (device) device.connect();
  });
  ```

### 5. Android Manifest and Gradle

- **Challenge**: Missing permissions or repositories.
- **Solution**: Add all required permissions to `AndroidManifest.xml` and add JitPack to `build.gradle`.

### 6. Debugging

- **Challenge**: Hard to debug BLE issues.
- **Solution**: Use `setLogLevel(LogLevel.Verbose)` for detailed logs.

---

## General BLE Integration Tips

- Always check and request permissions before scanning.
- Ensure Bluetooth and (on Android) location are enabled.
- Centralize UUIDs and characteristic constants.
- Organize code: separate scanning/connection logic from device-specific communication.
- Handle disconnects and errors robustly.
- Use verbose logging for debugging.
- Test with real devices and BLE scanner apps (e.g., nRF Connect).

---

## References

- [Official react-native-ble-plx Docs](https://github.com/dotintent/react-native-ble-plx)
- [BLE integration guide and pitfalls](https://stormotion.io/blog/what-to-consider-when-integrating-ble-in-your-react-native-app/)
