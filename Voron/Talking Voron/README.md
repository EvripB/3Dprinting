# Credits

Original implementation by:
https://github.com/robandwend/SV08-MultiHead-Conversion/tree/main/talkingserver

This version includes modifications such as caching so be careful 
# Hardware:  
1x ["TPA3118D2 Digital Audio Power Amplifier Board 45W*2 Daul Channel TPA3118 Stereo Power Amplifier Speaker Subwoofer DC 12V 24V 28V"](https://www.aliexpress.com/item/1005007622704966.html?spm=a2g0o.order_list.order_list_main.84.3fe31802zBjUHT)  
2x [8ohm 3W speakers](https://www.aliexpress.com/item/1005008267342362.html?spm=a2g0o.order_list.order_list_main.78.3fe31802zBjUHT)  
1x barrel power connector (5.5 * 2.1)  
1x Male-to-Male TRS jack (the ones with 3 contacts, not the ones with 4 which support microphones as well)  
4x M2x10 self-taping screws  

You can buy any amplifier/speaker you like. My criteria for the amplifier was:
1. To be able to work with 24V so i can power it from printer's power supply
2. To have an audio jack input for easy connection with rpi
3. To have a volume knob which I could insert inside skirt's hex hole to control from outside but the orientation of the mounting bracket (see below) doesn't allow for that

Voron's STLs for RPI mount in the electronics bay is very close this this amplifiers' size. Holes don't align perfectly but you can screw at least 2 of them which is sufficient.  
 Print [pcb_din_clip_x3.stl](https://github.com/VoronDesign/Voron-2/blob/Voron2.4/STLs/Electronics_Bay/pcb_din_clip_x3.stl) and [raspberrypi_bracket.stl](https://github.com/VoronDesign/Voron-2/blob/Voron2.4/STLs/Electronics_Bay/raspberrypi_bracket.stl)  

 Make sure amplifier's potentiometer is at max volume when you install the board as you probably won't have access to it later. You can adjust the volume from mainsailOS later.

# Software
The below will work on MainsailOS

### Create project directory
`mkdir -p ~/talking_voron`  
`cd ~/talking_voron`  

### Create virtual environment
`python3 -m venv venv`  
`source venv/bin/activate`  

### Install Piper
`pip install --upgrade pip setuptools wheel`  
`pip install piper-tts pathvalidate`  

### Download voice model
`python -m piper.download_voices en_US-amy-medium`  
You can find different voices and languages in [piper repo](https://huggingface.co/rhasspy/piper-voices/tree/main)

### Create helper script
Create a file `nano ~/talking_voron/say.sh`  
Paste the following code and save
```
#!/bin/bash
curl -G --data-urlencode "text=$*" http://127.0.0.1:4601/
```
*if for any reason you want to use a different port, you should modify the ttsserver.py as well. Just search for "4601" and replace with your own. It appears only 2 times at the end of the script.*  

Give "execute" permissions to the script  
`chmod +x ~/talking_voron/say.sh`  

### Download the TTS server
__Note__: This version is modified from the original project.  
In the default implementation, each request triggers Piper to generate audio from scratch, which introduces a delay of ~6 seconds per phrase.  
This modified version adds caching:  
* Each unique text is converted to a .wav file once and stored in /tmp
* Subsequent requests for the same text reuse the existing file
* Result: first execution is slow, repeated executions are nearly instant  

`wget -O ~/talking_voron/ttsserver.py https://raw.githubusercontent.com/EvripB/3Dprinting/main/Voron/Talking%20Voron/ttsserver.py`  

### Create systemd service
Create a file `sudo nano /etc/systemd/system/ttsserver.service`  
Paste the following code and save
```
[Unit]
Description=Text-to-Speech Web Server
After=network.target

[Service]
ExecStart=/home/pi/talking_voron/venv/bin/python /home/pi/talking_voron/ttsserver.py
WorkingDirectory=/home/pi/talking_voron
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

### Enable and start service
`sudo systemctl daemon-reload`  
`sudo systemctl enable --now ttsserver.service`  

### Get gcode_shell_command.py
NOTE: if you have kiauh installed already, you can skip this step
`cd ~/klipper/klippy/extras`  
`wget https://raw.githubusercontent.com/dw-0/kiauh/master/kiauh/extensions/gcode_shell_cmd/assets/gcode_shell_command.py`  

### Add Klipper macro
```
[gcode_macro SAY]
gcode:
    {% set s = params.S|default("") %}
    {% if s == "" %}
        {action_respond_info('Usage: SAY S="your text here"')}
    {% else %}
        RUN_SHELL_COMMAND CMD=RUNSAY PARAMS="{s}"
    {% endif %}

[gcode_shell_command RUNSAY]
command: /home/pi/talking_voron/say.sh
timeout: 30.0
verbose: True
```
Save and restart klipper

### Testing
In Mainsail's UI, execute the following command in the console
`SAY S="Hello World"`  
It will take few seconds to play the first time. Run it again and it should play instantly. 

### Useful commands
Add this in your `PRINT_START` macro:  
`SAY S="Print is starting"`  

Add this in your `PRINT_END` macro:  
`SAY S="Print completed"`  

If you have [hotend temperature macro](https://github.com/EvripB/3Dprinting/blob/main/Voron/Macros/README.md#hotend-temperature-monitor-macro), add this in [delayed_gcode temp_watchdog]:  
`SAY S="warning! warning! Hotend temperature too low!"`  

To modify the volume of your rpi, first run:  
`amixer scontrols`  
The output should be something like this: `Simple mixer control 'PCM',0`. You need the text between the ' ' which, in my case, is PCM. If you get both 'PCM' and 'Master', try both of them in the below command to see which one works for you.

Set the desired volume represended in percentage and replace 'PCM' with whatever you found in the previous command:  
`amixer set PCM 95%`