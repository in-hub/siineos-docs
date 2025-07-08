.. index:: Troubleshooting

.. _Troubleshooting:

Troubleshooting
===============

.. important:: All commands given in this chapter have to be run with superuser privileges. Therefore make sure to log in as user ``root`` for running the commands.

.. index:: USB driver, USB gadget

.. _UsbDriverIssues:

USB driver issues
-----------------

If you encounter issues during the USB driver installation, several measures should be taken:

* Unplug all USB devices (including the HUB-GM100) except for keyboard and mouse
* Uninstall any USB-related drivers and software such as *STLink*, *FTDI drivers* and similar – you can also postpone this step to a second iteration
* Start the :index:`Windows Device Manager` with the ``devmgr_show_nonpresent_devices=1`` environment variable set and click :guilabel:`Show hidden devices` on the :guilabel:`View` menu (see `Device Manager does not display devices that are not connected <https://support.microsoft.com/en-us/help/315539/device-manager-does-not-display-devices-that-are-not-connected>`_ for further information)
	* Uninstall any COM ports, STLink devices and other CDC and USB serial devices
	* Uninstall any :index:`USB Composite devices`
* Remove the registry key ``HKLM\SYSTEM\CurrentControlSet\Control\usbflags\1DB60104``
* Restart the computer

Now connect the HUB-GM100 again and check the driver installation process. If you still see an unknown serial gadget device in the Windows Device Manager, double click it, switch to the :guilabel:`Driver` tab, click :guilabel:`Update Driver` and choose :guilabel:`Browse my computer for driver software`. Before continueing, download a special `USB driver information file <https://download.inhub.de/siineos/linux-cdc-acm.inf>`_ . Back in the driver installation wizard, select the location in which the downloaded file has been saved. Windows should now use the information provided in that file to install the correct driver.

.. index:: systemd, systemctl, journalctl, System journal, dmesg, System errors, Logging, Log files, Units, Services, General problems

General problems
----------------

In case of general problems at the application or operating system level, messages in the system journal should be examined first since they might provide useful information on what went wrong at which time in which service, layer or component.

The first port of call on any Linux system is ``dmesg``. Running this command outputs all log messages issued by the Linux kernel. These messages are usually related to hardware and resources initializations and issues. Any hardware or filesystem errors are reported here.

Since SIINEOS is fully managed by ``systemd`` it can be inspected and controlled via ``systemctl`` and ``journalctl``. To get an overview of all loaded, active and running units (services, resources, mounts etc.), run ``systemctl``. To list all failed units run ``systemctl --failed``. Running ``systemctl status`` gives a summary of the current system state and print a tree view of all running units. To view the status and most recent log messages of one or multiple units, run ``systemctl status <NAME-PATTERN>``, e.g. ``systemctl status incore-app*``.

Additionally all kinds of log messages can be examined through the ``journalctl`` command. It can be run without any arguments which outputs both kernel and service messages. To query the messages of a certain unit (service, resource, mount), call it with the ``-u <UNIT>`` arguments, e.g.

.. code-block:: none

	root@hub-gm:~# journalctl -u incore-apps
	-- Logs begin at Thu 2019-02-14 10:12:00 UTC, end at Thu 2020-03-19 00:11:46 UTC. --
	Mar 19 00:00:02 hub-gm systemd[1]: Starting Start all InCore apps...
	Mar 19 00:00:03 hub-gm sudo[284]:     root : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/usr/bin/systemctl --no-pager -q start incore-app@test
	Mar 19 00:00:03 hub-gm sudo[284]: pam_unix(sudo:session): session opened for user root by (uid=0)
	Mar 19 00:00:03 hub-gm sudo[284]: pam_unix(sudo:session): session closed for user root
	Mar 19 00:00:03 hub-gm systemd[1]: Started Start all InCore apps.

.. seealso::

	* `systemctl manpage <https://www.freedesktop.org/software/systemd/man/systemctl.html>`_
	* `journalctl manpage <https://www.freedesktop.org/software/systemd/man/journalctl.html>`_
	* `How to use systemd to troubleshoot Linux problems <https://net2.com/how-to-use-systemd-to-troubleshoot-linux-problems/>`_
	* `11 Linux Performance Commands to Know as a System Administrator <https://geekflare.com/linux-performance-commands/>`_
	* `Linux troubleshooting 101: System performance <https://www.redhat.com/sysadmin/troubleshooting-system-performance>`_

