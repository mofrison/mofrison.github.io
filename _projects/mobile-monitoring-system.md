---
title: Mobile monitoring system
repository: mobile-monitoring-system
preview: ./assets/images/projects/mobile-monitoring-system/preview.png
excerpt:  Identification of sensors during the crawl, receiving information from the Control center about the values of their readings, visualizing the readings from the sensor on the screen of a mobile device, sending a message to the Control center that the object was examined
date: 27-06-2019
categories: projects
tags: [java, android, studies]
layout: project
lang: En

QR-codes:
  - path: ./assets/images/projects/mobile-monitoring-system/Company_Engine-room_oil-pressure.gif
    title: Pic.5. Oil pressure
  - path: ./assets/images/projects/mobile-monitoring-system/Company_Engine-room_revs.gif
    title: Pic.6. Number of revolutions
  - path: ./assets/images/projects/mobile-monitoring-system/Company_Foundry-Shop_temperature.gif
    title: Pic.7. Temperature
  - path: ./assets/images/projects/mobile-monitoring-system/Company_Assembly-line_speed.gif
    title: Pic.8. Assembly-line speed
---

## 1. Project target

Creation of a software system for automating control over the conduct of rounds at enterprises, improving the quality of their implementation, freeing personnel from routine operations to fill in the logs of the crawl, by identifying sensors during the crawl, obtaining information from the Control center about the values of their readings, visualizing the readings from the sensor on the screen of a mobile device, sending a message to the Control center that the object was inspected.

### 1.2. Scope of application of the development
The application is designed to work in enterprises with a system for centralized receipt of technological information from its infrastructure objects.

## 2. Functional requirements

* The identification of sensors in the implementation of inspection
* Implementation of connection to the server with data on sensor readings;
* Organization of automatic receipt of data on the readings of identified sensors;
* Storing the information received from the sensors in the device memory;
* Display of the information received from the sensors in the form of a list of sensors and the values of their readings.
* Allowing the user to selectively delete records or completely clear the app of sensor data.
* Implementation of automatic sending of data on the completion of the inspection, to maintain labor discipline.
* Implementation of an intuitive user interface. The following features should be implemented in the interface:
    * Authentication;
    * Configure the connection to the server;
    * Sensor identification;
    * Visualization of saved readings;

### 2.1. Input data
* The address of the server on the network(to configure the connection);
* User name and password(for logging in);
* The name of the sensor, presented in an easy-to-enter format(to get the sensor readings);

### 2.2. Output data
* Sensor readings on the mobile device screen.

### 2.3. Identification of sensors in the system
Since there can be a huge number of sensors, and their types and purpose will be repeated, it is advisable to structure them in the form of directories. The names of nested directories should be separated by a «slash» ( / ), the last one will be the name of the sensor, for example: 
`System_name / subsystem_name / sensor_name` - this will be the full name of the sensor. The full name of the sensor will be used to get information from it. 
In order to quickly and accurately transmit the sensor name to the app, you can use Bluetooth beacons and NFC tags by encoding it into the appropriate signal.

As part of my thesis work, I replaced these two ways of transmitting data with reading QR codes. This function works in a similar way, only it uses a smartphone camera instead of an NFC chip or Bluetooth module. This makes the system even more accessible, since you do not need to use additional equipment other than the printer. However, it has one significant drawback: an unscrupulous worker can copy the image and not go anywhere.

### 2.4. Hardware and software requirements
Mobile phone-a smartphone with support for network data transmission Wi-fi, Bluetooth, NFC, equipped with a digital camera, running the Android 8.1 and higher operating system.

### 2.5. Network protocol for transmitting data to the device
Devices use various industrial protocols to communicate with each other, and MQTT is one of the most popular protocols for this purpose. MQTT or Message Queue Telemetry Transport is a lightweight, compact, and open data exchange protocol designed to transfer data to remote locations where a small code size is required and there are bandwidth limitations. The above advantages allow it to be used in M2M (Machine-to-Machine Interaction) and IIoT (Industrial Internet of Things) systems.

## 3. System design

Based on the analysis of the requirements for the system, its structural diagram was compiled:

