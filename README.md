## Configuring a USB Hasp server based on Ubuntu Server 16.04 LTS (
a) ###### !ACHTUNG! Ubuntu Server version 16.04 LTS x86_64 was used with the linux kernel version-image-4.4.0-210- generic

## 1. Preparation 
###### 1.1. Installing the necessary software
```sh
sudo apt update
sudo apt install gcc g++ make libjansson-dev libusb-dev libc6-i386 libssl-dev git
```

###### 1.2. Installing Kernel Components
```sh
sudo apt update
sudo apt install linux-headers-`uname -r`
sudo apt install linux-tools-lts-xenial
```

###### 1.3. Cloning this repository
```sh
cd /usr/src
git clone https://github.com/rusishsoft/VH_Act.git
cd ./VH_Act
```

###### 1.4. Installing HASP Daemod (haspd)
###### !ACHTUNG No. 2! Despite the architecture name in the package name, haspd is a 32-bit component

```sh
cd ./haspd
sudo dpkg -i haspd_7.90-eter2ubuntu_amd64.deb
sudo dpkg -i haspd-modules_7.90-eter2ubuntu_amd64.deb
cd ../
```

## 2. Build from the source code of the kernel components and install them
###### 2.1. VHCI HCD Assembly and Installation
```sh
KVER=`uname -r`
cd ./vhci-hcd
sudo mkdir -p linux/${KVER}/drivers/usb/core
sudo cp /usr/src/linux-headers-4.4.0-210-generic/include/linux/usb/hcd.h linux/${KVER}/drivers/usb/core
sudo sed -i 's/#define DEBUG/\/\/#define DEBUG/' usb-vhci-hcd.c
sudo sed -i 's/#define DEBUG/\/\/#define DEBUG/' usb-vhci-iocifc.c
sudo make KVERSION=${KVER}

sudo make install

sudo tee -a /etc/modules <<< "usb_vhci_hcd"
sudo modprobe usb_vhci_hcd

sudo tee -a /etc/modules <<< "usb_vhci_iocifc"
sudo modprobe usb_vhci_iocifc
```

###### 2.2. Building and installing LibUSB VHCI
```sh
cd ../libusb_vhci
sudo ./configure
sudo make -s

sudo make install

sudo tee /etc/ld.so.conf.d/libusb_vhci.conf <<< "/usr/local/lib"
sudo ldconfig
```

###### 2.3. UsbHaspEmul Assembly and Installation
```sh
cd ../UsbHasp
sudo make -s

sudo cp dist/Release/GNU-Linux/usbhasp /usr/local/sbin
sudo mkdir /etc/usbhaspkey/
```

Let's create a unit usbhaspemul.service
```sh
sudo nano /etc/systemd/system/usbhaspemul.service
```
and add the following content to it
```unit
[Unit]
Description=Emulation HASP key for 1C
Requires=haspd.service
After=haspd.service
[Service]
Type=simple
ExecStart=/usr/bin/sh -c 'find /etc/usbhaspkey -name "*.json" | xargs /usr/local/sbin/usbhasp'
Restart=always
[Install]
WantedBy=multi-user.target
```

###### 2.4. Adding the UsbHaspEmul daemon to the startup
```sh
sudo systemctl daemon-reload
sudo systemctl enable usbhaspemul
```

###### 2.5. Uploading key dumps to the /etc/usbhaspkeys directory
```sh
cd ../dumps
sudo cp ./1c_server_x64.json /etc/usbhaspkey/
sudo cp ./50user.json /etc/usbhaspkey/
```

###### 2.6. Launching UsbHaspEmul
```sh
sudo systemctl start usbhaspemul
sudo systemctl status usbhaspemul
```

## 3. Installing and activating VirtualHere
###### !ACHTUNG No. 3! Despite the bit depth of the Ubuntu OS, VirtualHere is installed in 32-bit mode

###### 3.1. Installing the Linux-based backend
```sh
cd ../VirtualHere
sudo chmod +x ./install_server
sudo ./install_server
```

###### 3.2. Installing the Windows-based client part
To do this, in this repository, in the "WindowsClient" folder, find the file corresponding to your architecture:
* Windows (x86) - vhui32.exe
* Windows (x64) - vhui64.exe
* Windows (ARM64) - vhuiarm64.exe

Download and run the file.
In the window that opens, select the detected hub, tickle the PCM and click "License" in the menu.
In the window that opens, select and copy the serial number value:
Desktop Hub,s/n=```FE17189D-5211-C848-A448-788475CB15C8```,20 devices
This number is required to activate the VirtualHere server

###### 3.3. [X]Program activation
```sh
sudo systemctl stop virtualhere.service
sudo gcc ./activator.c -lcrypto -o ./activator
sudo ./activator /usr/local/sbin/vhusbdi386 <OUR COPIED SERIAL NUMBER>
```

###### 3.4. [X]Activation of the program. Adding a key to the Virtual Here config
The resulting string looks like:
```License=FE17189D-5211-C848-A448-788475CB15C8,20,MCECDwCdc5KISTF+TCfw6p6JJAIOS+CN+M5yfpp5LTXMofY=```
must be added to the Virtual Here config, which is stored in the following directory in Ubuntu Server:
```/usr/local/etc/virtualhere/config.ini```

To do this, copy the license string and paste it at the end of the Virtual Here config

```sh
nano /usr/local/etc/virtualhere/config.ini
```

or use the command:

```sh
sudo tee /usr/local/etc/virtualhere/config.ini <<< "License=FE17189D-5211-C848-A448-788475CB15C8,20,MCECDwCdc5KISTF+TCfw6p6JJAIOS+CN+M5yfpp5LTXMofY="

```

###### 3.5. Launching Virtual Here
```sh
sudo systemctl start virtualhere.service
sudo systemctl status virtualhere.service
```

## 4. Using
If the above steps are completed successfully, you can restart the key server and proceed to use it.
To do this, click on the device row in the client and click "Start using this device" in the menu that opens
