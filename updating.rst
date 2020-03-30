.. index:: Operating system update

Updating
========

.. index:: Slots, Active Slot, Inactive Slot, Dual boot

Introduction
------------

SIINEOS consists of a combined boot and system image and uses `RAUC <https://rauc.io>`_ as the underlying technology for installing updates. Unlike desktop or mobile operating systems SIINEOS is always updated as a whole, i.e. an update installs a full :index:`operating system image`. All SIINEOS-compatible devices provide two *slots* for such images while only one slot is active at a time (*Dual boot*). An *active slot* contains the boot and system image which is currently used for booting and running the system. This allows installing a different version of SIINEOS in the currently *inactive slot* without touching the active slot. In case a slot with a newly installed version of SIINEOS does not boot successfully after several attempts, the device will revert back to the previously active slot. This makes the update process reliable and prevents bricked devices.

In order to perform one of the available installation procedures, :ref:`connect to the device and log in <LoggingIn>` as user ``root``.

.. index:: Online update, OTA

Online update
-------------

SIINEOS provides a command which performs the download and installation of a certain version of SIINEOS:

.. code-block:: none

	update-siineos <VERSION>

If you want to install e.g. version *2.2.0* run the following command:

.. code-block:: none

	update-siineos 2.2.0

Make sure the device has access to the Internet e.g. via Ethernet or WLAN so it can contact the SIINEOS download servers.

.. warning:: Be careful when performing an online update via a broadband/LTE connection since at least 100 MB of data volume is consumed per download. Consider the possibility of an :ref:`OfflineUpdate` instead.

.. index:: Offline update

.. _OfflineUpdate:

Offline update
--------------

If the device does not have access to the Internet, the update bundle can be downloaded elsewhere and stored on a USB memory stick. After inserting the USB memory stick into the USB port of the device, run the following commands.

.. code-block:: none

	mount /dev/sda1 /mnt
	rauc install /mnt/siineos-armhf-update-<VERSION>.raucb


If the ``mount`` command prints an error about a device not found, try ``/dev/sda`` (i.e. without ``1``) as the first argument. The first command will make the USB memory stick available at :file:`/mnt`. The second command performs the actual update installation. If no error is reported, restart the device by entering ``reboot``.

.. index:: Auto-update

Updating automatically
----------------------

SIINEOS as is does not have mechanisms for installing a new version automatically. However an auto-update functionality can be implemented on the InCore application level easily since all required components are ready for use. See the `UpdateManager documentation <https://incore.readthedocs.io/en/latest/InCore.Foundation/UpdateManager.html>`_ for details.