![Pic.1. Structural diagram](/assets/images/projects/mobile-monitoring-system/structural-diagram.png "Pic.1. Structural diagram")

### 3.1. The algorithm of the application
In the work of the application, there are four main stages:
* Authentication and configuration
* Interaction with the server
* QR code detection
* Working with the database

All the steps are presented in the block diagram:

![Pic.2. Block diagram](/assets/images/projects/mobile-monitoring-system/block-diagram.gif "Pic.2. Block diagram")

### 3.2. Application Structure

Module              | Implemented a function
--------------------|-----------------------
MQTT                | Connecting to the broker, authentication sending messages with a report on the identification of the sensor, receiving messages with the readings of the identified sensors
QRDecoder.Camera    | Controlling the camera and getting an image from it
QRDecoder.Barcode   | Detection and decryption of QR codes received in the form of camera images
Database            | Interaction with the SQLite database
UI                  | Implementation of the user interface

![Pic.3. Class diagram](/assets/images/projects/mobile-monitoring-system/class-diagram.jpg "Pic.3. Class diagram")

### 3.3. Storage of information
The information stored in the app is divided into two groups:
* Camera settings(backlight, autofocus) and connection settings
* Sensor readings

The first group is characterized by a small but constant amount of data. Therefore, it will use SharedPreferences as its storage-a permanent storage on the Android platform used by applications to store their settings. The following fields will be created in the SharedPreferences file:
```xml
<map>
    <string name="Ключ">Значение</string>
</map>
```

The keys and parameter values will be set at the start of the application and stored in a private storage. You plan to use the following keys:
* _autoFocus_ — responsible for the use of autofocus, takes the values: 

    **ON** — autofocus is activated

    **OFF** — autofocus is deactivated

* _useFlash_ — it is responsible for turning the camera's backlight on / off, and takes the following values:

    **ON** — the camera light is on

    **OFF** — the camera light is off

* _url_address_ — corresponds to the value of the server url 
* _port_number_ — corresponds to the value of the port number

The second group — sensor readings-is very dynamic during operation and can both increase the amount of data used in the application and reduce it. The application uses an SQLite database to store information about sensor readings.

The application database will consist of three tables:
* _types_ — contains the main types of sensors
* _sensors_ — contains the unique names of the sensors
* _readings_ — stores sensor readings

![Pic.4. Application database](/assets/images/projects/mobile-monitoring-system/application-database.png "Pic.4. Application database")

In the types table, in addition to the id field, which is the primary key, there are four other fields:
* _type_neme_ — текстовое поле с названием типа датчиков
* _unit_of_measure_ — текстовое поле с названием единиц измерения
* _upper_limit_ — информация о верхней границеизмерений, тип real
* _lover_limit_ — информация о нижней границеизмерений, тип real

В таблице sensors, помимо поля id, которое является первичным ключом, имеются три поля:
* _sensor_name_ — a text field with the name of the sensor type
* _unit_of_measure_ — foreign key, the corresponding id from the types table is required to match the sensor and the units of measurement corresponding to this sensor type 
* _detection_time_ — the time of detection and recognition of the sensor, it is necessary to sort the data when output in the INT(long).

The third table reads also contains the id field(primary key), and three fields:
* _sensor_ — foreign key, corresponding to the id from the sensors table
* _value_ — stores the value received from the server
* _time_ — contains the time value since the last update of the value

## 4 Realization

## 4.1 Mqtt client
I used the PahoM QT client and the Android service provided by Eclipse.
How to add it to the project and how to interact with it is viewed in the article [MQTT-client for Android]({{site. url}}/cases/mqtt-client).

### 4.2 Graphical user interface
Four functions are expected for direct user interaction: authentication, setting up a connection to the server, identifying sensors and visualizing their readings. Based on this, the navigation map of the application was compiled:

![Pic.9. Navigation map](/assets/images/projects/mobile-monitoring-system/navigation-map.png "Pic.9. Navigation map")

The authentication screen contains text fields for entering the user name and password, which are used to authenticate the user in the system, the _«Sign in»_ button, which attempts to connect to the server, and a virtual keyboard for entering values in the text fields. Also in the top menu there is a button to go to the settings screen, designed in the form of a gear icon.
If after clicking on the _«Sign in»_ button, the connection to the server is not established, an error message will be displayed on the screen with a suggestion to check the connection settings. To set up the connection, click on the gear icon in the corner of the screen. This will take you to the settings screen.

