# ESP32 Flash Encryption Tutorial

This project demonstrates how to enable flash encryption (Development Mode) on the ESP32. First, we will see how devices without flash encryption are insecure. We will steal the WiFi credentials of a firmware running on the ESP32. Then we will enable flash encryption and show how an attacker cannot steal credentials when flash encryption is enabled. Finally, we will show how flash encryption can be disabled.

## Prerequisites

For this project, you need:

* An ESP32 development board, such as Hiletgo ESP-WROOM-32.
* A Linux Virtual Machine with ESP-IDF (version 4) installed.

You are encouraged to use the following Ubuntu VM: https://www.dropbox.com/s/0g7w8qduzj2rb1k/UbuntuIoT.ova?dl=0

Follow these instructions to install ESP-IDF version 4.4: https://docs.espressif.com/projects/esp-idf/en/v4.4/esp32/get-started/index.html#get-started-get-prerequisites

After installing ESP-IDF, download this repository into your VM:

```
git clone https://github.com/PBearson/ESP32_Flash_Encryption_Tutorial.git
```

## Steal WiFi Credentials in Plaintext Flash

First, we will see how an attacker can steal the credentials from a plaintext firmware.

### Configure the WiFi Application

Open a terminal, and navigate to the root directory of this project. Open the project configuration menu:

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

Press ESC and change to the `Partition Table` menu. We need to change the offset of the partition table because the bootloader (which is stored in flash _before_ the partition table) will grow in size. Change the offset of the partition table from 0x8000 to 0x10000.

Press ESC and change to the `Component config` menu. From here, navigate to the `NVS` menu. Disable the option `Enable NVS encryption`, since we are not interested in encrypting the NVS (non-volatile storage) partition.

Now leave and save the configuration.

### Upload the Application

Build, flash, and monitor the application just as before:

```
idf.py build flash monitor
```

When the serial terminal connects to the ESP32, you should see that the bootloader, partition table, and firmware are all encrypted:

![image](https://user-images.githubusercontent.com/11084018/158298515-6e9a4f03-aceb-4077-ae67-d2e44b9dcca5.png)

### Try to Steal WiFi Credentials

Now we will download the firmware and try to steal the WiFi credentials, just as before. Since we changed the partition table offset, the firmware was actually flashed at offset 0x20000 instead of 0x10000. Therefore, to download the first 65536 bytes of the firmware, use the following command:

```
esptool.py read_flash 0x20000 0x10000 flash_encrypted.bin
```

Now try to read the credentials just like before:

```
strings flash_encrypted.bin | grep -A1 <SSID>   # Replace <SSID> with your WiFi SSID
```

You will see that nothing is returned. This indicates that flash encryption is enabled and the WiFi credentials can no longer be recovered.

## Disable Flash Encryption

If the user wishes to disable flash encryption, please follow the steps below.

**CAUTION: The following procedure can only be performed 3 times, due to the limited size of the FLASH_CRYPT_CNT eFuse.**

First, open the configuration menu, navigate to `Security features`, and disable the option `Enable flash encryption on boot`.

Next, build and flash the application: 

```idf.py build flash```

Finally, set the next bit in the FLASH_CRYPT_CNT eFuse to disable flash encrpytion:

```
espefuse.py burn_efuse FLASH_CRYPT_CNT
````

Follow the instructions and type `BURN` to finish setting the eFuse.