.. index:: Apps filesystem

Apps filesystem
---------------

A FAT32 filesystem containing the installed apps is mounted at :file:`/apps` which you can verify by invoking ``df /apps`` or ``mount | grep apps``. The output should look like

.. code-block:: none

	root@hub-gm:~# df /apps/
	Filesystem     1K-blocks  Used Available Use% Mounted on
	/dev/mmcblk0p1     64366  1218     63148   2% /apps
	root@hub-gm:~# mount | grep apps
	/dev/mmcblk0p1 on /apps type vfat (ro,noatime,nodiratime,sync,uid=1000,gid=1000,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro,discard)

As indicated by the ``ro`` flag, the apps filesystem is mounted read-only per default to improve security and reliability and to reduce write cycles to the flash memory. It can be made writable by either activating the `InCore development mode <https://incore.readthedocs.io/en/latest/development-qsg/create-deploy-run.html#developmentmode>`_ or by running

.. code-block:: none

	mount /apps -o remount,rw

If the apps filesystem is not mounted properly during the boot process or ``dmesg`` indicates filesystem errors, you can try to either repair the filesystem by running

.. code-block:: none

	systemctl stop apps.mount
	fsck.vfat -a /dev/mmcblk0p1
	reboot

If the problem persists after the reboot, the filesystem has to be reinitialized. During this formatting process all data, i.e. all installed InCore apps, are removed and can't be recovered afterwards. So make sure to back up all required files before. When finished, run the following commands:

.. code-block:: none

	systemctl stop apps.mount
	mkfs.vfat -n SIINEOS_APP /dev/mmcblk0p1
	reboot

After rebooting, the filesystem should be mounted properly and should not contain any data.

.. index:: Storage filesystem, Data filesystem

Storage filesystem
------------------

An F2FS filesystem is mounted at :file:`/storage` allowing apps to store their data. It also holds all Docker-related data such as images, containers and volumes. To verify that the filesystem is mounted properly and to check the available space, run the ``df /storage`` and ``mount | grep storage`` commands:

.. code-block:: none

	root@hub-gm:~# df -h /storage/
	Filesystem      Size  Used Avail Use% Mounted on
	/dev/mmcblk0p6  6.7G  2.2G  4.6G  33% /storage
	root@hub-gm:~# mount | grep storage
	/dev/mmcblk0p6 on /storage type f2fs (rw,noatime,nodiratime,lazytime,background_gc=on,discard,no_heap,user_xattr,inline_xattr,acl,inline_data,inline_dentry,flush_merge,extent_cache,mode=adaptive,active_logs=6,alloc_mode=reuse,fsync)

The storage filesystem is checked for errors and inconsistencies on each boot. Any problems are fixed automatically whenever possible. If the filesystem is damaged unrecoverably, it has to be reinitialized. During this formatting process all InCore apps and Docker data are removed and can't be recovered afterwards. So make sure to back up all required files before. When finished, run the following commands:

.. code-block:: none

	systemctl stop storage.mount
	mkfs.f2fs -f /dev/mmcblk0p6
	reboot

After rebooting, the filesystem should be mounted properly and should not contain any data.

.. index:: Factory reset, Factory settings

Factory reset
-------------

.. warning:: When resetting the device to factory defaults, all installed apps, settings, database files, Docker containers and other custom files will be lost. Please make sure to back up all important files before.

The device can be reset to factory settings by flashing a SIINEOS image to the device (contrary to installing an update bundle only). Please make sure to back up all required files before. When finished, run the following commands to download and flash the device:

.. code-block:: none

	cd /tmp
	systemctl stop storage.mount
	systemctl stop apps.mount
	incore-cli download https://download.inhub.de/siineos/images/siineos-standard-armhf-disk-v2.7.7.img.gz
	zcat siineos-standard-armhf-disk-v2.7.7.img.gz | dd of=/dev/mmcblk0 bs=4M
	sync

Do not interrupt the last command and do not interrupt the power supply of the device while it is running. After the last command has finished, unplug the device from the USB port and any power supplies. Eventually the device should start with a clean SIINEOS |version| installation.
