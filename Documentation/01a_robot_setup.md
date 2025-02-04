# **ROBOTsetup**

The objectives are:

- Configure the raspberrypi
- Install and bringup the robot model

We will use a raspberrypi4 board with ubuntu20 ROS Noetic and a hotspot wifi connection.

## **1. Configure the raspberrypi**

We will use a raspberrypi4 board with ubuntu20 ROS Noetic intalled.

There are 2 different options for this installation:

- Install ubuntu20 server image and ROS Noetic
- Install Raspberrypi Desktop and ROS Noetic with Docker


### **1.1. Install Ubuntu20 server image and ROS Noetic**

Follow the steps in order to properly install the Ubuntu20 on Raspberrypi:

- Run Raspberry Pi Imager (https://www.raspberrypi.org/software/)
  - select Device: Raspberrypi4
  - select OS Ubuntu --> Ubuntu 20 server 64bits to the SD card
  - Select the configurations:
    - Name: rUBot_X
    - User: ubuntu
    - Pass: ubuntu1234
    - LAN config: wifi you want to connect (i.e. rUBotics)
    - Regional settings: ES
    - Services: activate ssh
- Insert the SD in a RBPi board and connect it to screen and ethernet cable to the router
- power the raspberrypi4 and login:
  - login: ubuntu
  - password: ubuntu1234
- update the OS:
  ````shell
  sudo apt update
  sudo apt upgrade
  sudo reboot
  ````

If you want to create other **wifi connections and a Hotspot**.

- Install ubuntu Desktop:
  ````shell
  sudo apt install ubuntu-desktop
  sudo reboot
  ````
  > this will take 10 minutes and you have to select "gdm3" during process

- In Ubuntu Desktop graphical interface, create a **Hotspot** following instructions in: https://www.debugpoint.com/2020/04/how-to-create-wifi-hotspot-in-ubuntu-20-04-lts/

  - Select Wi-Fi settings
  - Select "Turn On Wi-Fi Hotspot"
    >Carefull!:
    >- If "Turn On Wi-Fi Hotspot" is disabled select another setting (i.e. Bluetooth) and come back to Wi-Fi setting
  - Choose a SSID corresponding to your robot name
    - Name: rubot_01
    - Pass: rUBot_Mec
  - To change the Hotspot settings (name or password):
    ```shell
    sudo nm-connection-editor
    ```
- To create other **wifi networks**, use the Desktop graphical interface to easy select the available wifi and configure to start automatically with priorities (higher number corresponds to higher priority)
  >Review the default wifi network defined on the Ubuntu20 image creation with "Imager"
- If you want to use the robot Hotspot within Nomachine remote desktop, you will have to install Nomachine in raspberypi: https://downloads.nomachine.com/es/download/?id=106&distro=Raspberry&hw=Pi4

### **1.2. Install ROS Noetic Desktop**

Follow the instructions on: http://wiki.ros.org/noetic/Installation/Ubuntu
> Is recommended to update and upgrade first:
```shell
sudo apt update
sudo apt upgrade
```

### **1.3. Install Arduino**

Interesting review: https://www.clearpathrobotics.com/assets/guides/noetic/ros/Driving%20Husky%20with%20ROSSerial.html

- You need to install it with snap to be sure you have all dependencies. You can install Arduino IDE on Ubuntu and add your user to the dialout and tty groups:
  ````shell
  sudo snap install arduino
  sudo usermod -a -G tty ubuntu
  sudo usermod -a -G dialout ubuntu
  sudo reboot
  ````
- Install ROS Packages for Arduino:
  ````shell
  sudo apt install ros-noetic-rosserial
  sudo apt install ros-noetic-rosserial-arduino
  ````
- Install ROS Libraries:

  Go to the Arduino Sketchbook location and install libraries. With Arduino core you will surelly have to create the directory. If you have arduino programed, you will not need the ros_libraries:

  ````shell
  cd /home/ubuntu/snap/arduino/current/libraries/
  rosrun rosserial_arduino make_libraries.py .
  ````
  
  Restart your Arduino IDE and you should see the ros_lib part of your libraries!

### **1.4. Install rplidar**
You need to install the package: http://wiki.ros.org/rplidar

```shell
sudo apt install ros-noetic-rplidar-ros
```

### **1.5. Install usb-cam**
You need to install the package: https://wiki.ros.org/usb_cam

```shell
sudo apt install ros-noetic-usb-cam
```

### **1.6. Create a service to Bringup the robot on boot**

You need to create a service rubot_bringup.service in the /etc/systemd/system/ directory.
````shell
sudo nano /etc/systemd/system/rubot_bringup.service
````
- Add the Service Configuration:
````shell
[Unit]
Description=ROS Bringup Service for Rubot Mecanum
After=network.target
Requires=network-online.target

[Service]
Type=simple
User=ubuntu  # Replace with your actual username
WorkingDirectory=/home/ubuntu/rUBot_mecanum_ws  # Replace with your catkin workspace path
ExecStart=/bin/bash -c "source /opt/ros/noetic/setup.bash && source devel/setup.bash && roslaunch rubot_mecanum_description rubot_bringup_hw_arduino.launch"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
````
- Enable and Start the Service
````shell
sudo systemctl enable rubot_bringup.service
sudo systemctl start rubot_bringup.service
````
- Check the Service Status:
````shell
sudo systemctl status rubot_bringup.service
````
- To delete the service:
````shell
sudo systemctl stop rubot_bringup
sudo systemctl disable rubot_bringup
sudo rm /etc/systemd/system/rubot_bringup.service
sudo systemctl daemon-reload
sudo systemctl is-enabled rubot_bringup
````

**rc.local** method:
- Create the file rc.local
````shell
sudo nano /etc/rc.local
````
- update the file contents:
````shell
#!/bin/bash
/bin/bash /home/ubuntu/rubot_bringup.sh &
exit 0

````
- make file executable:
````shell
sudo chmod +x /etc/rc.local
````
- Create the rubot_bringup.sh execution file:
````shell
#!/bin/bash
source /opt/ros/noetic/setup.bash
source /home/ubuntu/rUBot_mecanum_ws/devel/setup.bash
roslaunch rubot_mecanum_description rubot_bringup_hw_arduino.launch
````
- Create the rc-local.service
````shell
sudo nano /etc/systemd/system/rc-local.service
````
- with the contents:
````shell
[Unit]
Description=/etc/rc.local Compatibility
After=network.target

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
````
- you need to enable it and start it to ensure it runs on boot (verify the service status):
````shell
sudo systemctl enable rc-local
sudo systemctl start rc-local
sudo systemctl status rc-local
````
- Verify the ros processes
````shell
ps aux | grep ros
journalctl -u rc-local
````
- Kill process if necessary:
````shell
sudo lsof -i :11311
ps aux | grep ros
sudo kill -9 12977
````


## **2. Install Raspberrypi Desktop and ROS Noetic with Docker**

To create a fast and robust image of ROS Noetic for our robot, an improved method is to use Docker.

### **2.1. Install Raspberrypi Desktop**

- Run Raspberry Pi Imager (https://www.raspberrypi.org/software/)
  - select Device: Raspberrypi4
  - select OS: RaspberryPi OS (64Bits) to the SD card
  - Select the configurations:
    - Name: rUBot_XX
    - User: ubuntu
    - Pass: ubuntu1234
    - LAN config: wifi you want to connect (i.e. rUBotics)
    - Regional settings: ES
    - Services: activate ssh
- Insert the SD in a RBPi board and connect an ethernet cable to the router
- power the raspberrypi4 and login:
  - login: ubuntu
  - password: ubuntu1234
- update the OS:
  ````shell
  sudo apt update
  sudo apt upgrade
  sudo reboot
  ````

  ### **2.2. Install VNC connections**

- Connect to the raspberrypi with ssh and activate **VNC connections**:
  - type: sudo raspi-config
  - Navigate to: Interface Options > VNC > Select Yes to enable it.
  - sudo reboot
- In your PC install Remote desktop on RealVNC Viewer: https://www.realvnc.com/es/connect/download/viewer

- If you want to connect to another network, you have to be connected first manually to the different networks to enable raspberrypi to connect to on reboot
- reboot and it will be connected to the first network available

### **2.3. Using VScode remote explorer**

You can install the Extension "Remote explorer" on VScode:

- Open VScode and connect remotelly to the Raspberrypi with ssh -X ubuntu@192.168.xxx.xxx
- If you can not connect to the raspberrypi, perhaps you have to regenerate permissions (replace IP-raspberrypi by 192.168.xx.xx):
  ````shell
  ssh-keygen -R IP-raspberrypi
  ````

### **2.4. Install Docker**

In raspberrypi, add Docker’s official repository for Ubuntu
````shell
sudo apt update
sudo apt upgrade
# install Docker automatically in function of the Raspbian version installed
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
# Start Docker service
sudo systemctl start docker
# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl enable containerd.service
# Add your user to the Docker group (to avoid using sudo for Docker commands)
sudo usermod -aG docker $USER
# Reboot to apply changes (especially for the user group change)
sudo reboot
````

**Create a custom Docker image**

We first create a Docker folder where we:
- Copy the rUBot_mecanum_ws
- Install the Docker extension
- Create a Dockerfile file to specify the image characteristics:
    - Image from arm64v8/ros:noetic
    - install all needed ros packages 
    - install X11 libraries to allow graphical applications
    - Optionally: Copy the rUBot_mecanum_ws to the /root/ folder and run "roslaunch rubot_mecanum_description rubot_bringup_hw_arduino.launch"
- Create the Docker Image (ros-noetic-rubot-mecanum) from the Dockerfile
````shell
cd /home/ubuntu/Desktop/Docker
sudo docker build -t ros-noetic-rubot-mecanum .
or
sudo docker build -t ros-noetic-rubot-mecanum:v2 -f Dockerfile2 .
````
**Ceate and Start Docker Container automatically**

Docker Compose is the best way to automate and manage container startup, as it allows you to easily specify the configuration for starting your container

- Create first the DISPLAY and RUBOT environment variables in ".env" file:
  ````shell
  export DISPLAY=192.168.88.72
  export RUBOT=192.168.88.93
  ````
  > Change the IP in function of the choosen network
- Create a file "docker-compose.yaml" with:
  ````shell
  services:
  ros-noetic-rubot-mecanum:
    image: ros-noetic-rubot-mecanum:v2  # La imatge personalitzada
    privileged: true
    network_mode: host # Comparteix la xarxa amb l'amfitrió
    container_name: container-ros-noetic-rubot-mecanum2
    environment:
      - DISPLAY=${DISPLAY}
      - ROS_MASTER_URI=http://localhost:11311
      - ROS_IP=localhost
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - /dev:/dev
    entrypoint: /root/entrypoint2.sh
    command: ["roslaunch", "rubot_mecanum_description", "rubot_bringup_hw_arduino.launch"]
    restart: always  # Reengega el contenidor si falla
  ````

- Start the container manually the first with Docker Compose to test if works properly
  ````shell
  export DISPLAY=192.168.88.72:0
  docker compose up
  or
  docker compose -f docker-compose2.yaml up -d
  ````
  > You can create a .env folder to store the Environment variables

- Close remote connection
- Verify on reboot

  > You have NOT to remove the container to ensure it starts automatically on reboot.

This start the container and next time when you connect the raspberrypi, this container will be executed automatically.
