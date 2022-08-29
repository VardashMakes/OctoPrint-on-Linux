# Octoprint-on-Linux (WORK IN PROGRESS)
In this tutorial, I will show you how to easily create your own OctoPrint server including a webcamDaemon using mjpg-streamer. I will walk you step by step through the installation & configuration.

Many of the commands were provided by the [Octoprint community](https://community.octoprint.org/t/setting-up-octoprint-on-a-raspberry-pi-running-raspberry-pi-os-debian/2337).

# Assumptions
- You already have a Linux distro installed
- Your using a device like a Notbook/PC that has USB ports
- This guide covers setting up a USB camera but you can skip that section if you don’t plan on using one


# Base Install

## Run Updates
```
sudo apt-get update && sudo apt-get dist-upgrade -y
```

## Install essential tools
```
sudo apt install ssh openssh-server openssh-client net-tools build-essential linux-headers-$(uname -r) python3-pip python3-dev python3-setuptools python3-venv git libyaml-dev neofetch locate -y
```

## Add pi User 
### Select a strong password
```
sudo adduser pi
```

## Add pi User to sudo group
```
sudo usermod -aG sudo pi
```

## Add pi User to Serial Ports & Video Devices
```
sudo usermod -aG tty pi
sudo usermod -aG dialout pi
sudo usermod -aG video pi
```

## Add pi User to sudoers files so it can run shutdown commands without Password promt
### This will get important later to execute commands from the OctoPrint Webinterface
```
sudo nano /etc/sudoers.d/octoprint-shutdown
pi ALL=NOPASSWD: /sbin/shutdown
```

## Update octoprint service as well:
```
sudo nano /etc/sudoers.d/octoprint-service
pi ALL=NOPASSWD: /usr/sbin/service
```

## Find the IP address of the server
### write down the IP Adress (see. underlined number in the example picture)
```
sudo ifconfig -a
```
![ifconfig_example_output](/assets/ifconfig_example_output.png)


## Reboot
### After the Server ist back on, you can login on the Console or SSH
```
sudo reboot now
```


# Login to User Pi
### Open a Commandpromt on Windows or a SSH APP ala Putty
### If you are on a Windowsterminal use the Command
```
 ssh pi@yourserverip
```

## Setup Octoprint Folder
```
cd ~
mkdir OctoPrint && cd OctoPrint
```

## Setup Virtual Environment
```
python3 -m venv venv
source venv/bin/activate
```

## Install Octoprint
```
pip install pip --upgrade
pip install octoprint
```

### Note: You can test the Octproint install now by starting the service “~/OctoPrint/venv/bin/octoprint serve” and connecting to HTTP://<Your IP>:5000. You should get a setup page.

## Set Octoprint to Automatically Start
```
wget https://github.com/OctoPrint/OctoPrint/raw/master/scripts/octoprint.service && sudo mv octoprint.service /etc/systemd/system/octoprint.service
ExecStart=/home/pi/OctoPrint/venv/bin/octoprint
sudo systemctl enable octoprint.service
```

### You can get the status with
```
sudo service octoprint {start|stop|restart|status }
```


# HAProxy Install/Setup
### HAProxy is used to serve the front end of the site on port 80 and direct the traffic to the backend ports as needed.

## Install HAProxy
```
sudo apt install haproxy
```

## Update HAProxy Config
```
sudo nano /etc/haproxy/haproxy.cfg
```

### Add this to the bottom of the config
```
frontend public
        bind *:80
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        http-request replace-path /webcam/(.*)   /\1
        server webcam1  127.0.0.1:8080
```
### press CTRL+X to save and exit the file

# Install/Setup Webcam Support
## Build mjpg-streamer
```
cd ~
sudo apt install subversion libjpeg62-turbo-dev imagemagick ffmpeg libv4l-dev cmake -y
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
export LD_LIBRARY_PATH=.
make
```

## Set mjpg-streamer to Autostart
```
mkdir /home/pi/scripts
nano /home/pi/scripts/webcamDaemon
```

## Copy code below into webcamDaemon file just created
```
#!/bin/bash

MJPGSTREAMER_HOME=/home/pi/mjpg-streamer/mjpg-streamer-experimental
MJPGSTREAMER_INPUT_USB="input_uvc.so"
MJPGSTREAMER_INPUT_RASPICAM="input_raspicam.so"

# init configuration
camera="auto"
camera_usb_options="-r 640x480 -f 10"
camera_raspi_options="-fps 10"

if [ -e "/boot/octopi.txt" ]; then
    source "/boot/octopi.txt"
fi

# runs MJPG Streamer, using the provided input plugin + configuration
function runMjpgStreamer {
    input=$1
    pushd $MJPGSTREAMER_HOME
    echo Running ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    LD_LIBRARY_PATH=. ./mjpg_streamer -o "output_http.so -w ./www" -i "$input"
    popd
}

# starts up the RasPiCam
function startRaspi {
    logger "Starting Raspberry Pi camera"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_RASPICAM $camera_raspi_options"
}

# starts up the USB webcam
function startUsb {
    logger "Starting USB webcam"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_USB $camera_usb_options"
}

# we need this to prevent the later calls to vcgencmd from blocking
# I have no idea why, but that's how it is...
vcgencmd version

# echo configuration
echo camera: $camera
echo usb options: $camera_usb_options
echo raspi options: $camera_raspi_options

# keep mjpg streamer running if some camera is attached
while true; do
    if [ -e "/dev/video0" ] && { [ "$camera" = "auto" ] || [ "$camera" = "usb" ] ; }; then
        startUsb
    elif [ "`vcgencmd get_camera`" = "supported=1 detected=1" ] && { [ "$camera" = "auto" ] || [ "$camera" = "raspi" ] ; }; then
        startRaspi
    fi

    sleep 120
done
```
### press CTRL+X to save and exit the file

## Setup Permissions on webcam File
```
chmod +x /home/pi/scripts/webcamDaemon
```

## Create webcamd Service
```
sudo nano /etc/systemd/system/webcamd.service
```
## Copy code below into webcamd service file just created
```
[Unit]
Description=Camera streamer for OctoPrint
After=network-online.target OctoPrint.service
Wants=network-online.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/scripts/webcamDaemon

[Install]
WantedBy=multi-user.target
```
### press CTRL+X to save and exit the file


## Enable the webcamd Service
```
sudo systemctl daemon-reload
sudo systemctl enable webcamd
```

## Reboot
```
sudo reboot
```

## Webcam URLs
```
Stream URL: /webcam/?action=stream
Snapshot URL: http://127.0.0.1:8080/?action=snapshot
Path to FFMPEG: /usr/bin/ffmpeg
```