# Reference Implementation: How to Build an Environment Monitor Solution

## What it Does

This guide demonstrates how existing IoT solutions can be adapted to address more complex problems (e.g., solutions that require more sensor monitoring). The solution we present here, an Environment Monitor, incorporates additional hardware and extends the use of IoT software libraries (sensors and I/O).

The Environment Monitor solution described in this how-to guide is built using an UP Squared\* Grove* IoT Development Kit available from Seeed\* Studio. The solution runs on an UP Squared Grove IoT Development Kit (running Ubuntu\* Server) with Intel® System Studio to enable the sensors.

## How it Works

This solution is built around MRAA (I/O library) and UPM (sensor library) from the [UP Squared* Grove* IoT Development Kit](https://software.intel.com/en-us/iot/hardware/dev-kit), which powers the interface between the platform I/O and sensor data. In this case, the MRAA library provides an abstraction layer for the hardware to enable direct access to the on board I/O using Firmata. The UPM sensor library was developed on top of MRAA, and exposes a user-friendly API that will allow the user to capture sensor data with just a few lines of code. Data is then sent periodically to Amazon Web Services (AWS)* using MQTT.

## Requirements
### Hardware
- UP Squared Grove IoT Development Kit
- Sensors

### Software
- Ubuntu\* Server
- IoT Software Libraries ([MRAA](http://mraa.io) and [UPM](http://upm.mraa.io))
- Intel® System Studio
- Amazon Web Services (AWS)\* using MQTT

## Setup the System Hardware

This section describes how to set up all the required hardware for an Intelligent Vending Machine: UP Squared\* and Grove\* Sensors.

Setting up the UP Squared Grove IoT Development Kit for this solution consists of the following steps:

1. Connect a monitor via the HDMI\* or VGA port and a USB keyboard. These are required for OS deployment and can be removed after the Up Sqared has been connected to the network and a connection from the development environment has been established.

2. Plug in an Ethernet cable from your network’s router.

3. Plug in the power supply for the UP Squared but DO NOT press the power button yet. First connect the GrovePi and other hardware components and then power on the UP Squared.

### Sensor Setup

The sensors used for this project are listed in Table 1.

First, plug in the Grove\* Base Shield on top of the UP Squared board.

Three sensors with various functions relevant to monitoring the environment have been selected:
 * **Grove\* - Gas Sensor (MQ2)** measures the concentration of several gases (CO, CH4, propane, butane, alcohol vapors, hydrogen, liquefied petroleum gas) and is connected to analog pin 1 (A1). Can detect hazardous levels of gas concentration.
 * **Grove\* - Dust Sensor** will detect fine and coarse particulate matter in the surrounding air. Connect to digital pin 4 (D4).
 * **Grove\* - Temperature, Humidity & Barometer Sensor** is based on the Bosch\* BME280 chip and used to monitor temperature and humidity. It can be plugged in any of the connectors labeled I2C on the shield.

The **Grove\* Green LED** acts as an indicator LED to show whether the application is running or not, and is connected to digital pin 2 (D2).

Table 1. Bill of Materials

| Component | Details  | Pin Connection | Connection Type |
|-----------|----------|----------------|-----------------|
| [Grove\* - LED Sensor](https://www.seeedstudio.com/Grove-Green-LED-p-1144.html) | LED indicates status of the monitor | D2 | Digital |
| [Grove\* - Relay](https://www.seeedstudio.com/Grove-Relay-p-769.html) | Fan control | D8 |
| [Grove\* - Gas Sensor(MQ2)](https://www.seeedstudio.com/Grove-Gas-Sensor(MQ2)-p-937.html) | Gas sensor (CO, methane, smoke, etc.) | A1 | Analog |
| [Grove\* - Dust Sensor](https://www.seeedstudio.com/Grove-Dust-Sensor-p-1050.html) | Particulate matter sensor | D4 | Digital |
| [Grove\* - Temp&Humi&Barometer Sensor (BME280)](https://www.seeedstudio.com/Grove---Temp%26amp%3BHumi%26amp%3BBarometer-Sensor-(BME280)-p-2653.html) | Temperature, humidity, barometer sensor | I2C Bus | I2C |

**Note:** A green LED is used, but any color LED (red, blue, etc.) can be used as an indicator.

### Install and Configure the Required Software

This section gives instructions for installation of the operating system and connecting UP Squared to the Internet, installing required software libraries, and finally cloning the project sources from a GitHub\* repository.
	
### Installing the OS: Ubuntu\* Server

Follow the instruction in https://www.ubuntu.com/download/iot/up-squared-iot-grove-server to install Ubuntu on UP Squared, if not already installed.

### Connecting the UP Squared to the Internet

This section describes how to connect the UP Squared to your network, which will enable you to deploy and run the project from a different host on the same network (i.e. your laptop). Internet access is required in order to download the additional software libraries and the project code.

### Additional Configuration via the Terminal (Shell) on the UP Squared

#### Ethernet

 1. Once Ubuntu is installed, restart the UP Squared and login using your user.
 2. Type in the command `ifconfig` and find the interface named `enp3s0` in the list. In some cases, this might show up as `eth0` instead.
 Use the name displayed here for the following step.
 3. Open the network interface file using the command: `vim /etc/network/interfaces` and the following lines to it:

    ```
    auto enp3s0
    iface enp3s0 inet dhcp
    ```

 4. Save and exit the file and restart the network service using the following command: `/etc/init.d/networking restart`.
 5. If you are connecting to external networks via a proxy, you will have to set it up as well.

#### Wi-Fi\* (optional)

This is an optional step that only applies if a wireless card has been added to the UP Squared board.

 1. Install Network Manager using the command: `sudo apt install network-manager` and then install WPA supplicant using: `sudo apt install wpasupplicant`
 2. Once these are in place, check your Wi-Fi\* interface name using `ifconfig`. This examples uses `wlp2s0`. Now run the following commands:
    * Add the wifi interface to the interfaces file at: `/etc/network/interfaces` by adding the following lines:

        ```
        auto wlp2s0
        iface wlp2s0 inet dhcp
        ```

    * Now restart the networking service: `/etc/init.d/networking restart`
    * Now run: `nmcli networking` and `nmcli n connectivity` and `nmcli radio`. These commands tell you whether the network is actually enabled or not. If either of them says "not enabled" then you’ll have to enable in order to reach full connectivity. For enabling the radio, use the following command: `nmcli radio wifi on`
    * Now check the connection status: `nmcli device status`
    * If the Wi-Fi interface shows up as "unmanaged", then you might have to troubleshoot that.
    * Now to view and add Wi-Fi connections:

        ```
        nmcli d wifi rescan
        nmcli d wifi
        nmcli c add type wifi con-name [network-name] ifname [interface-name] ssid [network-ssid]
        ```

    * Now, running `nmcli c` should show you the connection you have tried to connect to. In case you are trying to connect to an enterprise network, you might have to make changes to `/etc/NetworkManager/system-connections/[network-name]`
    * Now bring up the connection and the network interfaces:

        ```
        nmcli con up [network-name]
        ifdown wlp2s0
        ifup wlp2s0
        ```

#### Installing the MRAA and UPM libraries on UP Squared

In order to put UPM and MRAA on your system, you can just use the MRAA:PPA to update the libraries. The instructions are as follows:

```
sudo add-apt-repository ppa:mraa/mraa
sudo apt-get update
sudo apt-get install libupm-dev libupm-java python-upm python3-upm node-upm upm-examples
```

You can also build from source:

MRAA instructions: https://github.com/intel-iot-devkit/mraa/blob/master/docs/building.md
UPM instructions: https://github.com/intel-iot-devkit/upm/blob/master/docs/building.md

Install node and npm.

**Note:**
Now we can disconnect the monitor and keyboard from UP Squared, as we can remotely access the UP Squared using its IP address. This requires that the Laptop/Desktop to be on the same network as UP Squared.

## Setup and Connect to the Cloud Data Store

### AWS\* IOT Core

This solution was designed to send sensor data using the MQTT protocol to AWS. In order to connect the application to the online data store, first setup and create an account.

1.	To set up and create an account: https://github.com/intel-iot-devkit/intel-iot-examples-mqtt/blob/master/aws-mqtt.md

2.	Download the Certificates, Private Key and Root CA from AWS.

3.	We have to copy the certificates, private key and Root CA to the UP Squared board. For this, use this command:
```
scp -r <path where certificate is present>  upsquared:##.###.##.##:/home/upsquared/
scp -r <path where privatekey is present>  upsquared:##.###.##.##:/home/upsquared/
scp -r <path where RootCA is present>  upsquared:##.###.##.##:/home/upsquared/	
```

### Connection to Localhost

1. Install mosquitto with websocket connection on the UP Squared board.(http://goochgooch.co.uk/2014/08/01/building-mosquitto-1-4/)

2. Make this change to /etc/mosquitto/mosquitto.conf:
	listener 9001
	protocol websockets

3. Start the mosquitto on upsquared by sudo mosquitto -v -c /etc/mosquitto/mosquitto.conf.

### Add the Solution to Intel® System Studio

This section explains how to add the solution to Intel® System Studio, including creating a new project and populating it with the files needed for it to build and run.

1. Open Intel® System Studio. It will start by asking for a workspace directory. Choose one and then click **OK**.
2. From the Intel® System Studio , select File > New > Project, 
   then choose "Create a C/C++ project for building in a container and running on Linux" under Application Development in New Project. 
3. Give the project name such as "Environment" and in the examples choose the "Display Temperature on LCD" under C++ > Grove Starter Kit and then click **Next**.
4. Delete all the files present in src folder.
5. Right click on the src folder and select New > Header file. Type the name of the header file grovekit.hpp. In the same way, create a mqtt.h header file. 
6. Right click on the src folder and select New > Source File. Type the name of the Source file mqtt.cpp and click on finish. In the same, way create another source file and name it:  envmonitor.cpp  
7. Copy all the files present in the src folder to the respective files.   
8. Next, right click on the project name and follow the sequence: Properties > C/C++ Build > Settings > GNU 64-bit G++ Linker > Libraries and then Add libraies:  mraa, upm-bmp280, upm-led, paho-mqtt3cs, upm-gas, upm-ppd42ns, upmc-utilities. This can be done by clicking on the small green '+' icon on the top right side of the libraries view

9. In order to run this project, connect to the UP Squared first using the IP address already provided. Follow the instructions in "https://software.intel.com/en-us/developing-projects-with-intel-system-studio-c-creating-an-ssh-connection" to connect to UP Squared  as root. This is required to use pins on board.

10. Go to Run configurations, and in the Commands to execute before application field, type the following:

     ```
     chmod 755 /tmp/Environment; 
     export MQTT_SERVER="ssl://<Your host name>:8883"; 
     export MQTT_CLIENTID="<Your Client ID>"; //for testing type it as environment
     export MQTT_CERT="/home/upsquared/<your certificate name>"; 
     export MQTT_KEY="/home/upsquared/<your privatekey name>"; 
     export MQTT_CA="/home/upsquared/<your ROOTCA name>"; 
     export MQTT_TOPIC="environment"
     ```

  Click the **Apply** button to save these settings.
  Click the **Run** button to run the code on your board.
  
## Create the Development and Runtime Environment on Desktop/Laptop

- Download Intel® System Studio from https://software.intel.com/en-us/system-studio and install.


## More Information

 * [IoT Reference Implementation: Making of an Evironment Monitor Solution](https://software.intel.com/en-us/articles/iot-reference-implementation-making-of-an-environment-monitor-solution)
 * [Code Sample - Air Quality Sensor In Java\*](https://software.intel.com/en-us/articles/air-quality-sensor-in-java)
 * [Code Sample - Air quality sensor in C++](https://software.intel.com/en-us/articles/how-to-intel-iot-code-samples-air-quality-sensor-in-cpp)
