# ESP32 Flash Encryption

This project demonstrates how to enable [flash encryption (Development Mode) on the ESP32](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html) and how secure storage like eFuse is used to protect sensitive keys. First, we will see how devices without flash encryption are insecure. We will steal the WiFi credentials of a firmware running on the ESP32. Then we will enable flash encryption and show how an attacker cannot steal credentials when flash encryption is enabled. Finally, we will show how the user can upload additional firmware after flash encryption has been enabled, and how flash encryption can be disabled.

## eFuses as secure key storage
In flash encryption, firmware bootloader (i.e. second stage bootloader, which is loaded by the ROM bootloader) generates an AES-256 bit key and saves it in the *flash_encryption* eFuse, which is a secure storage. Software cannot access this key, which is used entirely by hardware for flash encryption. The reason why this AES-256 bit key is protected in the secure storage is if it is lost, an attacker can use it to decrypt a dumped firmware.

ESP32 has 4 eFuse blocks as secure key storage as the figure below shows. Each block is 256 bits.
The figure shows the AES key storage (i.e. flash_encryption eFuse) in eFuses. Please note there are many [other eFuses](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html#relevant-efuses) for different purposes.

<img src="imgs/eFuseBLKs.PNG" width=500>

## Setup

For this project, we need to use a variety of ESP-IDF tools. For simplicity, we will utilize the ESP-IDF extension within VS Code and its terminal to use the tools.

### 0. Prerequisite: Install the ESP-IDF extension 
Our new Ubuntu VM installed with ESP-IDF and Visual Studio Code with the ESP-IDF extension. If you do not want to install it yourself please download the VM from the specified website. 

Otherwise, please follow [this tutorial](https://github.com/espressif/vscode-esp-idf-extension/blob/master/docs/tutorial/install.md) to install it within VS Code.

### 1. Download this repository

After installing ESP-IDF, download this repository into your VM:
```sh
git clone https://github.com/PBearson/ESP32_Flash_Encryption_Tutorial.git
```
 **Note**: By default, this project is already located in the ``` ~/esp/IoT-Examples/ ``` directory of the Ubuntu VM.

### 2. Load the project into VS Code
Use *File* -> *Open Folder ...* to load the project into VS Code

### 3. Start terminal and set up the environment variables

We will start the ESP-IDF terminal within VS Code (not the Linux terminal) using the button at the bottom of VS Code. 

<img src="imgs/VSCode-ESP-Terminal.png">

Otherwise Within a Linux terminal, we will need to set up the environment variables so that we can use all the tools without inputting the full path names. This can be done with the ``` export.sh ``` script as shown below.
```sh
. $HOME/esp/esp-idf/export.sh
```

<img src="imgs/VSCodeTerminal.png">

**All later commands shall be entered in the same terminal**.

## Steal WiFi Credentials in Plaintext Flash

First, we will see how an attacker can steal the credentials from a plaintext firmware.

### 4. Configure the WiFi Application

Within the terminal with the configured environment, navigate to the root directory of this project if needed. Open the project configuration menu:

```sh
idf.py menuconfig
```

Using the up/down arrow keys, navigate to the `Example Configuration` menu, press _enter_ to enter into the menu, then press _enter_ to begin typing your WiFi SSID. When you are done, do the same for the WiFi Password. After you are done, press _ESC_ several times until you are prompted to save. Press _Y_ to save and exit.

### 5. Upload the Application

Now build, upload, and monitor the app. **NOTE: If using the Hiletgo ESP-WROOM-32 development board, you may need to hold down the IO0 button on the ESP32 when the build system tries to connect to the ESP32's serial port. If you do not hold down the IO0 button during this step, the build system may fail to detect the serial port.**

```sh
idf.py build flash monitor
```
*CTRL+]* is used to terminate the serial monitor when needed.

After a few minutes of compiling, the project will be flashed to the board, and the serial terminal will connect to the ESP32. You should see the application successfully connect to your WiFi:

![image](https://user-images.githubusercontent.com/11084018/158292160-46c9c3f7-0633-4d00-b2e4-19b427ad6cea.png)

### 6. Steal WiFi Credentials

Now we will attempt to steal the WiFi credentials by reading the flash contents of the ESP32 directly. Download the first 65536 bytes of the firmware from the ESP32 using the following command:

```sh
esptool.py read_flash 0x10000 0x10000 flash.bin
```

This command reads the contents from the flash chip in the address range 0x10000 - 0x20000. The output is written to the file `flash.bin`.

Now locate the WiFi credentials in the firmware with the following command:

```sh
strings flash.bin | grep -A1 <SSID>   # Replace <SSID> with your WiFi SSID
```

## Enable Flash Encryption

Now we will enable flash encryption, a security mechanism supported by the ESP32 that defeats attacks like the one demonstrated above. The ESP32 supports two kinds of flash encryption modes: Release, and Development:

* In **Release Mode**, flash encryption is permanently enabled, and the DISABLE_DL_ENCRYPT eFuse is burned. This means the user cannot upload anymore plaintext firmware to the board via UART any more, and the firmware can only be updated via OTA in the release mode.
* In **Development Mode**, flash encryption can be disabled a limited number of times, and the DISABLE_DL_ENCRYPT eFuse is not burned. This allows the user to upload new plaintext firmware.

For this project, we will only use Development Mode.

### 7. Configure the Application

Open the configuration menu again:

```sh
idf.py menuconfig
```

Navigate to the menu `Security features`. Select the option `Enable flash encryption on boot`. By default, the usage mode should be selected as `Developmeent (NOT SECURE)`. Do not change this setting.

Press ESC and change to the `Partition Table` menu. We need to change the offset of the partition table because the bootloader (which is stored in flash _before_ the partition table) will grow in size. Change the offset of the partition table from 0x8000 to 0x10000.

Press ESC and change to the `Component config` menu. From here, navigate to the `NVS` menu. Disable the option `Enable NVS encryption`, since we are not interested in encrypting the NVS (non-volatile storage) partition.

Do not touch other configurations. Now leave and save the configuration.

### 8. Upload the Application

Build, flash, and monitor the application just as before. Hold down the IO0 button whenever needed.

```sh
idf.py build flash monitor
```

When the serial terminal connects to the ESP32, you should see that the bootloader, partition table, and firmware are all encrypted. If you see errors, continue to read.

![image](https://user-images.githubusercontent.com/11084018/158298515-6e9a4f03-aceb-4077-ae67-d2e44b9dcca5.png)

If errors like "*flash read err, 1000*" as follows occur after running *idf.py build flash monitor*, use the following two commands to solve the problems
```
espefuse.py burn_efuse FLASH_CRYPT_CNT
idf.py encrypted-flash
````

![image](https://user-images.githubusercontent.com/69218457/162789473-2f6f5b53-7296-4fb9-936a-a6ec54d657e9.png)


## 9. Try to Steal WiFi Credentials

Now we will download the firmware and try to steal the WiFi credentials, just as before. Since we changed the partition table offset, the firmware was actually flashed at offset 0x20000 instead of 0x10000. Therefore, to download the first 65536 bytes of the firmware, use the following command:

```sh
esptool.py read_flash 0x20000 0x10000 flash_encrypted.bin
```

Now try to read the credentials just like before:

```sh
strings flash_encrypted.bin | grep -A1 <SSID>   # Replace <SSID> with your WiFi SSID
```

You will see that nothing is returned. This indicates that flash encryption is enabled and the WiFi credentials can no longer be recovered.

## 10. Disable Flash Encryption

To disable flash encryption, please follow the steps below.

**CAUTION: The following procedure can only be performed 3 times, due to the limited size of the FLASH_CRYPT_CNT eFuse.**

First, open the configuration menu, navigate to `Security features`, and disable the option `Enable flash encryption on boot`.

Next, build and flash the application: 

```sh
idf.py build flash
```

Finally, set the next bit in the FLASH_CRYPT_CNT eFuse to disable flash encryption:

```sh
espefuse.py burn_efuse FLASH_CRYPT_CNT
````

Follow the instructions and type `BURN` to finish setting the eFuse.

**NOTE**: This may throw an error ``` A fatal error occurred: New value required for efuse 'FLASH_CRYPT_CNT' (given None)  ``` in this case we need to provide the bit value to set.  This needs to be some binary value with an even number of bits, the first value in my case was 3 and this worked.

```sh
espefuse.py burn_efuse FLASH_CRYPT_CNT
```

We can use the command below to see the current efuse state. This will be after the *=* sign.
```sh
espefuse.py summary | grep -A 3 -B 3 "FLASH_CRYPT"
```

Now use the serial monitor to observe the output from the ESP32. There is no more flash encyption information being printed any more. Preset the reset button of the ESP32 and restart it if there is no output from the ESP32.
```sh
idf.py monitor
```

## Notes

### Flash Plaintext Firmware After Flash Encryption is Enabled

As long as the DISABLE_DL_ENCRYPT eFuse remains 0, the user can always upload plaintext firmware to the board, even after flash encryption is enabled. The firmware will be encrypted by the ESP32's UART bootloader when the user uploads new firmware. To upload new firmware after flash encryption is enabled, simply replace all `idf.py flash` commands with `idf.py encrypted-flash`.

### Re-enable Flash Encryption

To re-enable flash encryption, enable the option `Enable flash encryption on boot` in the configuration menu, and build and flash the application. The bootloader will automatically set the next bit in the FLASH_CRYPT_CNT eFuse to enable flash encryption; there is no need for the user to manually burn the efuse.

### Enable Secure Boot After Flash Encryption is Enabled

You can follow [this procedure](https://github.com/PBearson/ESP32_Secure_Boot_Tutorial) to enable secure boot even after flash encryption already enabled. There is no need to disable flash encryption as long as the DISABLE_DL_ENCRYPT eFuse is set to 0.

### espefuse.py
```sh
espefuse.py summary
```

### Possible failures

[Typical failure cases](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/security/flash-encryption.html#possible-failures)
