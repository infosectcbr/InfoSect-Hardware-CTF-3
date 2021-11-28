# Hardware CTF 3

This challenge will have you find and exploit bugs in a SOHO router.

## Flag 1 - Physical Access

Nb: If you do not have physical access to a real router, go onto the next section on emaulation.

Gain physical access to the UART on the router and establish a serial console. To do this, use a screw driver to pry off the case, and then pull out the circuit board. You can identify some header pins that will have UART access to give you a shell.

Read this section to learn how to connect to UART or look at the previous hardware CTFs on UART https://infosectcbr.github.io/InfoSect-Hardware-CTF-1/](https://infosectcbr.github.io/InfoSect-Hardware-CTF-1/) Level 1.

To connect the router to USB serial, connect GND on the router to GND on the serial bridge. TXO on the router to RX on the serial bridge. And RXO on the router to TX on the serial bridge. The router header pinout is as follows (and is also visible on the PCB silk screen. You might might need to use the camera on your phone and zoom in to take a magnified picture):

```
XXX0
0000

GND RXO TXO VCC
CLK  SO TX1 RX1
```

Install minicom on a Linux VM:

```
sudo apt-get install minicom
```

Pass through the USB serial to your VM via removable devices in VMWare.

Run minicom:

```
sudo minicom -D /dev/ttyUSB0
```

Configure minicom by typing ctrl-a then z on its own. O to enter setup. Disable software and hardware flow control. The baud rate is 115200. Exit out of the configuration and hit enter a couple of times to see your shell!

Sometimes in UART you are presented with a root shell without needing to login. If you need to login, grab a password from the workshop hosts.

Now you have a shell, look around the filesystem for an obvious file containing the flag.

If you do not have a router in physical possession, you will need to emulate the firmware as described in the next section.

### Firmware Emulation

Download the Router Firmware Infosect-Hardware-CTF-3-Firmware.tgz (or InfoSect-Hardware-CTF-3-Firmware-Golden.tgz). See the Discord pinned messages for links or grab a USB stick in the workshop.

Download the QEMU images from [https://people.debian.org/~aurel32/qemu/mipsel/](https://people.debian.org/~aurel32/qemu/mipsel/) (or grab a USB stick).

Then from Linux, install QEMU and run:

```
qemu-system-mipsel \ 
  -M malta \ 
  -kernel vmlinux-3.2.0-4-4kc-malta \ 
  -hda debian_wheezy_mipsel_standard.qcow2 \ 
  -append "root=/dev/sda1 console=tty0" \ 
  -net nic \ 
  -net user,hostfwd=tcp:127.0.0.1:7777-:22
```

You can add the -nographic option if you don't want to see QEMU in a GUI.

The root password is 'root'. QEMU is doing port fowarding so we should be able to use local port 7777 to connect to our image. Confirm you can ssh into the QEMU image with
```
ssh -p 7777 root@localhost
```

scp the firmware from your computer to your QEMU image:

```
scp -P 7777 InfoSect-Hardware-CTF-3-Firmware.tgz root@localhost:
```

Now back in your ssh session, untar the firmware image in the /root (the default on login) directory.

```
tar xzvf InfoSect-Hardware-CTF-3-Firmware.tgz
```

Now run the HTTPd daemon in your ssh session.

```
chroot ./root-ramips /usr/bin/lighttpd-custom -f /lighttpd.conf
```

Now look in the ./root-ramips directories to find the FLAG file.

## Flag 2 - Ports Incoming

On the device (or inside QEMU on the shell), run the command:

```
netstat -plant
```

You should see that lighttpd-custom is listening on 2 ports. One of these ports has a webserver, the other is serving a flag.

## Flag 3 - HTTP Headers

At this point, submit exploits to the workshop hosts over discord. We'll throw the exploit on a real router to confirm you won the flag.

You might have noticed some leaked lighttpd (HTTPd) source code on the system. It makes reference to a backdoor that has been inserted into the custom web server. To trigger the backdoor, you need to request /index.html from the server and use the correct HTTP header after the GET request. Look closely at the output of the webserver for the flag.

In your QEMU image, you have tools such as netcat (nc), telnet, python, bash, and wget available. These tools are enough to solve the challenge. It is recommended to use hand crafted HTTP GET requests for these challenges.

## Flag 4 - Buffer Overflow

There is a buffer overflow in the lighttpd custom webserver when it processes another HTTP header. The overflow overwrites an admin variable that needs to be set to a specific value for the flag to be revealed.

You will need to copy the vsftpd-custom binary off the router to a laptop so you can analyse it. If you have physical access to the router, try copying it to a USB stick inserted into the router. Most routers will need to support the filesystem of the USB stick. If you look at /proc/filesystems on the router, you can see which filesystems it supports. You might need to use mkfs.vfat to create the filesystem if you are doing it on command line.

You might want to look at the firmware in Ghidra and see if you can identify any functions that are named similar to do_buffer_overflow. Look at the code and the XREFS that use this function. You'll be able to figure out which HTTP header has the buffer overflow.

The overflow is in the data section of the HTTP header. It is between 128 and 256 bytes in size. If you trigger the buffer overflow, the web server will tell you if you constructed the buffer correctly. Construct the correct buffer and get the flag.

## Flag 5 - FTP

You will analyse the vftpd-custom binary in Ghidra. There is a backdoor FTP command that will reveal the flag.

Look for strings using Ghidra to identify USER and PASS, which are standard FTP commands. These strings are quite small so Ghidra doesn't immediately recognise them as strings since it is too ambiguous. Try searching memory using the 's' shortcut. Tell Ghidra you're looking for a string and try searching for USER. You might come across a string that is identified but begins with U and isn't fully displayed as USER in the output. You will need to change the data type of the memory at this address to say it's a string. You can find similar commands in the binary near to the USER string. You can also find XREFS to the strings. XREFS can tell us which functions refer to these strings and this can be used to identify the command parsing function(s). Look for command that is suspicious. The backdoor command will be preauthentication.

Another tip is to try downloading the vsftpd source code to see what the command parsing code looks like, and to identify any unusual changes.

One final tip is to simply run the strings command on the binary and see if anything in the command strings looks suspicious.

To emulate the vsftpd-custom binary, use:

```
chroot ./root-ramips /usr/bin/vsftpd-custom
```

Note that if you use an FTP client in your QEMU image, it won't be able to recognise the backoor commands since it is not part of the FTP specification. However, if you netcat or telnet to the FTP port, you can send raw commands to solve this challenge.
