# ESP32 Flash Encryption Tutorial

## Prerequisite - Steal WiFi Credentials in Plaintext Flash

### Configure the WiFi Application

Open the configuration menu:

```
idf.py menuconfig
```

Using the up/down arrow keys, navigate to the `Example Configuration` menu, press enter to enter into the menu, then press enter to begin typing your WiFi SSID. When you are done, do the same for the WiFi Password. After you are done, press ESC several times until you are prompted to save. Press Y to save and exit.

### Upload the Application

Now build, upload, and monitor the app:

```
idf.py build flash monitor
```

**NOTE: If using the Hiletgo ESP-WROOM-32 development board, you may need to hold down the IO0 button on the ESP32 when the build system tries to connect to the ESP32's serial port. If you do not hold down the IO0 button during this step, the build system may fail to detect the serial port.**

After a few minutes of compiling, the project will be flashed to the board, and the serial terminal will connect to the ESP32. You should see the application successfully connect to your WiFi:

![image](https://user-images.githubusercontent.com/11084018/158292160-46c9c3f7-0633-4d00-b2e4-19b427ad6cea.png)

### Steal WiFi Credentials

Download the first 65536 bytes of the firmware from the ESP32 using the following command:

```
esptool.py read_flash 0x10000 0x10000 flash.bin
```

This command reads the contents from the flash chip in the address range 0x10000 - 0x20000. The output is written to the file `flash.bin`.

Now locate the WiFi credentials in the firmware with the following command:

```
strings flash.bin | grep -A1 <SSID>   # Replace <SSID> with your WiFi SSID
```

## Enable Flash Encryption

Now we will enable flash encryption, a security mechanism supported of the ESP32 that defeats attacks like the one demonstrated above. The ESP32 supports two kinds of flash encryption modes: Release, and Development:

* In **Release Mode**, flash encryption is permanently enabled, and the user cannot upload anymore plaintext firmware to the board.
* In **Development Mode**, flash encryption can be disabled a limited number of times, allowing the user to upload new plaintext firmware.

For this project, we will only use Development Mode.

### Configure the Application

Open the configuration menu again:

```
idf.py menuconfig
```

Navigate to the menu `Security features`. Select the option `Enable flash encryption on boot`. By default, the usage mode should be selected as `Developmeent (NOT SECURE)`. Do not change this setting.

Press ESC and change to the `Partition Table` menu. We need to change the offset of the partition table because the bootloader (which is stored in flash _before_ the partition table) will grow in size. Change the offset of the partition table from 0x8000 to 0x10000. Now leave and save the configuration.



## Disable Flash Encryption

If the user wishes to disable flash encryption, please follow the steps below.

First, open the configuration menu, navigate to `Security features`, and disable the option `Enable flash encryption on boot`.

Next, build and flash the application: 

```idf.py build flash```

Finally, set the next bit in the FLASH_CRYPT_CNT eFuse to disable flash encrpytion:

```
espefuse.py burn_efuse FLASH_CRYPT_CNT
````

Follow the instructions and type `BURN` to finish setting the eFuse.
