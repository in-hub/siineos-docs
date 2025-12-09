Networking
==========

.. index:: Network interface, Network configuration, TCP/IP, IP address

.. _NetworkingGeneral:

General
-------

.. important:: The recommended method to configure network interfaces in SIINEOS (except for USB networking) is to implement InCore applications utilizing the `NetworkConfiguration <https://incore.readthedocs.io/en/latest/InCore.Foundation/NetworkConfiguration.html>`_ and any of the `NetworkInterface <https://incore.readthedocs.io/en/latest/InCore.Foundation/NetworkInterface.html>`_-based objects. The InCore objects allow an easy, dynamic and high-level network configuration without any command line interaction.

General network status information such as IP addresses are available through the ``networkctl`` command:

.. code-block:: none

	root@hub-gm:~# networkctl
	IDX LINK             TYPE               OPERATIONAL SETUP
	  1 lo               loopback           carrier     unmanaged
	  2 can0             n/a                off         unmanaged
	  3 eth0             ether              routable    configured
	  4 eth1             ether              no-carrier  configuring
	  5 usb0             gadget             routable    configured

	5 links listed.
	root@hub-gm:~# networkctl status
	*        State: routable
	       Address: 192.168.2.54 on eth0
	                192.168.123.1 on usb0
	                169.254.39.21 on usb0
	                fe80::3ad2:69ff:fe2d:1d5 on eth0
	                fe80::7450:1dff:fe7d:59f on usb0
	       Gateway: 192.168.2.1 on eth0
	           DNS: 192.168.2.1
	           NTP: 192.168.2.1
	root@hub-gm:~# networkctl status eth0
	* 3: eth0
	       Link File: n/a
	    Network File: /etc/systemd/network/eth0.network
	            Type: ether
	           State: routable (configured)
	      HW Address: 38:d2:69:2d:01:d5 (Texas Instruments)
	         Address: 192.168.2.54
	                  fe80::3ad2:69ff:fe2d:1d5
	         Gateway: 192.168.2.1
	             DNS: 192.168.2.1
	             NTP: 192.168.2.1

For diagnostic purposes detailed information of the current network configuration can be examined through certain invocations of the ``ip`` command:

* Link status: ``ip link`` (shortcut: ``ip l``)
* IP addresses: ``ip address`` (shortcut: ``ip a``)
* Routing table: ``ip route`` (shortcut: ``ip r``)

.. seealso::

	* `networkctl manpage <https://www.freedesktop.org/software/systemd/man/networkctl.html>`_
	* `WiredNetworkInterface example <https://incore.readthedocs.io/en/latest/InCore.Foundation/WiredNetworkInterface.html#example>`_
	* `Introduction to iproute2 <https://lartc.org/howto/lartc.iproute2.html>`_

.. index:: USB networking

USB
---

When connecting a SIINEOS-based device to a computer via USB, a virtual :index:`USB network adaptor` is created on the computer. It allows communicating with the device based on TCP/IP connections. While your computer is assigned a random IP address in the USB network subnet, the SIINEOS device is always accessible via the IP address ``192.168.123.1``. You can verify the USB network functionality by running ``ping 192.168.123.1`` in a command line window of your computer's operating system. The output looks like

.. code-block:: none

	$ ping 192.168.123.1
	PING 192.168.123.1 (192.168.123.1) 56(84) bytes of data.
	64 bytes from 192.168.123.1: icmp_seq=1 ttl=64 time=0.418 ms
	64 bytes from 192.168.123.1: icmp_seq=2 ttl=64 time=0.558 ms
	64 bytes from 192.168.123.1: icmp_seq=3 ttl=64 time=0.593 ms
	64 bytes from 192.168.123.1: icmp_seq=4 ttl=64 time=0.558 ms
	--- 192.168.123.1 ping statistics ---
	4 packets transmitted, 4 received, 0% packet loss, time 3058ms
	rtt min/avg/max/mdev = 0.418/0.531/0.593/0.072 ms

After :ref:`starting the SSH service <SshService>` you can use the IP address to log in via SSH or transfer files via *WinSCP* or ``scp``. When developing InCore applications providing a web interface, the web interface is accessible through the USB IP address.

.. index:: eth0, eth1, IEEE 802.3

Ethernet
--------

Both :index:`Ethernet interfaces` (``eth0``/``eth1``) are configured through :index:`DHCP` per default, i.e. they request an IP address from the network which they are connected to. You can obtain the assigned IP addresses using commands mentioned in section :ref:`NetworkingGeneral`.

If your network does not distribute IP addresses via DHCP, the Ethernet interfaces have to be configured manually. You can use the ``ip`` command to change the configuration temporarily. Alternatively modify :file:`/etc/systemd/network/eth0.network` or :file:`/etc/systemd/network/eth1.network` in a `SystemdNetworkd-compatible manner <https://wiki.debian.org/SystemdNetworkd>`_ and restart the networking service via ``systemctl restart systemd-networkd``. Please note that changes to these files are not persistent and will be lost after rebooting. Instead implement an InCore applications which performs the network configuration as mentioned in section :ref:`NetworkingGeneral`.

.. index:: Wiress network interface, WLAN, Wifi, IEEE 802.11, wlan0

Wireless
--------

The optional wireless network interface (``wlan0``) has to be configured by InCore applications using the `WirelessNetworkInterface <https://incore.readthedocs.io/en/latest/InCore.Foundation/WirelessNetworkInterface.html>`_ object.

For troubleshooting purposes you can examine wireless security related issues (i.e. WPA encryption issues) by running ``journalctl -u wpa_supplicant``. The assigned IP address(es) can be viewed via ``ip address show wlan0``.

.. index:: Broadband connection, LTE USB stick, Mobile network interface, Modem, wwan0

Broadband/LTE
-------------

The optional broadband/LTE network interface (``wwan0``) has to be configured by InCore applications using the `MobileNetworkInterface <https://incore.readthedocs.io/en/latest/InCore.Foundation/MobileNetworkInterface.html>`_ object.

For troubleshooting purposes you can examine modem-related issues by running ``journalctl -u ModemManager``. The assigned IP address(es) can be viewed via ``ip address show wwan0``.
