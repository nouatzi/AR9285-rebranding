# Wireless card rebranding
The idea here is to use a Atheros AR9285 wireless card, and make it look like a Intel Centrino Advanced-N 6205 Wireless card.  
We do this by modifying the EEPROM card. We copy the vendor/device/subsystem IDs of the Intel card into the EEPROM Atheros card.

My setup:
 - Lenovo Thinkpad X230 with BIOS whitelist (v2.67), and MacOS Sierra and its original Centrino Advanced-N 6205 wireless card.
 - Spared Laptop with no restriction, and Ubuntu 16.04.2, and Atheros AR9285 wireless card.

From now on, we'll use the spared laptop.  
First we need to identify both wireless cards. We start with the Centrino Advanced-N 6205 by putting it inside the spared laptop.  
Then, in terminal:  
`lspci`  
This will give us the pci slot used by the cards, for me: 03:00.0  
Then we take a look at:  
`ls /sys/bus/pci/devices/`  
And we look for the same pci slot, for me: 03:00.0  
And then we use:  
`udevadm info /sys/bus/pci/devices/0000:03:00.0`  
Result:
```
P: /devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DEVPATH=/devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DRIVER=iwlwifi
E: ID_MODEL_FROM_DATABASE=Centrino Advanced-N 6205 [Taylor Peak] (Centrino Advanced-N 6205 AGN)
E: ID_PCI_CLASS_FROM_DATABASE=Network controller
E: ID_PCI_SUBCLASS_FROM_DATABASE=Network controller
E: ID_VENDOR_FROM_DATABASE=Intel Corporation
E: MODALIAS=pci:v00008086d00000085sv00008086sd00001311bc02sc80i00
E: PCI_CLASS=28000
E: PCI_ID=8086:0085
E: PCI_SLOT_NAME=0000:03:00.0
E: PCI_SUBSYS_ID=8086:1311
E: SUBSYSTEM=pci
E: USEC_INITIALIZED=6291525
```
This command give us enough informations about the card. We are interested in the PCI_ID and PCI_SUBSYS_ID. We'll modify both of them.  
We save those informations somewhere, remove the N6205 card, and put back the AR9285 inside the spared laptop, and we repeat the process:  
Result:
```
P: /devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DEVPATH=/devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DRIVER=ath9k
E: ID_MODEL_FROM_DATABASE=AR9285 Wireless Network Adapter (PCI-Express) (AW-NE785 / AW-NE785H 802.11bgn Wireless Full or Half-size Mini PCIe Card)
E: ID_PCI_CLASS_FROM_DATABASE=Network controller
E: ID_PCI_SUBCLASS_FROM_DATABASE=Network controller
E: ID_VENDOR_FROM_DATABASE=Qualcomm Atheros
E: MODALIAS=pci:v0000168Cd0000002Bsv00001A3Bsd00001089bc02sc80i00
E: PCI_CLASS=28000
E: PCI_ID=168C:002B
E: PCI_SLOT_NAME=0000:03:00.0
E: PCI_SUBSYS_ID=1A3B:1089
E: SUBSYSTEM=pci
E: USEC_INITIALIZED=6046558
```
We save those informations somewhere.  
So finally we have:  
- PCI_ID 168C:002B will become 8086:0085  
- PCI_SUBSYS_ID 1A3B:1089 will become 8086:1311  

Now we will dump the AR9285 EEPROM into a file, modify that file(to make it look like the N6205), and copy it back to the wireless card.  
We need a few tools.  
`sudo apt-get install build-essential ghex`  
ghex is a hexadecimal editor.  
Then, we download the tool source code that allows us to read and write EEPROMs, iwleeprom:  
```
wget https://storage.googleapis.com/google-code-archive-source/v2/code.google.com/iwleeprom/source-archive.zip
unzip source-archive.zip
cd iwleeprom/branches/atheros
```
Then we need to modify a bit of code inside ath9kio.c, around line 795:  
```
if (dev->ops->eeprom_read16(dev, 128, &data) && (376 == data)) {
            short_eeprom_base = 128;
            short_eeprom_size = 376;
            goto ssize_ok;
}
```
change:  
```
            short_eeprom_base = 128;
            short_eeprom_size = 376;
```
into:  
```
            short_eeprom_base = 0;
            short_eeprom_size = 512;
```
save, and compile:  
`make`  

