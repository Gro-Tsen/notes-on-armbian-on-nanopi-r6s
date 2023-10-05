# Notes concerning Armbian on the NanoPi R6S #

## Ethernet adapters ##

There are three Ethernet ports on the NanoPi R6S:

* The leftmost one (when facing the port) is labeled "LAN2" on the
  box, and it is `/devices/platform/fe1c0000.ethernet` for Linux
  (`/ethernet@fe1c0000` in the DTB, also symbolized as `&gmac1`).
  This is a 1Gbps adapter using `rk_gmac-dwmac` as driver.  It is
  aliased as `ethernet1` in the default DTB, so Linux might give it
  the name `end1`…  except that the default Armbian config renames it
  to `wan` which is clearly an error since it is not the one with that
  label.

* The middle one (when facing the port) is labeled "LAN1" on the box,
  and it is
  `/devices/platform/fe190000.pcie/pci0004:40/0004:40:00.0/0004:41:00.0`
  for Linux (`/pcie@fe190000/pcie@40/pcie@40,0` in the DTB, also
  symbolized as `&r8125_u40`).  This is a 2.5Gbps adapter using
  `r8125` as driver.  It is *not* aliased in the default DTB (and this
  is clearly wrong); Linux might give it the name `enP4p65s0`.

* The rightmost one (when facing the port) is labeled "WAN" on the
  box, and it is
  `/devices/platform/fe180000.pcie/pci0003:30/0003:30:00.0/0003:31:00.0`
  for Linux (`/pcie@fe180000/pcie@30/pcie@30,0` in the DTB, also
  symbolized as `&r8125_u25`).  This is a 2.5Gbps adapter using
  `r8125` as driver.  It is aliased as `ethernet0` in the default DTB,
  so Linux might give it the name `end0` and/or also `enP3p49s0`.

## Trying to make sense of the Armbian boot process ##

(This section is mostly not specific to the NanoPi R6S and is just the
result of my trying to understand who gets what data from whence.)

* The U-Boot is contained in the files
  `/usr/lib/linux-u-boot-legacy-nanopi-r6s/idbloader.img` (bootstrap?)
  and `/usr/lib/linux-u-boot-legacy-nanopi-r6s/u-boot.itb` (FIT
  (=Flattened uImage Tree) image containing various firmwares and the
  main U-Boot) of the `linux-u-boot-nanopi-r6s-legacy` package.  They
  are installed via `/usr/lib/u-boot/platform_install.sh` of the same
  package, which copies them to offsets 64×512=0x8000 and
  16384×512=0x800000 of the eMMC: the files appeared untouched.  Run
  `dtc /usr/lib/linux-u-boot-legacy-nanopi-r6s/u-boot.itb` to see more
  of the content of the `u-boot.itb` file (add 0x800 to the
  `data-offset` values to get the offset values inside the file).

* Offset 0x8000 of the eMMC is probably the main entry point.  So my
  understanding of the boot process is that `idbloader.img` (as copied
  at that point) loads the various parts of `u-boot.itb` (as copied at
  fixed address 0x800000) and U-Boot proceeds to search for `boot.scr`
  in some way.

* At offset 7168×512=0x380000 of the eMMC is some kind of data
  segment?  Where does it come from?  It has the MAC addresses at
  offset 0x400 (i.e., 0x380400 of the whole).

* Somehow the U-Boot manages to read `/boot/boot.scr` (or `/boot.scr`,
  as the case may be).  The latter was probably generated at some
  point from `/boot/boot.cmd` by `mkimage -C none -A arm -T script -d
  /boot/boot.cmd /boot/boot.scr` as commented in the last line of the
  latter, but I was unable to find whence (since `boot.cmd` is not
  supposed to be modified, this is probably not run as a matter of
  course).

* The Armbian-specific `/boot/armbianEnv.txt` is read from `boot.scr`
  to get extra environment variables.

* The Armbian-specific `boot.scr` loads various FDT files, mainly
  `dtb/${fdtfile}` (which resolves to
  `/boot/dtb/rockchip/rk3588s-nanopi-r6s.dtb`) and possibly other
  overlays, including user-specified ones.

* To install a user overlay, run `armbian-add-overlay my-overlay.dts`
  which will compile it (using `dtc -@ -q -I dts -O dtb`) to a `.dtbo`
  file and place the latter in `/boot/overlay-user` and edit
  `armbianEnv.txt` to instruct `boot.scr` to fetch it.  (See
  <https://docs.armbian.com/User-Guide_Allwinner_overlays/> for
  overlays in general.)

* Recall that one can inspect the content of a DTB file by running
  `dtc`, e.g. `dtc -q /boot/dtb/rockchip/rk3588s-nanopi-r6s.dtb` or
  `dtc -q -I fs /proc/device-tree` for the one received by the kernel.

## Trying to set up MAC addresses correctly ##

* So, *if I understand correctly*, U-Boot uses the aliases found in
  `/aliases` in the DTB as `ethernet0`, `ethernet1`, `ethernet2`,
  etc. (which should be system paths to the device) and *patches* the
  `local-mac-address` property of the corresponding device (e.g.,
  `/ethernet@fe1c0000`) to the parameters `ethaddr`, `eth1addr`,
  `eth2addr`, etc., which it got in its environment.  (See
  <https://forum.armbian.com/topic/14525-mac-address-of-eth0-changes-on-every-boot/>
  on this topic.)