There are also two text fields on the settings screen:
* A field for entering the URL.
* Field for entering the port number.

The virtual keyboard depends on the active text field. If the URL field is active, a full keyboard with basic alphanumeric elements is displayed. If the port number field is active, only the numeric keypad is displayed.
The following items are activated in the top menu:
* _«Back arrow»_ — implements a return to the previous screen.
* _«Lighting»_ — activates the backlight using the camera (if supported by the device).
* _«Autofocus»_ — activates the camera's autofocus mode (if supported by the device).
* _«Save»_ — saves the entered settings.

The main screen of the app combines a sensor identification area and a list.
To identify the sensor, an image from the camera is used, which the user directs to the _QR-code_. The list of identified sensors is displayed below, in reverse order of the identification time. Each item in the list contains the name of the sensor and its readings.

At the bottom of the screen there is a horizontal menu, which consists of the following items:
* _«Relogin»_ - go to the authentication screen
* _«Settings»_ - go to the settings screen
* _«Clear all»_ - delete all records and disable requests

In addition, special movements allow you to adjust the zoom on the camera, and a long press on the list item opens the context menu, through which you can delete this item.

## 5. Testing

To check the health of the application, you will need:
* A computer with the ability to connect to a network and a network OS;
* WiFi Router;
* MQTT-broker;
* Script for modeling the generation of sensor readings;
* QR codes with sensor names encrypted in them.

### 5.1. Installing the MQTT Server
To organize the transmission of messages over the network, you need to install _Mosquitto_ – this is a popular **MQTT** server (or broker). It is easy to install and configure and is actively supported by the developer community.
Installing the _Mosquitto_ using the console command:

    sudo apt-get install mosquitto-clients

By default, the Mosquito service starts immediately after installation.
To set up a password in the console, enter:

    sudo mosquitto_passwd -c /etc/mosquitto/passwd <username>

Opening the configuration file:

    sudo nano /etc/mosquitto/mosquitto.conf

In the file that opens, you must specify the path to the file with the user name and password hash:

    allow_anonymous false
    password_file /etc/mosquitto/mosquitto.pwd

After saving the file, you need to restart the server:

    sudo systemctl restart mosquitto

### 5.2. Script for generating test values
To simulate the system operation, a script was written in the **bash** language. In an infinite loop, the script generates random values and sends them to the server using the `mosquito_pub` command, which redirects these messages to clients who have subscribed to the corresponding topics.
```bash
#!/bin/bash
echo "Restart the server"
sudo systemctl stop mosquitto
sleep 0.5
sudo systemctl start mosquitto
sleep 0.5

echo "Getting the server address"
ipm="$(ip -4 addr show scope global | awk '$1 ~ /^inet/ {print $2}')"
ip="$(echo $ipm | awk -F"/"'{print $1}')"
echo "ip = "$ip

echo "Starting the value generation cycle"
echo "-----------------------------------"
n=32
i=0

while [ 1 ]
do
    let "number = ($RANDOM % 100)"
    number=0.7$number
    number+="Bar"
    mosquitto_pub -h $ip -t "Company/Engine-room/oil-pressure" -m $number -u "8host" -P "1234"

    let "number = ($RANDOM % 10 + 1495)"
    number+="об/c"
    mosquitto_pub -h $ip -t "Company/Engine-room/revs" -m $number -u "8host" -P "1234"

    let "number = ($RANDOM % 10)"
    number=34.3$number
    number+="С"
    mosquitto_pub -h $ip -t "Company/Foundry-Shop/temperature" -m $number -u "8host" -P "1234"

    let "number = ($RANDOM % 10)"
    number=2.5$number\
    number+="м/с"
    mosquitto_pub -h $ip -t "Company/Assembly-line/speed" -m $number -u "8host" -P "1234"
done
```
### 5.3. QR-codes
To check the functionality, the corresponding QR-codes were generated:
{% include image-gallery.html collection = page.QR-codes number = "four" %}

_Thank you for your interest! :)_
