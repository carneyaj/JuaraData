# Raspberry Pi Startup Guide

Explanation

## Installing the Operating System

1. Download Raspian Buster Lite from https://www.raspberrypi.org/downloads/raspbian/

2. Flash to micro sd card using BalenaEtcher: https://www.balena.io/etcher/

3. Remove sd card from computer then reinsert, then move to the drive directory in terminal ```cd /Volumes/boot``` on mac, something similar on windows in putty or some other command line program.

4. Configure the pi to connect to your home wifi network. Type ```nano wpa_supplicant.conf``` to bring up a text editor window, then paste in the following code, replacing YOURSSID with the network name and YOURPASSWORD with the network password.
````ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US
network={
   ssid="YOURSSID"
  psk="YOURPASSWORD"
   scan_ssid=1
}
````
After entering the above, hit ```ctrl+X```, then ```y``` then ```enter``` to save the file and exit.

5. Type ```nano ssh```, type ssh (or anything you want, it doesn't matter) in the file, then ```ctrl+X```, ```y```, ```enter``` to create a blank file called ssh which will trigger the pi to allow incoming ssh connections (so that you can access it from your computer).

6. Type ```nano config.txt```, and paste in ```dtoverlay=dwc2``` after the last line, then save and exit as above.

7. Type ```nano cmdline.txt```, and paste in ```modules-load=dwc2,g_ether``` with one space on either side between ```rootwait``` and ```quiet```, then save and exit as above.

8. Safely eject the sd card, insert in the raspberry pi, then connect the pi to a usb power source. Give it a couple minutes to start up (the first startup will take longer than subsequent ones)

## Your Pi is now setup and running!
Next up is logging in and changing the password.

1. Log into your router and view connected devices to find the ip address of the raspberry pi, probably something like ```192.168.0.xxx```. Then in terminal, type ```ssh pi@192.168.0.xxx```. Type ```yes``` to accept the connection, then type in the default password, which is ```raspberry``` (we'll change this next). You're now connected to the pi, and should see something like ```pi@raspberrypi:~ $``` in terminal.

2. (only if you get an error with the above because you've previously accessed a pi or something else at this ip address) Delete the existing ssh key by typing ```ssh-keygen -R 192.168.0.xxx``` with the relevant ip address. Now try step 1 again.

3. Type ```sudo raspi-config``` to access the configurations menu. Hit enter on the first option to change your password, type in ```Jaguars14``` followed by enter twice, then use the down then right arrows to navigate to ```finish``` in the menu, and hit enter to exit. To log in in the future, repeat step one using this new password.

## Next we set up the audio
Skip these steps if you're not using the audio-injector hat to record audio.

1. Log out of the pi by typing ```exit```.

2. Download the audio-injector package from http://forum.audioinjector.net/viewtopic.php?f=5&t=3 and unpack it.

3. Copy the .deb file to the pi by typing ```scp /Users/alex/Downloads/audio.injector.scripts_0.1-1_all.deb pi@192.168.0.227:/home/pi/``` with the correct dowload path on your computer (you can just drag and drop the file onto terminal after typing ```scp``` instead of typing it) and correct ip address for the pi. Type the pi's password when prompted.

4. ssh back into the raspberry pi as above. Type ```ls``` to display the files in the home directory, and verify that ```audio.injector.scripts_0.1-1_all.deb``` now appears.

5. Type ```sudo dpkg -i audio.injector.scripts_0.1-1_all.deb``` (note: once you start typing ```aud``` hitting tab will autocomplete the full filename). When that completes, type ```sudo apt-get -f install```. Finally, type ```audioInjector-setup.sh```, and type ```y``` when prompted. Each step could take a couple minutes to run. Once this completes, the audio-injector card is installed.

6. Reboot the pi by typing ```sudo reboot```. This will terminate the ssh connection, so give it a minute or so to reboot then ssh back in.

7. Type ```alsactl --file /usr/share/doc/audioInjector/asound.state.MIC.thru.test restore``` to setup microphone input. Type ```alsamixer``` to verify that the system recognizes the soundcard, and use the function and arrow keys to adjust the input and output levels as needed.

8. Install FFmpeg, an audio and video conversion library, by typing ```sudo apt-get install ffmpeg```. This is a large library and will take some time to download and install, so this is a good time to open your second beer.

## Install rclone to enable easy uploads to various cloud services

1. Type ```curl https://rclone.org/install.sh | sudo bash``` to download and install RClone.

2. Type ```rclone config``` and follow the relevant instructions for whichever cloud service you want to use, as described here: https://rclone.org/docs/.  Note that you can setup multiple services if desired.

3. Note: to successfully complete the configuration on a headless pi, you will also need to install rclone on your own computer, then type in ```rclone authorize "dropbox"``` (or whatever relevant service) in a different terminal window from the one logged into the pi to prompt logging in to your account in a web browser to authorize access.

## All the necessary software and libraries are now installed. Now it's time to write some programs using them.

1. Create a directory for the image and audio data you're going to produce. You can organize this however you want. Type ```mkdir NEWFOLDERNAME``` to make a directory, ```cd NEWFOLDERNAME``` to move into that directory, use ```mkdir``` to create further subdirectories, ```ls```to display the contents of the current directory, and ```cd ~``` to return to the home directory. For the following example, I've created two directories off the home directory: ```local_data``` and ```upload_data```.

2. Enable the camera by typing ```sudo raspi-config``` and navigating to ```5 interfacing options```.

3. Return to the home directory (```cd ~```). We now write a bash script which takes a photo and saves it to a subdirectory based on the date and time. Type ```nano capture.sh```. Paste in the following code:
````
#!/bin/bash
DATE=$(date +"%Y-%m-%d_%H%M")
raspistill -vf -hf -o /home/pi/upload_data/$DATE.jpg
````
Save the file, then type ```chmod +x capture.sh``` to make it executable.

4. Next, write a bash script which records one minute of audio to the local folder, then converts it to mp3, saves that to the upload folder, then deletes the .wav file to save space: ```nano record.sh``` then paste in:
````
#!/bin/bash
DATE=$(date +"%Y-%m-%d_%H%M")
arecord -d 10 -f cd -c 1 /home/pi/local_data/$DATE.wav
ffmpeg -i /home/pi/local_data/$DATE.wav -b:a 64k /home/pi/upload_data/$DATE.mp3
rm /home/pi/local_data/$DATE.wav
````
The options, in order, mean 60 second duration, cd quality (44.1khz, 16bit), and 1 channel (mono). Save and make this executable.

5. Next up, a program ```sync.sh``` that syncs the ```upload_data``` directory to the cloud:
````
#!/bin/bash
rclone copy /home/pi/upload_data dropboxremote:Apps/pisync
````
Save this program and make it executable.

6. Finally, we make everything run on a schedule:
type ```crontab -e``` and pick option one to edit with nano. Next, copy in the following three lines at the bottom:
```
 */10 * * * * /home/pi/capture.sh
 */10 * * * * /home/pi/record.sh
 */10 * * * * /home/pi/sync.sh
````
This runs each of the above files every ten minutes.
