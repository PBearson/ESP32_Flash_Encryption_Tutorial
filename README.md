# ESP32 Flash Encryption Tutorial

## Prerequisite - Steal WiFi Credentials in Plaintext Flash

Open the configuration menu:

```
idf.py menuconfig
```

Using the up/down arrow keys, navigate to the `Example Configuration` menu, press enter to enter into the menu, then press enter to begin typing your WiFi SSID. When you are done, do the same for the WiFi Password. After you are done, press ESC several times until you are prompted to save. Press Y to save and exit.

Now build, upload, and monitor the app:

```
idf.py build flash monitor
```

**NOTE: If using the Hiletgo ESP-WROOM-32 development board, you may need to hold down the IO0 button on the ESP32 when the build system tries to connect to the ESP32's serial port. If you do not hold down the IO0 button during this step, the build system may fail to detect the serial port.**

After a few minutes of compiling, the project will be flashed to the board, and the serial terminal will connect to the ESP32. You should see the application successfully connect to your WiFi:

![image](https://user-images.githubusercontent.com/11084018/158292160-46c9c3f7-0633-4d00-b2e4-19b427ad6cea.png)