Now make sure this is the AR9285 card that we are actually using.
We'll dump the AR9285 EEPROM into a file:  
`sudo ./iwleeprom -o ./AR9285-original.eeprom`  
Result:  
```
Supported devices detected:
  [1] 0000:03:00.0 [RW] AR9285 Wireless Adapter (PCI-E) (168c:002b, 1a3b:1089)
Select device [1-1] (or 0 to quit): 1
Using device 0000:03:00.0 [RW] AR9285 Wireless Adapter (PCI-E)
IO driver: ath9k
HW: AR9285 (PCI-E) rev 0002
RF: integrated
Checking NVM size...
ath9k short eeprom base: 0  size: 512
Saving dump with byte order: LITTLE ENDIAN
0000 [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]
0f80 [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]

EEPROM has been dumped to './AR9285-original.eeprom'
```

The file is owned by the root user, but you can change it back to a normal user owner `(chown)...`

We keep the original intact, and we'll work on a copy:  
`cp AR9285-original.eeprom AR9285-patched.eeprom`  

Now it's time to open the eeprom dump file with ghex, find and replace vendor/device/subsystem IDs:  
`ghex AR9285-patched.eeprom`  
In the menu `"View" --> "Group Data As" --> "Words"`, it's just a preference.  

Now something IMPORTANT, the eeprom dump file is byte-flipped, which means, if we want to look for "168C", we will look for "8C16".  
So from:  
- PCI_ID 168C:002B  will become 8086:0085
- PCI_SUBSYS_ID 1A3B:1089 will become 8086:1311

We obtain:  
- PCI_ID 8C16:2B00  will become 8680:8500
- PCI_SUBSYS_ID 3B1A:8910 will become 8680:1113

And a bit more clearly:  
- find 8C162B00       replace by 86808500
- find 3B1A8910       replace by 86801113

Don't forget, there are 2 OCCURENCES of each to find and replace by.  
Then save file.  

And now we write back the modified eeprom dump file into the wireless card, using iwleeprom tool:  
`sudo ./iwleeprom -i AR9285-patched.eeprom`  

Now our AR9285 card is patched and ready. We put it back in the Thinkpad X230, and boot it. You should not see any warning from the BIOS.  