* Except that the Armbian U-Boot doesn't seem to obey the overrides in
  `armbianEnv.txt` for MAC addresses, but it seems that we can
  monkey-patch at addresses 0x380400 etc. of the eMMC (see earlier
  about them).

* So, concretely, read the U-Boot stored MAC addresses with `sudo dd
  if=/dev/mmcblk0 bs=1 skip=$((0x380400)) count=18 | hd` and write
  them with something like `perl -e 'print(pack("H*",
  "f2:90:cd:36:82:c8 f2:90:cd:36:82:cc f2:90:cd:36:82:c9" =~
  s/[:\x20\n]//gr));' | sudo dd of=/dev/mmcblk0 bs=1
  seek=$((0x380400)) count=18` (where, again, the addresses are given
  in order of `ethernet0`, `ethernet1`, `ethernet2`, as found in
  `/aliases` in the — overlaid — DTB).  Except that only the one for
  `/ethernet@fe1c0000` seems to actually have any effect.

* But `/ethernet@fe1c0000` seems to have its own problems, at least
  with the `5.10.160-legacy-rk35xx` kernel from Armbian 23.8.2 (driver
  sometimes fails with error message: `rk_gmac-dwmac
  fe1c0000.ethernet: Failed to reset the dma`: see
  <https://www.t95plus.com/forums/index.php?threads/ethernet-not-working-jl2101.11/>
  for what may be the cause); this is not too reproducible, and I may
  have done something Wrong to cause this.

## Miscellaneous notes ##

A possible `/etc/udev/rules.d/70-persistent-net.rules` file:

    # Written by David A. Madore to try to give sensible names to devices
    
    # Avoid name conflicts
    # SUBSYSTEM=="net", KERNEL=="eth0", NAME="eth0.torename"
    # SUBSYSTEM=="net", KERNEL=="eth1", NAME="eth1.torename"
    # SUBSYSTEM=="net", KERNEL=="eth2", NAME="eth2.torename"
    
    # /devices/platform/fe1c0000.ethernet
    # This one is labeled "LAN2" and is on the left when facing the back
    SUBSYSTEM=="net", DRIVERS=="?*", KERNELS=="fe1c0000.ethernet", NAME="ethlan2"
    
    # /devices/platform/fe190000.pcie/pci0004:40/0004:40:00.0/0004:41:00.0
    # This one is labeled "LAN1" and is in the middle when facing the back
    SUBSYSTEM=="net", DRIVERS=="r8125", KERNELS=="0004:41:00.0", NAME="ethlan1"
    
    # /devices/platform/fe180000.pcie/pci0003:30/0003:30:00.0/0003:31:00.0
    # This one is labeled "WAN" and is on the right when facing the back
    SUBSYSTEM=="net", DRIVERS=="r8125", KERNELS=="0003:31:00.0", NAME="ethwan"

Alternative link files to use as
`/etc/systemd/network/10-persistent-net-*.link` (not tested):

    # /etc/systemd/network/10-persistent-net-ethlan2.link
	[Match]
	Path=platform-fe1c0000.ethernet
	
	[Link]
	Name=ethlan2

    # /etc/systemd/network/10-persistent-net-ethlan1.link
	[Match]
	Path=platform-fe190000.pcie-pci-0004:41:00.0
	
	[Link]
	Name=ethlan1

    # /etc/systemd/network/10-persistent-net-ethwan.link
    [Match]
    Path=platform-fe180000.pcie-pci-0003:31:00.0
    
    [Link]
    Name=ethwan

A file `rk3588s-nanopi-r6s-fix-ethernet.dts` that adds an alias
`ethernet2` in the DTB for the missing Ethernet adapter (the one
labeled "LAN1", see above):

    /dts-v1/;
    /plugin/;
    
    /* On Armbian, run `armbian-add-overlay rk3588s-nanopi-r6s-fix-ethernet.dts` */
    
    / {
    	compatible = "friendlyelec,nanopi-r6s";
    
    	fragment@0 {
    		target-path = "/aliases";
    		__overlay__ {
    			ethernet1 = "/ethernet@fe1c0000";  // In stock DTB
    			ethernet2 = "/pcie@fe190000/pcie@40/pcie@40,0";
    			ethernet0 = "/pcie@fe180000/pcie@30/pcie@30,0";  // In stock DTB
    		};
    	};
    
    	/* This one is labeled "LAN2" and is on the left when facing the back */
    	/* It is aliased as ethernet1 */
    	fragment@1 {
    		target-path = "/ethernet@fe1c0000";  /* target = <&gmac1>; */
    		__overlay__ {
    			local-mac-address = [00 00 00 00 00 00];
    		};
    	};
    
    	/* This one is labeled "LAN1" and is in the middle when facing the back */
    	/* It is aliased as ethernet2 */
    	fragment@2 {
    		target-path = "/pcie@fe190000/pcie@40/pcie@40,0";  /* target = <&r8125_u40>; */
    		__overlay__ {
    			local-mac-address = [00 00 00 00 00 00];
    		};
    	};
    
    	/* This one is labeled "WAN" and is on the right when facing the back */
    	/* It is aliased as ethernet0 */
    	fragment@3 {
    		target-path = "/pcie@fe180000/pcie@30/pcie@30,0";  /* target = <&r8125_u25>; */
    		__overlay__ {
    			local-mac-address = [00 00 00 00 00 00];
    		};
    	};
    };

