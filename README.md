# Secura-Pi-Cam
![logo 1](../master/logo/1.png?raw=true)
A simple security system to run on a [Raspberry Pi](https://www.raspberrypi.org/).

![picture 1](../master/logo/Picture1.png?raw=true)

Features:
  - Motion triggered image capture.
  - Mobile notifications with photos.
  - Detects when you are home and arms or disarms automatically.
  - Can be remotely disabled or queried for status using [Telegram](https://telegram.org/).

## Requirements

You will need this hardware:
  - Raspberry Pi with camera interface. I used a model 3.
  - Raspberry Pi camera module.
  - PIR sensor module. Any generic HC SR501 (or similar) module should work. 
  - USB Wi-Fi that supports monitor mode. You need this because even though raspberry pi 3 has inbuilt wifi it doesn't support monitor mode.
  
Other requirements:
  - A [Telegram bot](https://telegram.org/blog/bot-revolution). It's free and easy to setup.
  - Raspbian OS to run inside the raspberry pi 3.
  - Python 2.7.

## How it works

### Automatic presence detection

To have the system completely automatic the easiest way was to try and detect the mobile phones of the home occupants. Conceptually this was quite simple but in practice it was the most challenging part because:
  - Capturing all packets on a Wi-Fi interface is too resource intensive.
  - There are presently no good 5Ghz USB Wi-Fi adapters that support monitor mode. This means packet monitoring is restricted to 2.4Ghz where most modern mobile phones use 5Ghz now.
  - Mobile phones are not always online and sending packets over Wi-Fi. Sometimes they stay unconnected for 15 minutes or longer.
  - Even with 99% accuracy false alarms are annoying.

I used an approach that mixes active (ARP scan) and passive (packet capture) detection over the Wi-Fi adapter based on knowing the MAC addresses of the mobile phones. The mobile phone MAC addresses are set in the configuration and the secura-pi-cam application captures packets on a monitor mode interface with the following filter:
1. Wi-Fi probe requests from any of the configured MACs.
2. Any packets sent from the configured MACs to the host running rpi-security.

The application resets a counter when packets are detected and if the counter goes longer than ~10 minutes the system is armed. To eliminate the many false alarms, when transitioning from armed to disarmed state or vice versa, the application performs an ARP scan directed at each of the configured MAC addresses to be sure they are definitely online or offline. Both iOS and Android will respond to this ARP scan 99% of the time where a ICMP ping is quite unreliable. By combining the capture of Wi-Fi probe requests and using ARP scanning, the Wi-Fi frequency doesn't matter because mobile phones send probe requests on both frequencies and ARP scan works across both frequencies too.

### Notifications

A [Telegram](https://telegram.org/blog/bot-revolution) bot is used to send notifications with the captured images. 
If the system is in an armed state and motion is detected then a message with the captured image is sent to you from the Telegram bot.

Notifications are also sent on any alarm state change.

### Remote control

You can send the Telegram bot commands that trigger certain actions.

  - */disable*: Disables the service until re-enabled.
  - */enable*: Enables the service after it being disabled.
  - */status*: Sends a status report.

### Python

I wrote the whole application in python. Large parts of the functionality are provided by the following pip modules:
  - [picamera](https://github.com/waveform80/picamera)
  - [Scapy](http://www.secdev.org/projects/scapy/)
  - [python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot)

The application uses multithreading in order to process events asynchronously. There are 4 threads:
  - telegram_bot: Responds to commands.
  - monitor_alarm_state: Arms and disarms the service when packets are detected.
  - capture_packets: Captures packets from the mobile devices.
  - process_photos: Sends captured images via Telegram messages.

## Installation, configuration and Running

The interface used to connect to your WiFi network must be the same interface that supports monitor mode. And this must be the same WiFi network that the mobile phones connect to.

Be sure to connect your PIR sensor to the correct pin. The pin is specified in ``/etc/secura_pi_cam.conf`` using the pin number.

First install required packages:

```
sudo apt-get update
sudo apt-get install tcpdump iw python-dev python-pip 
```

Update pip:

```
pip install --upgrade pip
```

Install current master of scapy as the current release is missing a required bug fix:

```
sudo pip install https://github.com/secdev/scapy/zipball/master
```

Install rpi-security, reload systemd configuration and enable the service:

```
sudo pip install https://github.com/RAVEENSAVINDA/rpi-security/zipball/master
sudo systemctl daemon-reload
sudo systemctl enable secura_pi_cam.service
```

Add your MAC address or addresses, Telegram bot API key and any other changes to ``/etc/secura_pi_cam.conf``.

Ensure you have enabled the camera module using ``raspi-config``.

And start the service:

```
sudo systemctl start secura_pi_cam.service
```

You need to send at least one message to the Telegram bot otherwise it won't be able to send you messages. This is so the service can save the telegram chat_id. So just send the ``/status`` command.

It runs as a service and logs to syslog. To see the logs check ``/var/log/syslog``.

There is also a debug option that logs to stdout:

```
root@raspberrypi:~# iw phy phy0 interface add mon0 type monitor
root@raspberrypi:~# ifconfig mon0 up
root@raspberrypi:~# secura_pi_cam.py -d
2016-05-28 14:43:30 DEBUG   secura_pi_cam.py:73  MainThread          State file read: /var/lib/secura_pi_cam/state.yaml
2016-05-28 14:43:30 DEBUG   secura_pi_cam.py:44  MainThread          Calculated network: 192.168.178.0/24
2016-05-28 14:43:41 INFO    secura_pi_cam.py:214 monitor_alarm_state thread running
2016-05-28 14:43:41 INFO    secura_pi_cam.py:196 capture_packets     thread running
2016-05-28 14:43:41 INFO    secura_pi_cam.py:259 telegram_bot        thread running
2016-05-28 14:43:41 INFO    secura_pi_cam.py:154 process_photos      thread running
2016-05-28 14:43:43 INFO    secura_pi_cam.py:392 MainThread          rpi-security running
2016-05-28 14:43:43 INFO    secura_pi_cam.py:112 MainThread          Telegram message Sent: "secura_pi_cam running"
2016-05-28 14:44:29 DEBUG   secura_pi_cam.py:191 capture_packets     Packet detected from aa:aa:aa:bb:bb:bb
2016-05-28 14:44:29 DEBUG   secura_pi_cam.py:191 capture_packets     Packet detected from aa:aa:aa:bb:bb:bb
2016-05-28 14:44:48 DEBUG   secura_pi_cam.py:280 Dummy-1             Motion detected but current_state is: disarmed
```

This shows my WLAN network device arrangement:

```
root@raspberrypi:~# iw dev
phy#0
        Interface mon0
                ifindex 4
                wdev 0x3
                addr 00:0f:60:08:9c:01
                type monitor
        Interface wlan0
                ifindex 2
                wdev 0x1
                addr 00:0f:60:08:9c:01
                type managed
                channel 1 (2412 MHz), width: 40 MHz, center1: 2422 MHz
```

You could have interfaces with different names, just be sure to change the ``network_interface`` parameter in ``/etc/secura_pi_cam.conf`` and also the reference to mon0 in [rpi-security.service](https://github.com/RAVEENSAVINDA/Secura-Pi-Cam/blob/master/etc/secura_pi_cam.service)

### Reboot on connectivity loss

I created a cron job to check connectivity and reboot if the check fails.

```
echo '*/20 * * * * root /usr/bin/host api.telegram.org > /dev/null 2>1 || (/usr/bin/logger "Rebooting due to connectivity issue"; /sbin/shutdown -r now)' > /etc/cron.d/reboot-on-connection-failure
```
