# Passy's Raspbian ADSL Router

<img src="https://www.raspberrypi.org/wp-content/uploads/2015/08/raspberry-pi-logo.png" width=150 align=left>

## Setup

I bought
a [DrayTek Vigor 130](http://www.draytek.co.uk/products/business/vigor-130)
which is a bridge-only ADSL2+/VDSL modem mostly used in business environments (I
like my cheap enterprise hardware) which is plugged into a normal ethernet
switch which also connects my Raspberry Pi 2B (provisioned by this repo) and
a [UniFi® AP AC LITE](https://www.ubnt.com/unifi/unifi-ap-ac-lite/) for WiFi.
The switch is just some boring Gigabit switch from Netgear which had a
trustworthy looking amount of stars on Amazon.

Nothing special or particularly exciting about this. It looks roughly like this:

```
+---------+     +---------+     +---------+
|         |     |         |     |         |
|  Modem  |     |  DHCP/  |     |  WiFi   |
|         |     |  PPP    |     |         |
+--------++     +----|----+     ++--------+
         |           |           |
         |           |           |
         |           |           |
      +--v-----------v-----------v---+
      |                              |
      |            Switch            |
      |                              |
      +------------------------------+
```

The nice thing about this is that I get a lot of control about all the
individual pieces and if something goes wrong I don't just have one big black
box that I'll have to hope will answer my prayers.

In particular, my previous modem had the annoying hard-coded behavior of
retrying five times on connection issues and then require a hard reset.

With my Raspberry Pi having this responsibility now, I get fine-grained
control over what it will do in this situation.

## Subnets and prefix allocation

For a long time I was under the impression that I'd have to use something
like wide-dhcpcv6-client to get a /56 subnet assigned from my ISP. In actuality,
the had assigned me a static /56 and the timeouts I was seeing in the log
weren't an incorrect setup on my end, but simply indicating that they did
not even have a server running on their end.

The important bit that took me embarrassingly long to figure out was
that I needed to set the static IPv6 address of my Pi to one within
that subnet. I started just going with one of the addresses within
the /64 of the IPv6 address I got assigned on `ppp0`. This worked in
so far that I could actually make connections from my Pi, but
none of my other devices on the network could access the internet.
This is where my NAT-trained brain collided with the IPv6 realities
where routing doesn't simply mean adding a magical default gateway
and letting masquerading take care of it.

From everything I've read online, it is rather unusual to have your
/56 assigned to your account, so you may very likely have to set up WIDE-DHCPv6.
See the resources section at the end of this page for more information.

## Plan

Since this is a Rasperry Pi, I'd like to make some use of the GPIO ports. Maybe
show the number of devices on an LCD, or have an LED flash up if someone new
gets a lease. Now that it does PPP, I should definitely be able to show an LED
with the connection status.

Crazy long-term goal: build my own LTE fallback when the ADSL goes down?

## Preparing the SD Card

- [Get Raspbian Jessie Lite](https://downloads.raspberrypi.org/raspbian_lite_latest.torrent).
  I used `2016-03-18` for this. Also, don't be a jerk, and use the Torrent
  option.
- I use Lite because I don't even want to connect a monitor to the Pi and
  updating Xorg takes ages.
- Copy it onto an SD card. I'll leave it to you to figure out which device
  you're writing to, but keep in mind that this image is intended to partition
  the entire card, so don't specify a partition (i.e. `sdb` not `sdb1`):
  `dd bs=4M if=2016-03-18-raspbian-jessie.img of=/dev/sdb`

## Initial setup

- Plug the Pi in and connect it to a DHCP-enabled router via ethernet.
- My old DD-WRT had the `.lan` domain, so I could directly connect to
  `raspberrypi.lan`. Obviously, take care of conflicts if you have more than
  one. `nmap` is your friend.
- `ssh pi@raspberrypi.lan`, password is `raspberry`.
- This could potentially be part of the ansible script, but I prefer pushing the
  SSH key over manually to minimize the chance of locking myself out.
- `ssh pi@raspberrypi.lan "mkdir -p ~/.ssh/"; scp ~/.ssh/id_rsa.pub pi@raspberrypi.lan:/home/pi/.ssh/authorized_keys`
- *Dont forget this one:* Resize your partition or you're gonna have a bad time.
  Raspbian only leaves you with a few hundred megabytes left, so run
  `sudo raspi-config` and select option 1: "Expand Filesystem".

## Ansible

Copy `playbook.yml.example` to `playbook.yml` and make the necessary adjustments,
notably add your premiumize.me credentials to it. Afterwards, you can run it
like:

```
ansible-playbook -i hosts playbook.yml
```

This assumes that the hostname in `hosts` can be resolved and you can log in
password-less via the `pi` user. It also expects the Pi to already have a
working internet connection.

## Resources

- [https://blog.confirm.ch/using-pppoe-on-linux/](Using PPPoE on Linux)
- [https://major.io/2015/09/11/time-warner-road-runner-linux-and-large-ipv6-subnets/](Time Warner Road Runner, Linux, and large IPv6 subnets)
