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

After a few minutes of compiling, the project will be flashed to the board, and the serial terminal will connect to the ESP32. You should see the application successfully connect to your WiFi.
