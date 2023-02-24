<h1 align="center">VoodooInjector</h1>
<br>
<p align="center">
  A <a href="https://github.com/VoodooI2C/VoodooI2C">VoodooI2C</a> injector for the <b>ASUS ZenBook Duo 14"</b> and <b>Asus Zenbook Pro Duo 15"</b> laptops.
</p>

<div align="center">

  <a href="/LICENSE">![License](https://img.shields.io/badge/⚖_License-BSD_3_Clause-lightblue?labelColor=3f4551)</a>
  <a href="">![Supported Models](https://img.shields.io/badge/Supported%20Models-UX481FA/FL%20|%20UX581GV/LV%20|%20UX582LV-important?labelColor=3f4551)</a>
  <a href="https://github.com/Qonfused/VoodooInjector/actions/workflows/build.yml">![XCode Build](https://github.com/Qonfused/VoodooInjector/actions/workflows/build.yml/badge.svg?branch=main)</a>

</div>

This kext resolves an architecture incompatibility in VoodooI2CHID applicable to
laptops with multiple I2C-HID devices (e.g. trackpad/screenpad and touchscreen).
While this kext is primarily focused on supporting ASUS Zenbook Duo laptops, it
is also applicable to other ASUS screenpad devices.

## Background

VoodooI2CHID comes with multiple types of IOKit personalities or drivers (e.g. Generic Mouse, Precision Touchpad, Stylus, Touchscreen, etc). These drivers attach to a device based on a IOProbeScore property for each event driver in the kext's Info.plist, which sets the priority score for ordering which IOKit personalities attach to I2C-HID devices first. The precision trackpad event driver can be configured to attach first before any other event drivers, enabling multi-touch gestures on the trackpad by default.

This works fine when addressing a single I2C-HID device like a trackpad, as the precision touchpad event driver can attach first using a higher IOProbeScore. But when dealing with additional I2C-HID devices like touchscreen, there exists no explicit control of which event drivers attach to which devices. Overriding priority with a higher IOProbeScore cannot configure which event drivers attach to which devices as changes are applied equally. This can cause issues as the touchscreen event driver is prevented from attaching to the touchscreen device before the touchpad HID event driver takes hold, and vice versa for the trackpad.

The workaround is to edit the Info.plist for the VoodooI2CHID kext to force specific drivers to match specific devices based on their product and vendor IDs. By default, most event drivers' `IOPropertyMatch` keys specify the event driver to attach to any device using the I2C transport, but they can also be configured to match only a specific device by matching against it's specific product ID or vendor ID. This only allows for a match for a single device, but for multiple devices you can use the `IOPCIMatch` key instead.

## Gathering device info

### Windows Instructions

#### Finding device identifiers

If you're unfamiliar with what I2C HID devices your system has, you'll want to
follow these instructions first, and then proceed with the macOS instructions.

First open Device Manager and look for any `I2C HID Device` entries under 'Human
Interface Devices'. For each entry, right click on it and select 'properties'.
Then navigate to the 'Details' tab and select the 'Hardware Ids' property from
the dropdown:

I2C HID Device | Associated I2C Device
--- | ---
![screenshot-0](https://user-images.githubusercontent.com/32466081/210290786-a9df1199-96d3-411d-8d15-b7b1c110474d.png) | ![screenshot-1](https://user-images.githubusercontent.com/32466081/210291005-d670b023-0d1a-42ac-977c-4b048f00dae0.png)

^ Here we see the identifier `ELAN1207` is associated with the trackpad, which
we can verify by repeating the same step with the trackpad device.

You'll need to repeat this step checking other devices for identifying the
remaining I2C HID devices, though they are usually listed as trackpad or
touchscreen devices. Look for any `HID-compliant trackpad` or
`HID-compliant touchscreen` entries first.

For this example, repeating this step yields the following information:

(Device) | Identifier
--- | ---
Trackpad | `ELAN1207`
Touchscreen - Primary Display | `ELAN9008`
Touchscreen - Screenpad Plus | `ELAN9009`

### macOS Instructions

#### Finding product and vendor ids

You'll first need to grab your devices' product and vendor ids by running the
below command in Terminal:

```bash
ioreg -rxn IOHIDInterface -k "VoodooI2CServices Supported" -t | grep -E 'IOACPIPlatformDevice|IOPCIDevice|VoodooI2CDeviceNub|VendorID|ProductID|HIDEventDriver  <class VoodooI2C|Interface  <class VoodooI2C' | cut -d "<" -f1
```

Example output:
```bash
$ ioreg -rxn IOHIDInterface -k "VoodooI2CServices Supported" -t | grep -E 'IOACPIPlatformDevice|IOPCIDevice|VoodooI2CDeviceNub|VendorID|ProductID|HIDEventDriver  <class VoodooI2C|Interface  <class VoodooI2C' | cut -d "<" -f1
>>>      +-o PCI0@0  
>>>          +-o I2C0@15  
>>>                  +-o TPL1  
>>>                        |   "VendorID" = 0x4f3
>>>                        |   "ProductID" = 0x2b6a
>>>                        +-o VoodooI2CTouchscreenHIDEventDriver  
>>>                          +-o VoodooI2CMultitouchInterface  
>>>      +-o PCI0@0  
>>>          +-o I2C1@15,1  
>>>                  +-o ETPD  
>>>                        |   "VendorID" = 0x4f3
>>>                        |   "ProductID" = 0x310e
>>>                        +-o VoodooI2CPrecisionTouchpadHIDEventDriver  
>>>                          +-o VoodooI2CMultitouchInterface  
>>>      +-o PCI0@0  
>>>          +-o I2C3@15,3  
>>>                  +-o TPL0  
>>>                        |   "VendorID" = 0x4f3
>>>                        |   "ProductID" = 0x29de
>>>                        +-o VoodooI2CTouchscreenHIDEventDriver  
>>>                          +-o VoodooI2CMultitouchInterface
```
^ This output also shows which event drivers and interfaces are attached to each
device. You can use this command again later to double-check that the correct
event drivers are being attached to the correct devices.

#### Checking device identifiers

You may also wish to verify the devices' identifiers for identifying which
device is which. Use the device's ACPI name (e.g. `ETPD`, `TPL0`, `TPL1`) from
the previous command's output to run the following command:
```sh
ioreg -rxn <name> -k i2cAddress -d1 | grep IOName | cut -d "=" -f2
```
Example output:

```sh
$ ioreg -rxn ETPD -k i2cAddress -d1 | grep IOName | cut -d "=" -f2 
>>> "ELAN1207"
$ ioreg -rxn TPL1 -k i2cAddress -d1 | grep IOName | cut -d "=" -f2 
>>> "ELAN9008"
$ ioreg -rxn TPL0 -k i2cAddress -d1 | grep IOName | cut -d "=" -f2 
>>> "ELAN9009"
```

For this example, we now have the following information for each of our I2C
devices:

(Device) | Identifier | ACPI Path | Vendor ID (VID) | Product ID (PID)
--- | --- | --- | --- | ---
Trackpad | `ELAN1207` | `\_SB.PCI0.I2C1.ETPD` | `04F3` | `310E`
Touchscreen - Primary Display | `ELAN9008` | `\_SB.PCI0.I2C0.TPL1` | `04F3` | `2B6A`
Touchscreen - Screenpad Plus | `ELAN9009` | `\_SB.PCI0.I2C3.TPL0` | `04F3` | `29DE`

Once you have the vendor and product id for each device, proceed with the next
step.

## Modifying VoodooI2CHID IOKitPersonalities

For each event driver of interest, you'll want to make either of the two changes
below. In our example for the Zenbook Duo, we use both the `IOPropertyMatch` and
`IOPCIMatch` keys to match against a single device and multiple devices
respectively.

### Single Device

In the case of the trackpad device in our example, we use the `ProductId` key
under `IOPropertyMatch` to specify our trackpad's product id. You'll want to
convert the hex form of the product id to decimal (e.g. `310E` -> 12545) and
apply the below changes:

```xml
<!-- VoodooI2CHID.kext/Contents/Info.plist
... Root > IOKitPersonalities > VoodooI2CHIDDevice Precision Touchpad HID Event Driver
301      | <key>IOClass</key>
302      | <string>VoodooI2CPrecisionTouchpadHIDEventDriver</string>
303      | <key>IOProbeScore</key>
304      | <integer>300</integer>
305      | <key>IOPropertyMatch</key>
306      | <dict> -->
[-] 307  | <key>Transport</key>
[-] 308  | <string>I2C</string>
[+] 307  | <key>ProductID</key>
[+] 308  | <integer>12558</integer>
<!-- VID=310E       ^^^^^ (base 10 form)
309      | <\dict> -->
```

### Multiple devices

In the case of the touchscreen devices in our example, we use the `IOPCIMatch`
key for multiple values. The structure for each value is `0x<product-id><vendor id>`
(e.g. PID=`2C56` and VID=`04f3` -> `0x2C5604f3`), where each consecutive value
is joined by `&amp;`. You'll want to combine these values for all appropriate
devices and apply the below changes:

```xml
<!-- VoodooI2CHID.kext/Contents/Info.plist
... Root > IOKitPersonalities > VoodooI2CHIDDevice Touchscreen HID Event Driver
354      | <key>IOClass</key>
355      | <string>VoodooI2CTouchscreenHIDEventDriver</string>
356      | <key>IOProbeScore</key>
357      | <integer>400</integer> -->
[+] 358  | <key>IOPCIMatch</key>
[+] 359  | <string>0x2C5604f3&amp;0x2C2304f3</string>
<!-- PID=2C56,2C23   ^^^^           ^^^^          -->
<!-- VID=04f3            ^^^^           ^^^^      -->
```

You may want to also remove any existing `IOPropertyMatch` keys if they exist:
```xml
<!-- VoodooI2CHID.kext/Contents/Info.plist
... Root > IOKitPersonalities > VoodooI2CHIDDevice Touchscreen HID Event Driver
354      | <key>IOClass</key>
355      | <string>VoodooI2CTouchscreenHIDEventDriver</string>
356      | <key>IOProbeScore</key>
357      | <integer>300</integer> -->
[-] 358  | <key>IOPropertyMatch</key>
[-] 359  | <dict>
[-] 360  | <key>Transport</key>
[-] 361  | <string>I2C</string>
[-] 362  | </dict>
```

## Adding new IOKitPersonalities to VoodooInjector

To add new IOKitPersonalities to VoodooInjector, you can simply copy the entry
you've modified in VoodooI2CHID and add it as a new entry under VoodooInjector's
IOKitPersonalities:
```xml
<!-- VoodooInjector.kext/Contents/Info.plist
... Root > IOKitPersonalities
45       | <key>ELAN1207 Precision Touchpad HID Event Driver</key>
46-74    | <dict>...</dict> -->
[+] 75   | <key>My custom IOKitPersonality</key>
[+] 76+  | <dict>...</dict>
```

You'll need to preserve the same `CFBundleIdentifier` and `IOClass` properties
from the original entry to match to the corresponding VoodooI2CHID driver.

## Resources

- Refer to [VoodooI2C#474 issue comment](https://github.com/VoodooI2C/VoodooI2C/issues/474#issuecomment-966665616) from [gvkt](https://github.com/gvkt) and [VoodooI2CHID#59 PR](https://github.com/VoodooI2C/VoodooI2CHID/pull/59) describing architectural issues in more depth.
- Refer to [VoodooI2C#485 issue](https://github.com/VoodooI2C/VoodooI2C/issues/485) for a basic explanation of VoodooI2C's IOKit matching behavior.
- Refer to [ASUS-ZenBook-Duo-14-UX481-Hackintosh#12 issue](https://github.com/Qonfused/ASUS-ZenBook-Duo-14-UX481-Hackintosh/issues/12) in regards to the Info.plist changes alongside GPIO pinning (changes are to match drivers to device-specific product ids rather than device names).

## ⚖️ License
[BSD 3-Clause License](/LICENSE).

## 🌟 Credits
- [@gvkt](https://github.com/gvkt), and [@1Revenger1](https://github.com/1Revenger1/) for their work on better supporting multi-touchscreen devices with VoodooI2CHID.
