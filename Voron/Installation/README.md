Software installation

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
