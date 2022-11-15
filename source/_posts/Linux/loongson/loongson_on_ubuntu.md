---
title: How to setup environment for loongson LS1C0300B(mipsel) on Ubuntu18.04
date: 2019-11-27 15:48
tags:
    - loognson
    - mipsel
    - cross-compile
---



This article is about setting up an environment for enbedded Linux application development for loongson. The resulting environment enables cross-platform application development for Loongson mipsel-based SOMs/COMs using Ubuntu18.04 for application development.

## Communicate with loongson development board using USB cable
The first thing we need to do is communicating with hardware using USB cable.The boards are equipped with PL2303-based device. To be able to talk to an application, our machine must have appropriate PL2303 drivers. Fortunately, Ubuntu18.04 is default equipped with the PL2030 drivers(we could use `lsmod | grep pl2303` to check it). The only thing necessary is to check whether the Ubuntu machine can communicate over USB with the attached loongson development board.

To do this:

- Connect the PC via an USB cable to the USB UART port of the loongson development board.
- Turn the board on.

Note the correspondence between the colored cable and thier pinout:

- Black wire(GND)
- Green wire(RX)
- White wire(TX)

Use `lsusb` command to list the usb devices connected to our machine. The output text of this command in terminal is something like this:

```bash
Bus 001 Device 007: ID 0bda:0129 Realtek Semiconductor Corp. RTS5129 Card Reader Controller
Bus 001 Device 008: ID 0cf3:0036 Atheros Communications, Inc. 
Bus 001 Device 005: ID 0c45:6a04 Microdia 
Bus 001 Device 004: ID 17ef:6050 Lenovo 
Bus 001 Device 010: ID 046d:c326 Logitech, Inc. 
Bus 001 Device 011: ID 067b:2303 Prolific Technology, Inc. PL2303 Serial Port
Bus 001 Device 002: ID 8087:8000 Intel Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

This shows that the PL2303 devices on the development board has been detected and that the drivers on the PC are running. To find out what RS232(UART) port is used by the USB driver, type:

```bash
dmesg | grep -ie PL2303
```

The return message:

```bash
[ 8907.910272] usbcore: registered new interface driver pl2303
[ 8907.910924] usbserial: USB Serial support registered for pl2303
[ 8907.911054] pl2303 1-1.1:1.0: pl2303 converter detected
[ 8907.913355] usb 1-1.1: pl2303 converter now attached to ttyUSB0
```

The message tells us where the driver connects to a TTY port, in this case, `usb 1-1.1: pl2303 converter now attached to ttyUSB0`.

### picocom
Picocom is a minimal dumb-terminal emulation program that is great for accessing a serial port based Linux console; which is typical done when developing an embedded Linux based product. 

Install picocom with this command:

```bash
sudo apt-get install picocom
```

Now, we know the serial port, set the speed to 115200 baud.

```bash
sudo picocom -b 115200 /dev/ttyUSB0
```

Now, we can communicate with the board.

```bash
picocom v2.2

port is        : /dev/ttyUSB0
flowcontrol    : none
baudrate is    : 115200
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        : 
omap is        : 
emap is        : crcrlf,delbs,

Type [C-a] [C-h] to see available commands

Terminal ready

[root@Loongson:/]#
```

## Cross Compile
Now we should install the cross tool chain. I download the `mips-loongson-gcc4.9-2019.08-05.linux-gnu.tar.gz` from the loongson website. Unpack the *tar.xz* file and move it to the `/opt`. Export the environment variables by adding the following lines into the `~/.bashrc`:

```bash
export PATH="/opt/gcc-4.3-ls232/bin:$PATH"
export PATH="home/readlnh/Application/mipsel-linux-musl-cross/bin:$PATH" 
```

Don't forget to run command `source ~/.bashrc`.

Now, we could compile `a.c`:

```bash
$ mipsel-linux-gnu-gcc a.c
```

## tftp server
On ubuntu, run ` sudo apt-get install tftpd-hpa tftp-hpa`. Then edit the `/etc/default/tftpd-hap`:

```bash
# /etc/default/tftpd-hpa                                                     
  TFTP_USERNAME="tftp"
  TFTP_DIRECTORY="/home/readlnh/tftpboot"
  TFTP_ADDRESS="0.0.0.0:69"
  TFTP_OPTIONS="-l -c -s"
```

Create file `/etc/xinetd.d/tftp` with the following contents:

```bash
service tftp
{
	disable = no
	socket_type = dgram
	protocol = udp
	wait = yes
	user = root
	server = /usr/sbin/in.tftpd
	server_args = -s /home/readlnh/tftpboot -c
	per_source = 11
	cps = 100 2
}
```

Create directory `/home/readlnh/tftpboot` and set its permissions:

```bash
$ sudo mkdir /home/readlnh/tftpboot
$ sudo chmod -R 777 /home/readlnh/tftpboot
$ sudo chown -R nobody /home/readlnh/tftpboot
```

Restart the `xinetd` service:

```bash
$ sudo service xinetd restart
```

We could use `ifconfig` to get the ip address of our host machine. Make sure that ethernet cable is plugged into the board and a dhcp server is running in either your host machine or your router. In this case, my host machine's ip address is `192.168.1.230`. Run `ifconfig` to set the ip address of the board to make sure the address fall into the same subnet, here I set it `192.168.1.108`. 

Now, we could `ping` our host machine by `ping 192.168.1.230`:

```bash
PING 192.168.1.230 (192.168.1.230): 56 data bytes
64 bytes from 192.168.1.230: seq=0 ttl=64 time=5.054 ms
64 bytes from 192.168.1.230: seq=1 ttl=64 time=4.140 ms
64 bytes from 192.168.1.230: seq=2 ttl=64 time=0.843 ms
64 bytes from 192.168.1.230: seq=3 ttl=64 time=3.813 ms
64 bytes from 192.168.1.230: seq=4 ttl=64 time=3.727 ms
64 bytes from 192.168.1.230: seq=5 ttl=64 time=0.562 ms
64 bytes from 192.168.1.230: seq=6 ttl=64 time=3.749 ms
...
```

Download the `a.out` we compiled before by `tftp -r a.out -g 192.168.1.230`:

```bash
a.out                100% |*******************************|  7796   0:00:00 ETA
```

Run the `a.out`:

```bash
[root@Loongson:/]#chmod u+x a.out 
[root@Loongson:/]#./a.out
Hello World!
```
## Hints
Loongson only provide `gcc4.9` for LSC1C0300B, howerver, the `mipsel-linux-gnu-gcc` in Ubuntu apt source also works well.


## Reference
- [Communicate with hardware using USB cable for Ubuntu](https://elinux.org/Communicate_with_hardware_using_USB_cable_for_Ubuntu) 

- [TFTP Boot using u-boot](https://rechtzeit.wordpress.com/2013/01/16/tftp-boot-using-u-boot/) 
