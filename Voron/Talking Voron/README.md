All credits go to original creator: https://github.com/robandwend/SV08-MultiHead-Conversion/tree/main/talkingserver

# Hardware:  
1x ["TPA3118D2 Digital Audio Power Amplifier Board 45W*2 Daul Channel TPA3118 Stereo Power Amplifier Speaker Subwoofer DC 12V 24V 28V"](https://www.aliexpress.com/item/1005007622704966.html?spm=a2g0o.order_list.order_list_main.84.3fe31802zBjUHT)  
2x [8ohm 3W speakers](https://www.aliexpress.com/item/1005008267342362.html?spm=a2g0o.order_list.order_list_main.78.3fe31802zBjUHT)  
1x barrel power connector (5.5 * 2.1)  
1x Male-to-Male TRS jack (the ones with 3 contacts, not the ones with 4 which support microphones as well)  

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
`python -m piper.download_voices en_GB-southern_english_female-low`  

### Create helper script
Create a file `nano ~/talking_voron/say.sh`  
Paste the following code and save
```
#!/bin/bash
curl -G --data-urlencode "text=$*" http://127.0.0.1:4601/
```
Give "execute" permissions to the script  
`chmod +x ~/talking_voron/say.sh`  

### Download the TTS server

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
