# Raspberry Honey Pot

This is a project that I made to understand honeypots and how to make one myself. The project is using the opencanary daemon. 

### Hardware and Software Used:
- Raspberry pi 3B V1.2
- 16 GB Micro SD Card
- Ethernet Cable
- Raspbian OS Lite x64
- Opencanary


### Installation and Setup:

------------
#### 1. Installation :
Download and flash Raspbian OS lite on the micro SD card. The selection of Raspbian lite is purely on the basis that it's being run on a Raspberry Pi so it'll be easier on the processing and won't require a GUI considering it's just a honeypot. I just used the Raspberry Pi imager software given out by the official Raspberry Pi community. 

	https://www.raspberrypi.com/software/

  After that connect your micro SD card using a card reader, and open the Raspberry Pi Imager tool.
  Select the Raspberry Pi, mine was Raspberry Pi 3, and then for the OS go to "Raspberry Pi OS (Other) >    Raspberry Pi OS Lite(64-Bit)". Finally, select the connected sd card to flash(Make sure that you   select the correct device as it'll format whatever has been selected and accidentally selecting an important drive will wipe it).
 
  At this point click next and a pop-up will show up asking for OS customization. Click on edit and   change the things you'd like to change and add. I changed the hostname to something that sounds like a home NAS storage name and configured login credentials and also added my main home WIFI network so that it can connect to it as soon as it boots up.  Make sure to enable SSH in the services tab to easily SSH into the honeypot Pi as soon as it boots the first time.

------------
#### 2. Basic Setup :
Now comes the part to set up the Honeypi. Using a network scanner like Angry-IP scanner to figure out which IP has the RPi taken. If you set up a login credential earlier use that to ssh into the raspberry pi or just use "pi" as username and "raspberry" as password.

	ssh pi@<ip address of RPI>  or  ssh <username>@<ip address of RPI>

 The first thing to do after logging is to update the RPI.
 
	sudo apt-get update && sudo apt-get upgrade -y

Enter the password if prompted and wait for the update and upgrade to finish. After the update has finished update once.

------------
#### 3. Hiding traces of Raspberry Pi:
Now we need to work on hiding the fact that it is a Raspberry Pi  so that it isn't obvious that it's a honeypot and looks more promising to a hacker.

We need to fake the MAC address to do that as it is given out during a network scan. To do that first, make a fake MAC Address which makes it look like another device but something that would be appropriate for the environment. Considering I'm putting it on my home network a NAS makes sense. I chose to make a Synology address but any NAS company address can be used like Asustor or QNAP. 

So, in this case, a Synology MAC address starts with "00:11:32", you can use a MAC address generator like [this website](https://miniwebtool.com/mac-address-generator/ "this website") to generate a random MAC address. 
Select the format to the one which looks like this "MM:MM:MM:SS:SS:SS" and enter 001132 in the prefix section, and make sure it's uppercase.

After that I used the tool macchanger to change the device's MAC address. Install it with the following command.

	sudo apt-get install macchanger

After that we need to make a system service to change the ethernet and wifi MAC address on startup. 
First use this command to identify the interface names to change mac for. 

	ifconfig

not down the wlan and eth named interfaces. After that create a systemd file using a note editor I use nano here. 

	sudo nano /etc/systemd/system/changemac@.service

Then paste the following script into the file. Make sure to change the MAC address to the one you generated.

	[Unit]
	Description=changes mac for %I
	Wants=network.target
	Before=network.target
	BindsTo=sys-subsystem-net-devices-%i.device
	After=sys-subsystem-net-devices-%i.device
		
	[Service]
	Type=oneshot
	ExecStart=/usr/bin/macchanger --mac=<mac address you created> %I
	RemainAfterExit=yes
		
	[Install]
	WantedBy=multi-user.target

After that enable the the systemd service with the following commands for both network interfaces.
Change the interface names to the ones you noted beforehand.

	sudo systemctl enable changemac@<ethernet interface>.service
	sudo systemctl enable changemac@<wlan interface>.service

Then once again reboot, this time the mac addresses of the interfaces so it will probably pick up another IP address so use the scanner method to find the IP and connect again.

------------

#### 4. Opencanary Installation :

Use the following commands to install the required dependecies, then install opencanary.

	sudo apt-get install python3-dev python3-pip python3-virtualenv python3-venv python3-scapy libssl-dev libpcap-dev libffi-dev samba pcapy-ng
	sudo pip3 install pcapy scapy
