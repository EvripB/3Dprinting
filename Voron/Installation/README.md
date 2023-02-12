The below consists my personal documentation of the steps performed when I built my Voron and should not be considered as a guide for someone else.

- [Install Mainsail](#install-mainsail)
- [Boot Rpi](#boot-rpi)
- [Build & install Firmware](#build---install-firmware)
- [Wi-Fi disconnections fix](#Wi-Fi-disconnections-fix)

# Install Mainsail

Guide followed: https://docs.vorondesign.com/build/software/installing_mainsail.html
1)	Download MainsailOS (around 1GB). It should be a .zip file: https://github.com/raymondh2/MainsailOS/releases

2)	Unzip the downloaded file. It should contain an .img file

3)	Open balenaEtcher

4)	Select the .img file

5)	Select the SD card. <span style="color:red">**Careful! It might have selected a normal drive**</span>

6)	Open mainsailos-wpa-supplicant.txt and add the WiFi connections. Under “WPA/WPA2 secured”, add the following two networks:

```
## WPA/WPA2 secured
network={
  ssid="addYourSSID"
  psk="addYourPasswordHere"
}

network={
  ssid="addAnotherSSID"
  psk=" addYourPasswordHere"
}
```

7)	Scroll near the end of the file where the country is set. Comment out any uncommented country and add the following line:

```
country=GR # Greece
```

8)	Safely remove the memory card and insert it into the Rpi

# Boot Rpi

1)	Connect Rpi to power and wait to boot
2)	Check modem’s DHCP server and search for a new record. It should mention “mainsail” as a name
3)	Note down the MAC address of the Rpi and set a static IP in DHCP server
4)	SSH into Rpi
5)	Execute: date and check the result. The date and time should be correct. If it isn’t:  
a.	Execute: `sudo raspi-config`  
b.	Select option “5 Localisation Options”  
c.	Select option “L2 Timezone”  
d.	Select “Europe”  
e.	Select “Athens”  
f.	Select “Finish” (of whatever.. just exit the config)  
6)	Execute: `sudo raspi-config` and select “8 Update”. Wait until it finishes. Usually there are no updates to be performed

# Build & install Firmware

Guide followed: https://docs.vorondesign.com/build/software/spider_klipper.html
1)	Power-on the Rpi
2)	Execute: `sudo apt install make`
3)	Execute: `cd ~/klipper`
4)	Execute: `make clean`
5)	Execute: `make menuconfig`

In the menu that opened, do the following:
1)	The first item you should land on is “Enable extra low-level configuration options”. Press spacebar to enable it
2)	Move down to “Micro-controller Architecture” and press spacebar to go into the submenu. In the submenu, move down to STMicroelectronics STM32 and press spacebar. It should move you back to the previous menu and “STMicroelectronics STM32” should appear in a parenthesis next to the Micro-Controller Architecture menu.
3)	Move down to “Processor model” and press spacebar. In the submenu, move down to STM32F446 and press space bar
4)	Move down to “Bootloader offset” and press spacebar. In the submenu, select “64KiB bootloader” and press spacebar. <span style="color:red">**Note**</span>: in the documentation, the following is mentioned:  
![bootloader_offset](https://github.com/EvripB/3Dprinting/blob/main/Voron/Installation/images/bootloader_offset.png?raw=true)  
Asked in Voron Discord and there is no way we can tell when the board was made. I ordered mine on 15/7/2021 which is very close to the above dates. I was advised to try with 64KiB first. If the value is wrong, the board won’t boot  

5)	Move down to “Clock Reference” and press spacebar. In the submenu, select “12 MHz crystal”
6)	Move down to “Communication interface”. In the submenu, select “USB (on PA11/PA12)”.
7)	The final configuration should look like this:  
![make_config](https://github.com/EvripB/3Dprinting/blob/main/Voron/Installation/images/make_config.png?raw=true)

Validate the configuration is correct and press “q” to exit.  

Press “Y” to save the configuration

8)	Execute: `make`
The output is the firmware that needs to be installed on the board. It is located in `/home/pi/klipper/out` and it is named `klipper.bin`.

9)	There are several ways to flash the firmware into the Spider board. We will use the sd-card method mentioned in the documentation. Login into Rpi via WinSCP or Moba’s SFTP and download the klipper.bin we created in step (8). I maintain these firmwares in `E:\MyFiles\Documents\3D_Printing\Voron\Software\klipper`  
10)	Rename the downloaded file from `klipper.bin` to `firmware.bin`  
11)	Connect the sd-card and paste the firmware.bin file into it. The card should be formatted in FAT32.  
12)	Remove the card from PC, insert it into Spider board and power-on the printer. There is a flashing-red led on the board which will light-up while installing the new firmware. Wait a minute or two (I’m not sure if the flash is finished when the light turns off) and turn off the printer
13)	Remove the sd-card from the printer  


# Wi-Fi disconnections fix

Guide followed: https://weworkweplay.com/play/rebooting-the-raspberry-pi-when-it-loses-wireless-connection-wifi/

1)	Log into Rpi and execute:  
`cd /usr/local/bin`  
`sudo touch checkwifi.sh`  
`sudo chmod 775 checkwifi.sh`  
`sudo vi checkwifi.sh`  

2)	Paste the following code
```
ping -c4 192.168.1.254 > /dev/null

if [ $? != 0 ]
then
  echo "No network connection, restarting wlan0"
  /sbin/ifdown 'wlan0'
  sleep 5
  /sbin/ifup --force 'wlan0'
fi
```
3)	Execute `crontab -e` and add the following line  
```*/1 * * * * /usr/bin/sudo -H /usr/local/bin/checkwifi.sh >> /dev/null 2>&1```

4)	Verify cron execution with the below command  
```sudo grep CRON /var/log/syslog```