However, the process is not finished. We've bypass the X230 BIOS. But now the AR9285 card is seen as an Intel N6205 card by Mac OS.  
So there is the process of put the right IDs back, by using [FakePCIID kexts from Rehabman](https://github.com/RehabMan/OS-X-Fake-PCI-ID).  

If you have multi OSs boot, your rebranded wireless card won't work on any of those other OSs without any equivalent solution like FakePCIID.

If for some reasons, we want to change the IDs again, or we want to simply restore the original EEPROM, we cannot just write it back with iwleeprom. Because the card is recognized neither by the system nor by iwleeprom (it's identified as Intel but it's not). So we'll need to:
- unload the intel module, which is trying to work with the fake intel card,
- download the linux kernel, 
- modify the atheros module source code by puting the intel IDs,  
- build, and launch the modify kernel module,
- modify iwleeprom source code by puting the intel IDs,
- build, and now use iwleeprom to write back the original EEPROM.

So here the process:
First we need to unload Intel module, which is loaded because of our fake Intel ID inside our Atheros card:
`sudo modprobe -r iwldvm`

Next we download the Ubuntu Linux kernel source code:
`apt source linux-image-$(uname -r)`

It'll give us a source code folder, so get into it:
`cd linux-xxx`

Then we copy our actual config:
`cp /boot/config-$(uname -r) .config`

Then we prepare some kernel files:
`make prepare
make scripts`

Then we go into the atheros module folder:
`cd drivers/net/wireless/ath/ath9k/`

And into that folder, there will be 2 source files to modify:
- hw.h
- and pci.c

Let's start with hw.h, and look for:
`#define AR9285_DEVID_PCIE         0x002b`
and we replace it with the fake intel ID we're actually using, with is 85:
`#define AR9285_DEVID_PCIE         0x0085`

Next we go for pci.c, and look for:
```
{ PCI_VDEVICE(ATHEROS, 0x002B) }, /* PCI-E */
{ PCI_VDEVICE(ATHEROS, 0x002C) }, /* PCI-E 802.11n bonded out */
{ PCI_VDEVICE(ATHEROS, 0x002D) }, /* PCI   */
{ PCI_VDEVICE(ATHEROS, 0x002E) }, /* PCI-E */
```
And we add a new line with our fake Intel IDs too:
`{ PCI_VDEVICE(INTEL, 0x0085) },`
Don't forget the change `ATHEROS` into `INTEL`.

Now we're ready to compile to modified atheros module, with:
`make -C /lib/modules/$(uname -r)/build M=$(pwd) modules`
And then, we're ready to install the modified atheros module:
`sudo make -C /lib/modules/$(uname -r)/build M=$(pwd) modules_install`
There might be some sign errors but that should not be a problem.

This will install the modified module into an extra folder: `/lib/modules/(your kernel version)/extra`, this way, it won't overwrite any original atheros module.

However, we'll need to temporary disable the original atheros so they won't launch.
So we go inside the original atheros module folder, and we rename them:
`cd /lib/modules/$(uname -r)/kernel/drivers/net/wireless/ath/ath9k/`

There will be 4 module files:
```
ath9k_common.ko
ath9k_htc.ko
ath9k_hw.ko
ath9k.ko
```
We rename them into (by using `sudo mv ...`):
```
ath9k_common.ko.backup
ath9k_htc.ko.backup
ath9k_hw.ko.backup
ath9k.ko.backup
```

Now it's time to launch the modified atheros module:
```
sudo depmod
sudo modprobe -v ath9k
```
The atheros card should be up.

Now it's time to modify iwleeprom source code. The idea is pretty much the same: we put the wrong IDs in the source code, so that iwleeprom will recognize the card.
But first we need to modify `iwlio.c`. Because right now the card is seen as a real Intel card, so we look for:
```
/* Intel 6x00/6x50 devices */
const struct pci_id iwl6k_ids[] = {
{ INTEL_PCI_VID,   0x0082, "6000 Series Gen2 (6x05)"},
{ INTEL_PCI_VID,   0x0083, "Centrino Wireless-N 1000"},
{ INTEL_PCI_VID,   0x0084, "Centrino Wireless-N 1000"},
{ INTEL_PCI_VID,   0x0085, "6000 Series Gen2 (6x05)"},
{ INTEL_PCI_VID,   0x0087, "Centrino Advanced-N + WiMAX 6250"},
```
And we comment the line with the ID 85 (which is our wrong ID):

`/*{ INTEL_PCI_VID,   0x0085, "6000 Series Gen2 (6x05)"},*/`

That way, iwleeprom won't see it as a real Intel card.

Then we modify ath9kio.c, by looking for atheros IDs:
```
/* Atheros 9k devices */
const struct pci_id ath9k_ids[] = {
{ ATHEROS_PCI_VID, 0x0023, "AR5416 (AR5008 family) Wireless Adapter (PCI)" },
{ ATHEROS_PCI_VID, 0x0024, "AR5416 (AR5008 family) Wireless Adapter (PCI-E)" },
{ ATHEROS_PCI_VID, 0x0027, "AR9160 802.11abgn Wireless Adapter (PCI)" },
{ ATHEROS_PCI_VID, 0x0029, "AR922X Wireless Adapter (PCI)" },
{ ATHEROS_PCI_VID, 0x002A, "AR928X Wireless Adapter (PCI-E)" },
{ ATHEROS_PCI_VID, 0x002B, "AR9285 Wireless Adapter (PCI-E)" },
{ ATHEROS_PCI_VID, 0x002C, "AR2427 Wireless Adapter (PCI-E)" }, /* PCI-E 802.11n bonded out */
{ ATHEROS_PCI_VID, 0x002D, "AR9287 Wireless Adapter (PCI)" },
{ ATHEROS_PCI_VID, 0x002E, "AR9287 Wireless Adapter (PCI-E)" },
{ 0, 0, "" }
```

And we add a line with our fake Intel ID:

`{ INTEL_PCI_VID,   0x0085, "My wrong ID Wireless Adapter (PCI)" },`

for the `INTEL_PCI_VID` word to be recognized, we add:
`#define INTEL_PCI_VID       0x8086` at the begining of file ath9kio.c

Now we compile iwleeprom `make` and we are ready to read or write the eeprom again. When we launch iwleeprom we should see `My wrong ID Wireless Adapter (PCI)` as choice of card.

When everything is done, don't forget to:
- restore the orginal atheros modules we've backed up. 
- remove the modified atheros modules from the `extra` folder. 
- launch `sudo depmod` 
