# Raspberry Honey Pot

This is a project that I made to understand honeypots and how to make one myself. The project uses the opencanary daemon. Thanks to Bob McKay for his extensive guide on setting up an RPi Honeypot based on which this project is made, which can be found [here](https://bobmckay.com/i-t-support-networking/hardware/create-a-security-honey-pot-with-opencanary-and-a-raspberry-pi-3-updated-2021/ "here").

### Hardware and Software Used:
- Raspberry Pi 3B V1.2
- 16 GB Micro SD Card
- Ethernet Cable
- Raspbian OS Lite x64
- Opencanary


### Instructions:

------------
#### 1. Installation :
Download and flash Raspbian OS lite on the micro SD card. The selection of Raspbian lite is purely because it's being run on a Raspberry Pi so it'll be easier on the processing and won't require a GUI considering it's just a honeypot. I just used the Raspberry Pi imager software given out by the official Raspberry Pi community. 

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

After that, I used the tool Macchanger to change the device's MAC address. Install it with the following command.

	sudo apt-get install macchanger

After that, we need to make a system service to change the ethernet and wifi MAC address on startup. 
First, use this command to identify the interface names to change MAC for. 

	ifconfig

not down the WLAN and eth named interfaces. After that create a Systemd file using a note editor I use nano here. 

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

After that enable the systemd service with the following commands for both network interfaces.
Change the interface names to the ones you noted beforehand.

	sudo systemctl enable changemac@<ethernet interface>.service
	sudo systemctl enable changemac@<wlan interface>.service

Then once again reboot, this time the MAC addresses of the interfaces so it will probably pick up another IP address so use the scanner method to find the IP and connect again.

------------

#### 4. Opencanary Installation :

Use the following commands to install the required dependencies, then install Opencanary.

	sudo apt-get install python3-dev python3-pip python3-virtualenv python3-venv python3-scapy libssl-dev libpcap-dev libffi-dev samba pcapy-ng
	sudo pip3 install pcapy scapy

#### 5. Opencanary configuration :

run the following command to make an Opencanary config file.

	opencanaryd --copyconfig

After that, a message says "A sample config file is ready" After that run the following command to start the opencanary service.

	opencanaryd --start

This should start the opencanary service and should start showing something which shows like a log.

After that let's change the SSH to a more non-obvious port number so that we can access it for configuration and maintenance when needed. Open "/etc/ssh/sshd_config" with your choice text editor and change the line "Port 22" to something very high number like 65421 and remember it so that you can connect to it later. Save to file and reboot.

Rename the current Samba config to keep it as a backup just in case. 

	sudo mv /etc/samba/smb.conf /etc/samba/smb.conf_backup

Now create a new samba config which resembles a config file of a Windows fileserver made with your selected company's devices.

	sudo nano /etc/samba/smb.conf
------------

#### 6. Setting up notifications :

Now to get alerts from the honeypot so that it can be logged and monitored we need to add an email to add a sender email, application password, and a receiver email address. This can be done by adding the following config lines to the "/etc/opencanaryd/opencanary.conf" file.

	"SMTP": {
	"class": "logging.handlers.SMTPHandler",
	"mailhost": ["smtp.gmail.com", 587],
	"fromaddr": "johndoe@gmail.com",
	"toaddrs" : ["securityalerts@email.com"],
	"subject" : "OpenCanary Alert at home!",
	"credentials" : ["sender@gmail.com", "YOURAPPLICATIONPASSWORD"],
	"secure" : []
	}

If you want to get instructions on how to get the application password of a Gmail account you can follow the instructions by Google [here](https://support.google.com/mail/answer/185833?hl=en-GB "here").
Make sure you don't use your main email or any important email just set up a new account for this project.

After saving this check it by either logging into the honeypot or ping it. You should start receiving logs on the receiver email soon.
------------

#### 7. Final setups :

Now we need to make the Onpencanary service start on each startup automatically. For that, we make a systemd service file. using your preferred text editor or Touch make a file like the following then edit it. I am using Nano.

	sudo nano /etc/systemd/system/opencanary.service

Then add the following config to it.

	[Unit]
	Description=OpenCanary
	After=syslog.target
	After=network.target
	
	[Service]
	User=root
	Restart=always
	WorkingDirectory=/home/pi/opencanary
	ExecStart=/home/pi/opencanary/bin/opencanaryd --dev
	
	[Install]
	WantedBy=multi-user.target

After saving that file enable the service and check it with the following files.

	sudo systemctl enable opencanary.service
	sudo systemctl start opencanary.service
	systemctl status opencanary.service
------------

After all this sprinkle some files and artifacts in the honeypot to make the attacker interested in the pot for longer and record a pattern. I just added some basic files and images.

![Screenshot 2023-10-24 at 6 07 00 PM](https://github.com/shadow-dragon-2002/rpi_honeypot/assets/73079262/b9283708-ad55-452e-8e22-c692e0fd6f6f)


And by this, the Raspberry Pi honeypot should be fully set up. Now you can experiment with adding more services to it or opening other service ports etc. Just make sure not to make it obvious that it's a honeypot, it should be just like a normal server with some bare minimum vulnerabilities to make it look appealing.
