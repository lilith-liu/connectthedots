This document explains how to set up a Raspberry Pi as a very simple gateway to read data from sensors connected over Serial, USB, or REST interfaces, and to send that data without change over AMQP to Azure. 
It assumes that you have the right tools installed and that you have cloned or downloaded the ConnectTheDots.io project on your machine.

This configuration creates is very basic - it does not provide any functions such as device management, authentication, or access control. As such, it should be viewed as part of a Proof of Concept for building an IoT solution, not an integral part of a secure enterprise infrastructure. A recommended configuration will be published shortly.


##Hardware requirements ##
See [Hardware](Hardware.md) file in this folder.


##Prerequisites ##

To build the project you will need Visual Studio 2013 [Community Edition](http://www.visualstudio.com/downloads/download-visual-studio-vs) or above. You will also need wired Internet access for the device. (Configuration details for wireless connectivity are not provided.)

## Configure the Raspberry Pi ##

* Connect the Raspberry Pi to a power supply, keyboard, mouse, monitor, and Ethernet cable (or Wi-Fi dongle) with an Internet connection.
* Get a Raspbian NOOBS SD Card or download a NOOBS image as per the instructions on [http://www.raspberrypi.org/downloads/](http://www.raspberrypi.org/downloads/)
* Boot the NOOBS SD Card and choose Raspbian (see [http://www.raspberrypi.org/help/noobs-setup/](http://www.raspberrypi.org/help/noobs-setup/) for more information).
* Connect to the Raspberry Pi from your laptop, either via a USB-Serial adapter or via the network via SSH (enable once as per these instructions while booting via a monitor on HDMI and a USB keyboard). To connect using SSH:
    * For Windows, download PuTTY and PSCP from [here](http://www.putty.org/).
    * Connect to the Pi using the IP address of the Pi.
* Once you have connected to the Pi, install on it the Mono runtime and root certs required for a secure SSL connection to Azure:
    * Run the following from a shell (i.e. via SSH)
    

                  sudo apt-get update 
				  sudo apt-get upgrade 
                  sudo apt-get install mono-complete
                  sudo mozroots --import --ask-remove
				  sudo apt-get -y install python libusb-1.0
				  sudo apt-get -y install python-pip
				  sudo pip install pyusb

    * This will install the regular build of mono on the Pi, which may have some limitations. If you want to run the latest build of mono instead, which may work better for you, run the following commands before running the above. Note that the latest build below may not work with all the functions in the Connect The Dots project for your specific model of Pi, so using it is for the advanced user only. 
    
                  sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF 
                  echo "deb http://download.mono-project.com/repo/debian wheezy main" | sudo tee /etc/apt/sources.list.d/mono-xamarin.list 



* Open the Devices\Gateways\GatewayService\GatewayService.sln solution in Visual Studio
* In Visual Studio, update \GatewayService\WindowsService\App.config with any one of the four amqp address strings returned by AzurePrep.exe, i.e. amqps://D1:xxxxxxxx@yyyyyyyy.servicebus.windows.net, and the 
name that you assigned to your gateway. It is important that the key is being url-encoded, meaning all special characters should be replaced by their ASCII code (e.g. "=" should be replaced by "%3D". You can use tools like [http://meyerweb.com/eric/tools/dencoder/](http://meyerweb.com/eric/tools/dencoder/) to url-encode the key. Four strings you can use are in the file created on your desktop by the AzurePrep.exe utility used earlier. Copy one of those strings and replace the relevant line in App.config:

	Before:
    
 
		<AMQPServiceConfig
		AMQPSAddress="amqps://[keyname]:[key]@[namespace].servicebus.windows.net"
		EventHubName="ehdevices"
		EventHubMessageSubject="gtsv"
		EventHubDeviceId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
		EventHubDeviceDisplayName="SensorGatewayService"/>

	After:
 
		<AMQPServiceConfig
		AMQPSAddress="amqps://D1:iKwblb9AwHD2GPzu1TRF5Jz76QiSynsjbuWbdxsIi98%3D@sstest20-ns.servicebus.windows.net"
		EventHubName="ehdevices"
		EventHubMessageSubject="gtsv"
		EventHubDeviceId="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
		EventHubDeviceDisplayName="SensorGatewayService"/>

	You can also replace the EventHubDeviceId with an ID of your choice.

* Use  the file \Scripts\RaspberryPi\deploy.cmd to copy all requisite files from your computer to the Pi. To use the .CMD file, you will need to 
        
    * Update the IP address
    * Change the Putty and Project directories in the .CMD file as necessary
    * Change Configuration to Release or Debug to reflect whether you built the solution to Debug or Release. 
    
* On the Raspberry Pi, modify /etc/rc.local with nano:
    
		Sudo nano /etc/rc.local
 
* When you are in the nano editor, edit rc.local to the following:
    
		# Print the IP address
		_IP=$(hostname -I) || true
		if [ "$_IP" ]; then
		  printf "My IP address is %s\n" "$_IP"
		fi

		export GW_ACCOUNT_HOME=/home/pi
		export GW_HOME=$GW_ACCOUNT_HOME/ctdgtwy
		if [ ! -d $GW_HOME/logs ]
		  then
		   sudo mkdir $GW_HOME/logs
		fi
		sudo echo "$(date) booting..." >> $GW_HOME/logs/booting.log
		$(cd $GW_HOME/ ; sh deploy_and_start_ctd_on_boot.sh) &
		exit 0


* To exit the nano editor use Ctrl-x, and press Y to save the changes. To have the new settings take effect, reboot the Raspberry Pi by cycling the power or by issuing the command 
    
		Sudo reboot


* At this point your Raspberry Pi is ready to be used as a Gateway for sensor devices connected over USB to send their data to Azure Event Hubs.

## Wifi ##

If you want to use Wifi instead of Ethernet in your configuration, you can follow [these instructions](WiFi-Configuration.md)

## Log file ##

If you look at the RaspberryPiGateway.Log file on the Raspberry, you will see the same JSON formatted data being read from the serial port of the Raspberry as you saw being sent from the serial port of the Arduino:
    
	Parsed data from serial port as: ":0,"windgustmph_10m":0.0,"windgustdir_10m":0,"hmdt":44.9,"temp":73.1,"tempH":23.6,"rainin":0.0,"dailyrainin":0.0,"prss":100432.75,"batt":4.39,"lght":0.74}


Shortly later in the log file you will see “Message sent” meaning that same data was sent over AMQPS to Azure.