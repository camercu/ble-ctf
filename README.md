# Bluetooth LE CTFs
[BLE CTF Github](https://github.com/hackgnar/ble_ctf)

[Advanced BLE CTF Github](https://github.com/hackgnar/ble_ctf_infinity)

The CTFs were part of the Workshops at DEF CON 29 (2021). To do the challenges, you need to have an ESP32 board with the CTF firmware on it. You also need a Linux box with the Bluez tools installed.

These are my notes for the CTFs.

# Getting Set Up
See the Github repos for instructions on installing the firmware (if required).

```bash
# Enable BLE on Linux
$ sudo rfkill unblock all
$ sudo btmgmt le on

# Check that you have a bluetooth antenna device
$ hcitool dev
Devices:
	hci0	B8:27:EB:76:11:25

# if necessary, restart bluetooth service
$ sudo service bluetooth restart
$ sudo service dbus restart

# disable/enable bluetooth device
$ sudo hciconfig hci0 down
$ sudo hciconfig hci0 up

# finding the MAC of your device
$ sudo hcitool lescan
LE Scan ...
08:3A:F2:AA:12:AA BLECTF
--- snip ---

# you can specify the interface to scan on:
$ sudo hcitool -i hci0 lescan

# how to check your score
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x002a|awk -F':' '{print $2}'|tr -d ' '|xxd -r -p;printf '\n'

# saving time checking score with an alias
alias score="gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x002a|awk -F':' '{print \$2}'|tr -d ' '|xxd -r -p;printf '\n'"
```

## Submitting Flags

All flags are MD5 sums truncated to 20 characters to avoid MTU limits of some hardware. They can be submitted to the gatt server on handle 44. Example of submission:

```bash
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x002c -n $(echo -n "some flag value"|xxd -ps)
```

# Flag 1 - Gimme
This flag is provided by the [hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag1.md) just to make sure you can use the tools.

```bash
# first, check your score
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x002a|awk -F':' '{print $2}'|tr -d ' '|xxd -r -p;printf '\n'
Score: 0/20

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x002c -n $(echo -n "12345678901234567890"|xxd -ps)
Characteristic value was written successfully

# check score again, verify you got the points
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x002a|awk -F':' '{print $2}'|tr -d ' '|xxd -r -p;printf '\n'
Score:1 /20
```


# Flag 2 - 0x002e - Learn to Read Handles

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag2.md)

Check out the ascii value of handle `0x002e` and submit it to the flag submission handle `0x002c`. If you are using `gatttool`, make sure you convert it to hex with `xxd`. If you are using `bleah`, you can send it as a string value.

You read values using `--char-read` and specify the handle address with `-a`:
```bash
# perform read
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x002e | cut -d':' -f2 | xxd -r -p; echo
d205303e099ceff44835

# write value to submission handle. Note need to convert back to hex
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n d205303e099ceff44835| xxd -ps)
Characteristic value was written successfully

# check score using alias
$ score
Score:2 /20
```

# Flag 3 - 0x0030 - Read handle puzzle

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag3.md)

Check out the ascii value of handle `0x0030`. Do what it tells you and submit the flag you find to `0x002c`.

```bash
# read to get instructions
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x30 --char-read | cut -d':' -f2 | xxd -r -p; echo
MD5 of Device Name

# using device name from "hcitool lescan" from earlier
$ echo -n "BLECTF" | md5sum
5cd56d74049ae40f442ece036c6f4f06  -

# first try, using hex encoded MD5, failed
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 5cd56d74049ae40f442ece036c6f4f06 | xxd -ps)
Characteristic value was written successfully
$ score
Score:2 /20

# second try, using raw MD5, failed
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n 5cd56d74049ae40f442ece036c6f4f06
Characteristic value was written successfully
$ score
Score:2 /20
```

Reading up on Bluetooth Low Energy, found a wonderful introduction by [Adafruit](https://learn.adafruit.com/introduction-to-bluetooth-low-energy/). [This blog post](https://epxx.co/artigos/bluetooth_gatt.html) goes into a little more technical detail about the structure of GATT. [This blog post](https://cxiao.net/posts/2015-12-13-gatttool/) explains how to use `gatttool` for exploring a BLE device.

In summary, GATT is the way BLE peripherals offer services to clients (central nodes, like your phone). There are "Profiles", which are just standardized collections of Services. "Services" are standardized (or customized) collections of data points. The individual datapoints themselves are called "Characteristics". An example service is the "Device Information Service", which has a 16-bit UUID of 0x180A. It contains mandatory attributes like ProductID and VendorID, which each have their own Attribute IDs (16-bit UUIDs). The services and characteristics are accessed through "handles", which are like addresses that are used to read/write information from the server.

In BLE, UUIDs provide contextual information about a service or characteristic's handle. BLE uses a standard 128-bit UUID format:

```
0000XXXX-0000-1000-8000-00805f9b34fb
```
The only part that changes from one service/characteristic/etc. to another is the 16-bit UUID (the `XXXX` in the 128-bit UUID format above). For example, when first interacting with a device, you might want to know what primary services it offers. To see that, you'd run:

```bash
$ gatttool -b 08:3A:F2:AA:12:AA --primary
```

Which returns the following response for our BLECTF device:

```
attr handle = 0x0001, end grp handle = 0x0005 uuid: 00001801-0000-1000-8000-00805f9b34fb
attr handle = 0x0014, end grp handle = 0x001c uuid: 00001800-0000-1000-8000-00805f9b34fb
attr handle = 0x0028, end grp handle = 0xffff uuid: 000000ff-0000-1000-8000-00805f9b34fb
```

Looking at the `uuid` fields above, we see the standard 128-bit BLE UUID format, with three 16-bit UUIDs:
- 1801 (Generic Access)
- 1800 (Generic Attribute)
- 00ff (Custom Service)

How do we know what these UUIDs mean? We can look in the specification of [Assigned Numbers for BLE](https://btprodspecificationrefs.blob.core.windows.net/assigned-values/16-bit%20UUID%20Numbers%20Document.pdf)! The services 1800 and 1801 should be present for all BLE devices, because that's how you interact with them. The service `00ff` isn't in the list of assigned numbers, so that one's custom.

Next, we can see what characteristics are available for reading/writing on the server:

```bash
$ gatttool -b 08:3A:F2:AA:12:AA --characteristics
```

Which returns the following for our BLECTF device:

```
handle = 0x0002, char properties = 0x20, char value handle = 0x0003, uuid = 00002a05-0000-1000-8000-00805f9b34fb
handle = 0x0015, char properties = 0x02, char value handle = 0x0016, uuid = 00002a00-0000-1000-8000-00805f9b34fb
handle = 0x0017, char properties = 0x02, char value handle = 0x0018, uuid = 00002a01-0000-1000-8000-00805f9b34fb
handle = 0x0019, char properties = 0x02, char value handle = 0x001a, uuid = 00002aa6-0000-1000-8000-00805f9b34fb
handle = 0x0029, char properties = 0x02, char value handle = 0x002a, uuid = 0000ff01-0000-1000-8000-00805f9b34fb
handle = 0x002b, char properties = 0x0a, char value handle = 0x002c, uuid = 0000ff02-0000-1000-8000-00805f9b34fb
handle = 0x002d, char properties = 0x02, char value handle = 0x002e, uuid = 0000ff03-0000-1000-8000-00805f9b34fb
handle = 0x002f, char properties = 0x02, char value handle = 0x0030, uuid = 0000ff04-0000-1000-8000-00805f9b34fb
handle = 0x0031, char properties = 0x0a, char value handle = 0x0032, uuid = 0000ff05-0000-1000-8000-00805f9b34fb
handle = 0x0033, char properties = 0x0a, char value handle = 0x0034, uuid = 0000ff06-0000-1000-8000-00805f9b34fb
handle = 0x0035, char properties = 0x0a, char value handle = 0x0036, uuid = 0000ff07-0000-1000-8000-00805f9b34fb
handle = 0x0037, char properties = 0x02, char value handle = 0x0038, uuid = 0000ff08-0000-1000-8000-00805f9b34fb
handle = 0x0039, char properties = 0x08, char value handle = 0x003a, uuid = 0000ff09-0000-1000-8000-00805f9b34fb
handle = 0x003b, char properties = 0x0a, char value handle = 0x003c, uuid = 0000ff0a-0000-1000-8000-00805f9b34fb
handle = 0x003d, char properties = 0x02, char value handle = 0x003e, uuid = 0000ff0b-0000-1000-8000-00805f9b34fb
handle = 0x003f, char properties = 0x1a, char value handle = 0x0040, uuid = 0000ff0c-0000-1000-8000-00805f9b34fb
handle = 0x0041, char properties = 0x02, char value handle = 0x0042, uuid = 0000ff0d-0000-1000-8000-00805f9b34fb
handle = 0x0043, char properties = 0x2a, char value handle = 0x0044, uuid = 0000ff0e-0000-1000-8000-00805f9b34fb
handle = 0x0045, char properties = 0x1a, char value handle = 0x0046, uuid = 0000ff0f-0000-1000-8000-00805f9b34fb
handle = 0x0047, char properties = 0x02, char value handle = 0x0048, uuid = 0000ff10-0000-1000-8000-00805f9b34fb
handle = 0x0049, char properties = 0x2a, char value handle = 0x004a, uuid = 0000ff11-0000-1000-8000-00805f9b34fb
handle = 0x004b, char properties = 0x02, char value handle = 0x004c, uuid = 0000ff12-0000-1000-8000-00805f9b34fb
handle = 0x004d, char properties = 0x02, char value handle = 0x004e, uuid = 0000ff13-0000-1000-8000-00805f9b34fb
handle = 0x004f, char properties = 0x0a, char value handle = 0x0050, uuid = 0000ff14-0000-1000-8000-00805f9b34fb
handle = 0x0051, char properties = 0x0a, char value handle = 0x0052, uuid = 0000ff15-0000-1000-8000-00805f9b34fb
handle = 0x0053, char properties = 0x9b, char value handle = 0x0054, uuid = 0000ff16-0000-1000-8000-00805f9b34fb
handle = 0x0055, char properties = 0x02, char value handle = 0x0056, uuid = 0000ff17-0000-1000-8000-00805f9b34fb
```

Based on the output of `--primary` from before, we know that the custom service hosts all characteristics starting with handle `0x0028`. The handles below this are part of the standard services. Looking again at the list of assigned numbers, there is one UUID that catches my attention:

- `0x2A00` - Device Name (GATT Characteristic and Object Type)

I see that UUID provided in the list of characteristics:

```
handle = 0x0015, char properties = 0x02, char value handle = 0x0016, uuid = 00002a00-0000-1000-8000-00805f9b34fb
```

Note that it shows the full 128-bit UUID (0000***2a00***-0000-1000-8000-00805f9b34fb), but we only care about the last 4 characters of the set before the first dash.

So, maybe we can get the device name from that characteristic?

```bash
# read from the "char value handle" for the Device Name
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x0016 | cut -d':' -f2 | xxd -r -p; echo
2b00042f7481c7b056c4b410d28f33cf
```

Hmm, already looks like an MD5 hash? I'll try submitting both styles (raw and MD5 of that value):

```bash
# raw value, fails
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n 2b00042f7481c7b056c4b410d28f33cf

# hex-encoded raw value, fails
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 2b00042f7481c7b056c4b410d28f33cf | xxd -ps)

# hashed value, fails
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n 8489c638085eb7b7416e682af1dd5474

# hex encoded hashed value, fails
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 8489c638085eb7b7416e682af1dd5474 | xxd -ps)
```

Hmm. Got stumped for a long time on this one.

Eventually, I went back and re-read the [README](https://github.com/hackgnar/ble_ctf/blob/master/README.md) for the CTF, and found this little gem under `How to Submit Flags` that I'd overlooked before:

> Ok, ok, ok, on to the flags! All flags are md5 sums ***truncated to 20 characters*** to avoid MTU limits by some hardware.

Well, I feel dumb. It looks like the flag isn't the whole MD5, but only the first 20 characters. Looking it up online, it seems that most BLE hardware has an MTU of 20 bytes. To send more data, those systems can either negotiate a larger MTU or they can chunk the data and send it in multiple write-requests.

```sh
# correct solution: only using first 20 characters of MD5 sum
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n "BLECTF" | md5sum | awk '{print $1}' | head -c 20 | xxd -ps)
Characteristic value was written successfully

$ score
Score:3 /20
```



# Flag 4 - 0x0016 - Learn about discoverable device attributes

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag4.md)

Bluetooth GATT services provide some extra device attributes. Try finding the value of the Generic Access -> Device Name.

This seems to line up with what I stumbled on in the last challenge. To recap, "services" are collections of pieces of data (each data point is called a "characteristic") to accomplish a specific function or provide a feature of a device. According to the [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification-5-3/), the Generic Access service (or more formally, the "Generic Access Profile/GAP service") is a mandatory service for all GATT servers, and it has certain required characteristics, including Device Name. The [Bluetooth Assigned Numbers](https://www.bluetooth.com/specifications/assigned-numbers/) document specifies that the Device Name characteristic has the UUID `0x2A00` (see previous challenge for longer explanation on UUID formats).

First, we need to see what handle the Device Name characteristic is at by listing all the available characteristics.

```sh
$ gatttool -b 08:3A:F2:AA:12:AA --characteristics
```

And in the output, we see the Device Name characteristic, which is located at char value handle `0x0016`:

```sh
handle = 0x0015, char properties = 0x02, char value handle = 0x0016, uuid = 00002a00-0000-1000-8000-00805f9b34fb
```

Now to see what is in that handle:

```sh
# reading device name from its char value handle
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x0016
Characteristic value/descriptor: 32 62 30 30 30 34 32 66 37 34 38 31 63 37 62 30 35 36 63 34 62 34 31 30 64 32 38 66 33 33 63 66

# reading the value as ASCII
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x0016 | cut -d: -f2 | xxd -r -p
2b00042f7481c7b056c4b410d28f33cf
# oddly looks like a hash?

# submit the flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 2b00042f7481c7b056c4b410d28f33cf | head -c 20 | xxd -ps)
```




# Flag 5 - 0x0032 - Learn about reading and writing to handles

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag5.md)

Read handle 0032 and do what it says. Notice that its not telling you to write to the flag handle as you have been. When you find the flag, go ahead and write it to the flag handle you have used in the past flags.

```sh
# reading the instructions from the handle
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x0032 | cut -d: -f2 | xxd -r -p
Write anything here

# write random byte ('A') to handle 0x0032
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x0032 -n '41'
Characteristic value was written successfully

# read the handle again to see if it gave us a flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x0032 | cut -d: -f2 | xxd -r -p
3873c0270763568cf7aa

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 3873c0270763568cf7aa | xxd -p)
Characteristic value was written successfully
```




# Flag 6 - 0x0034 - Learn about reading and writing ascii to handles

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag6.md)

Follow the instructions found from reading handle 0x0034. Keep in mind that some tools only write hex values while other provide methods for writing either hex or ascii.

```sh
# get instructions from the handle
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x34 | cut -d: -f2 | xxd -r -p
Write the ascii value "yo" here

# writing "yo". Gatttool requires values to be entered as hex, so ascii must be converted
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x34 -n $(echo -n yo | xxd -p)
Characteristic value was written successfully

# get flag by re-reading handle
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x34 | cut -d: -f2 | xxd -r -p; echo
c55c6314b3db0a6128af

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n c55c6314b3db0a6128af | xxd -p)
Characteristic value was written successfully
```




# Flag 7 - 0x0036 - Learn about reading and writing hex to handles

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag7.md)

Follow the instructions found from reading handle 0x0036. Keep in mind that some tools only write hex values while other provide methods for writing either hex or ascii.

```sh
# read the instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x36 | cut -d: -f2 | xxd -p -r; echo
Write the hex value 0x07 here

# write hex value 0x07 to address 0x0036
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x36 -n 07
Characteristic value was written successfully

# re-read address 0x0036 to get flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x36 | cut -d: -f2 | xxd -p -r; echo
1179080b29f8da16ad66

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 1179080b29f8da16ad66 | xxd -p)Characteristic value was written successfully
```




# Flag 8 - 0x0038 - Learn about reading and writing to handles differently

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag8.md)

Follow the instructions found from reading handle 0x0038. Pay attention to handles here. Keep in mind handles can be refrenced by integer or hex. Most tools such as `gatttool` and `bleah` allow you to specify handles both ways.

```bash
# read handle to get instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x38 | cut -d: -f2 | xxd -p -r; echo
Write 0xC9 to handle 58

# write 0xC9 to handle 58 (note just using decimal form, not hex)
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 58 -n C9
Characteristic value was written successfully

# re-read to get flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x38 | cut -d: -f2 | xxd -p -r; echo
f8b136d937fad6a2be9f

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n f8b136d937fad6a2be9f | xxd -p)
Characteristic value was written successfully
```




# Flag 9 - 0x003c - Learn about write fuzzing

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag9.md)

Take a look at handle 0x003c and do what it says. You should script up a solution for this one. Also keep in mind that some tools write faster than others.

```bash
# read challenge instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x3c | cut -d: -f2 | xxd -r -p; echo
Brute force my value 00 to ff
```

Here's the script I wrote (`brute.sh`):

```bash
#!/bin/bash

for i in {0..255}; do
    gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x3c -n $(printf "%02X" "$i")
    flag="$(gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x3c | cut -d: -f2 | xxd -r -p)"
    if [[ "$flag" != "Brute force"* ]]; then
        printf "FLAG from 0x%X: $flag\n" $i
        exit 0
    fi
done

echo "Flag NOT FOUND!"
exit 1
```

Then running it to get the flag:

```bash
$ bash brute.sh
# --- snip ---
FLAG at 0xD1: 933c1fcfa8ed52d2ec05

# submit the flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 933c1fcfa8ed52d2ec05 | xxd -p)
Characteristic value was written successfully
```




# Flag 10 - 0x003e - Learn about read and write speeds

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag10.md)

Take a look at handle 0x003e and do what it says. Keep in mind that some tools have better connection speeds than other for doing reads and writes. This has to do with the functionality the tool provides or how it uses cached BT connections on the host OS. Try testing different tools for this flag. Once you find the fastest one, whip up a script or bash 1 liner to complete the task. FYI, once running, this task takes roughly 90 seconds to complete if done right.

```bash
# read instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x3e | cut -d: -f2 | xxd -r -p; echo
Read me 1000 times

# time a read-loop using gatttool
$ time for i in {1..1000}; do echo -n "$i: "; gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x3e | cut -d: -f2 | xxd -r -p; echo; done
# --- snip ---
999: Read me 1000 times
1000: 6ffcd214ffebdc0d069e

real	1m3.854s
user	0m24.088s
sys	0m8.640s

# Bleah's repo (https://github.com/evilsocket/bleah) says it's deprecated,
# and that we should use bettercap instead. Too much hassle, not doing it.

# submitting flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x2c -n $(echo -n 6ffcd214ffebdc0d069e | xxd -p)
Characteristic value was written successfully
```




# Flag 11 - 0x0040 - Learn about single response notifications

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag11.md)

Check out handle 0x0040 and google search gatt notify. Some tools like `gatttool` have the ability to subscribe to gatt notifications.

[This blog](https://community.nxp.com/t5/Wireless-Connectivity-Knowledge/Indication-and-Notification/ta-p/1129270) provides a brief explanation of notifications vs indications. Indications are messages that require acknowledgement, and notifications are just one-way broadcast messages.

How do I know what characteristics support notifications? Look at the "characteristic properties". These are provided as a set of flags, where each bit has a meaning. The meanings of all flag bits are spelled out on [this page](https://docs.silabs.com/bluetooth/3.2/group-sl-bt-gattdb-characteristic-properties). Notifications are supported if the flag bit 0x10 is set.

```sh
# look at characteristics of handle 0x40
$ gatttool -b 08:3A:F2:AA:12:AA --characteristics --start 0x3f --end 0x40
handle = 0x003f, char properties = 0x1a, char value handle = 0x0040, uuid = 0000ff0c-0000-1000-8000-00805f9b34fb
#                                    ^ 0x10 = notify supported

$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x40 | cut -d: -f2 | xxd -r -p; echo
Listen to me for a single notification
```

So handle 0x40 supports notify. Now how do we receive notifications?

```sh
$ gatttool --help-all
Usage:
  gatttool [OPTION?]

Help Options:
  -h, --help                                Show help options
  --help-all                                Show all help options
  --help-gatt                               Show all GATT commands
  --help-params                             Show all Primary Services/Characteristics arguments
  --help-char-read-write                    Show all Characteristics Value/Descriptor Read/Write arguments

GATT commands
  --primary                                 Primary Service Discovery
  --characteristics                         Characteristics Discovery
  --char-read                               Characteristics Value/Descriptor Read
  --char-write                              Characteristics Value Write Without Response (Write Command)
  --char-write-req                          Characteristics Value Write (Write Request)
  --char-desc                               Characteristics Descriptor Discovery
  --listen                                  Listen for notifications and indications

Primary Services/Characteristics arguments
  -s, --start=0x0001                        Starting handle(optional)
  -e, --end=0xffff                          Ending handle(optional)
  -u, --uuid=0x1801                         UUID16 or UUID128(optional)

Characteristics Value/Descriptor Read/Write arguments
  -a, --handle=0x0001                       Read/Write characteristic by handle(required)
  -n, --value=0x0001                        Write characteristic value (required for write operation)

Application Options:
  -i, --adapter=hciX                        Specify local adapter interface
  -b, --device=MAC                          Specify remote Bluetooth address
  -t, --addr-type=[public | random]         Set LE address type. Default: public
  -m, --mtu=MTU                             Specify the MTU size
  -p, --psm=PSM                             Specify the PSM for GATT/ATT over BR/EDR
  -l, --sec-level=[low | medium | high]     Set security level. Default: low
  -I, --interactive                         Use interactive mode
```

The `--listen` flag seems to be the thing to use, but when I tried, no notifications came through. After looking on Google more, I found that there [seems to be a need](http://blog.firszt.eu/index.php?post/2015/09/13/bt) to enable the notifications by writing a value of "01" to the Client Characteristic Configuration (UID 0x2902), but I don't see that UID in the list of characteristics. Maybe it is found somewhere else? Looking at the command help again, it has the `--char-desc` argument, which is also used in the previously linked blog.

```sh
$ gatttool -b 08:3A:F2:AA:12:AA --char-desc | grep 2902
handle = 0x0004, uuid = 00002902-0000-1000-8000-00805f9b34fb
#                           ^^^^ 2902 = Client Characteristic Configuration
```

So, backing up. I found a [writeup of this CTF](https://blog.yekki.co.uk/ble-hacking/) that mentions the `--listen` flag can be used after any write to the handle. Trying that:

```sh
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x40 --char-write-req -n 69 --listen
Characteristic value was written successfully
Notification handle = 0x0040 value: 35 65 63 33 37 37 32 62 63 64 30 30 63 66 30 36 64 38 65 62
^C
$ echo "35 65 63 33 37 37 32 62 63 64 30 30 63 66 30 36 64 38 65 62" | xxd -p -r; echo
5ec3772bcd00cf06d8eb
```

It worked! So I guess single notifications can be sent after a write to the handle? I wish I could find more documentation on this.

```sh
# submitting the flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "5ec3772bcd00cf06d8eb" | xxd -p)
```




# Flag 12 - 0x0042 - Learn about single response indicate

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag12.md)

Check out handle 0x0042 and google search gatt indicate. For single response indicate messages, like this challenge, tools such as `gatttool` will work just fine.

Everything I could find online seemed to treat indications and notifications the same (including the help of `gatttool`'s `--listen`, so maybe the same command as the last challenge will work?

```sh
# read instructions for challenge
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x42 --char-read | cut -d: -f2 | xxd -p -r; echo
Listen to handle 0x0044 for a single indication

# listen for indication after writing arbitrary value to handle 0x44
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x44 --char-write-req -n 69 --listen
Characteristic value was written successfully
Indication   handle = 0x0044 value: 63 37 62 38 36 64 64 31 32 31 38 34 38 63 37 37 63 31 31 33
^C
$ echo "63 37 62 38 36 64 64 31 32 31 38 34 38 63 37 37 63 31 31 33" | xxd -p -r; echo
c7b86dd121848c77c113

# submitting the flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "c7b86dd121848c77c113" | xxd -p)
Characteristic value was written successfully
```




# Flag 13 - 0x0046 - Learn about multi response notifications

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag13.md)

Check out handle 0x0046 and do what it says. Keep in mind that this notification challenge requires you to receive multiple responses in order to complete.

```sh
# get challenge directions
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x46 --char-read | cut -d: -f2 | xxd -p -r; echo
Listen to me for multi notifications

# try listening for multiple notifications (leave command running)
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x46 --char-write-req -n 69 --listen
Characteristic value was written successfully
Notification handle = 0x0046 value: 55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00
Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64
Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64
Notification handle = 0x0046 value: 63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64
^C
$ echo "55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00" | xxd -p -r; echo
U no want this msg
$ echo "63 39 34 35 37 64 65 35 66 64 38 63 61 66 65 33 34 39 66 64" | xxd -p -r; echo
c9457de5fd8cafe349fd

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "c9457de5fd8cafe349fd" | xxd -p)
Characteristic value was written successfully
```




# Flag 14 - 0x0048 - Learn about multi response indicate

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag14.md)

Check out handle 0x0048 and google search gatt indicate. Keep in mind that this challenge will require you to parse multiple indicate responses in order to complete the challenge.

```sh
# get instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x48 | cut -d: -f2 | xxd -p -r; echo
Listen to handle 0x004a for multi indications

# listening for multiple indications (let listen keep running)
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -n 69 -a 0x4a --listen
Characteristic value was written successfully
Indication   handle = 0x004a value: 55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00
Indication   handle = 0x004a value: 62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61
Indication   handle = 0x004a value: 62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61
^C
$ echo "55 20 6e 6f 20 77 61 6e 74 20 74 68 69 73 20 6d 73 67 00 00" | xxd -p -r; echo
U no want this msg
$ echo "62 36 66 33 61 34 37 66 32 30 37 64 33 38 65 31 36 66 66 61" | xxd -p -r; echo
b6f3a47f207d38e16ffa

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "b6f3a47f207d38e16ffa" | xxd -p)
```




# Flag 15 - 0x004c - Learn about BT client device attributes

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag15.md)

Check out handle 0x004c and do what it says. Much like ethernet or wifi devices, you can also change your bluetooth devices mac address.

```sh
# get challenge instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x4c | cut -d: -f2 | xxd -p -r; echo
Connect with BT MAC address 11:22:33:44:55:66
```

Googling how to change BT MAC, found a [Stack Overflow answer](https://raspberrypi.stackexchange.com/a/124117/35979), and a [forum post](https://community.infineon.com/t5/Wi-Fi-Bluetooth-for-Linux/How-to-change-Bluetooth-MAC-address-on-Raspberry-Pi-4/td-p/179476). The SO answer mentions a command `bdaddr`, but I don't see that on my Pi, nor when searching apt. The other option is to use `hcitool cmd` to send arbitrary HCI commands to the bluetooth device. I don't know what commands are possible, so looking those up, I found this list: [HCI commands list](https://www.pocketmagic.net/hci-commands-list/). I also didn't understand what OGF and OCF mean for HCI commands, so I looked that up and found a good [explanation here](https://community.nxp.com/t5/Wireless-Connectivity-Knowledge/Custom-HCI-command/ta-p/1101555), which actually pulls directly from the Bluetooth Specification.

> Each HCI command is assigned a 2 byte Opcode which is divided into two fields, called the Opcode Group Field (OGF) and Opcode Command Field (OCF).
>
> The OGF of 0x3F is reserved for vendor-specific debug commands. The organization of the opcodes allows additional information to be inferred without fully decoding the entire opcode.

For Informational Parameters commands, the OGF is defined as 0x04, which contains a command to read the BDADDR, with the OCF 0x0009. However, setting the address seems to be part of the vendor-specific debug commands. Researching further to see if this is documented somewhere, and I found [another blog post](https://community.murata.com/s/article/Drill-down-HCI-Commands-and-Setting-BD-ADDR) that spells out how to change the BT MAC. Following along:

```sh
# set BDADDR
$ sudo hcitool cmd 0x3f 0x001 0x66 0x55 0x44 0x33 0x22 0x11
< HCI Command: ogf 0x3f, ocf 0x0001, plen 6
  66 55 44 33 22 11
> HCI Event: 0x0e plen 4
  01 01 FC 00
  
# read BDADDR to confirm change
$ hcitool cmd 0x04 0x009
< HCI Command: ogf 0x04, ocf 0x0009, plen 0
> HCI Event: 0x0e plen 10
  01 09 10 00 66 55 44 33 22 11
  
# confirm change via normal method
$ hcitool dev
Devices:
	hci0	B8:27:EB:76:11:25
# but it didn't change?

# Try resetting hci0 to see if change reflects
$ sudo hciconfig hci0 down
$ sudo hciconfig hci0 up
$ hcitool dev
Devices:
	hci0	11:22:33:44:55:66
# This time the change seems to be reflected properly

# re-read handle with new client BDADDR to get flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x4c | cut -d: -f2 | xxd -p -r; echo
aca16920583e42bdcf5f

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "aca16920583e42bdcf5f" | xxd -p)
```



# Flag 16 - 0x004e - Learn about message sizes MTU

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag16.md)

Read handle 0x0048 and do what it says. Setting MTU can be a tricky thing. Some tools may provide mtu flags, but they dont seem to really trigger MTU negotiations on servers. Try using gatttool's interactive mode for this task. By default, the BLECTF server is set to force an MTU size of 20. The server will listen for MTU negotiations, and look at them, but we dont really change the MTU in the code. We just trigger the flag code if you trigger an MTU event with the value specified in handle 0x0048. GLHF!

```sh
# get challenge instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x4e | cut -d: -f2 | xxd -p -r; echo
Set your connection MTU to 444

# try changing MTU with CLI flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read --mtu 444 -a 0x4e | cut -d: -f2 | xxd -p -r; echo
Set your connection MTU to 444
# FAIL

# Go interactive to change MTU
$ gatttool -I
[                 ][LE]> connect 08:3A:F2:AA:12:AA
Attempting to connect to 08:3A:F2:AA:12:AA
Connection successful
[08:3A:F2:AA:12:AA][LE]> mtu 444
MTU was exchanged successfully: 444
[08:3A:F2:AA:12:AA][LE]> char-read-hnd 0x4e
Characteristic value/descriptor: 62 31 65 34 30 39 65 35 61 34 65 61 66 39 66 65 35 31 35 38
[08:3A:F2:AA:12:AA][LE]> exit
$ echo "62 31 65 34 30 39 65 35 61 34 65 61 66 39 66 65 35 31 35 38" | xxd -r -p; echo
b1e409e5a4eaf9fe5158

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "b1e409e5a4eaf9fe5158" | xxd -p)
```




# Flag 17 - 0x0050 - Learn about write responses

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag17.md)

Check out handle 0x0050 and do what it says. This challenge differs from other write challenges as your tool that does the write needs to have write response ack messages implemente correctly.  This flag is also tricky as the flag will come back as notification response data even though there is no "NOTIFY" property.

```sh
# get instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x50 | cut -d: -f2 | xxd -p -r; echo
Write+resp 'hello'

# write "hello", try listening for notification
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x50 -n $(echo -n "hello" | xxd -p) --listen
Characteristic value was written successfully
^C
# no notification received...

# read to get flag
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x50 | cut -d: -f2 | xxd -p -r; echo
d41d8cd98f00b204e980

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "d41d8cd98f00b204e980" | xxd -p)
```

The instructions were more daunting than the actual challenge was. Not sure if `gatttool` is just "implemented properly" or what. But all it took was to write "hello" and re-read the handle. What I don't understand is why the flag came back from a read when the hint said it should come back as a notify response.




# Flag 18 - 0x0052 - Hidden notify property

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag18.md)

Take a look at handle 0x0052. Notice it does not have a notify property. Do a write here and listen for notifications anyways! Things are not always what they seem!

```sh
# get instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x52 | cut -d: -f2 | xxd -p -r; echo
No notifications here! really?

# look at characteristic properties
$ gatttool -b 08:3A:F2:AA:12:AA --characteristics --start 0x50 --end 0x52
handle = 0x0051, char properties = 0x0a, char value handle = 0x0052, uuid = 0000ff15-0000-1000-8000-00805f9b34fb
#                                    ^ notification bit not set

# try write + listen anyway
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x52 -n 69 --listen
Characteristic value was written successfully
Notification handle = 0x0052 value: 66 63 39 32 30 63 36 38 62 36 30 30 36 31 36 39 34 37 37 62
^C
$ echo "66 63 39 32 30 63 36 38 62 36 30 30 36 31 36 39 34 37 37 62" | xxd -p -r; echo
fc920c68b6006169477b

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "fc920c68b6006169477b" | xxd -p)
```




# Flag 19 - 0x0054 - Use multiple handle properties

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag19.md)

Check out all of the handle properties on 0x0054! Poke around with all of them and find pieces to your flag.

```sh
# check for instructions from read
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x54 | cut -d: -f2 | xxd -p -r; echo
So many properties!

# look at properties
$ gatttool -b 08:3A:F2:AA:12:AA --characteristics --start 0x53 --end 0x54
handle = 0x0053, char properties = 0x9b, char value handle = 0x0054, uuid = 0000ff16-0000-1000-8000-00805f9b34fb
```

The properties value 0x9b is `1001 1011` in binary. Looking at the [property flags](https://docs.silabs.com/bluetooth/3.2/group-sl-bt-gattdb-characteristic-properties), it appears to have NOTIFY (0x10), RELIABLE_WRITE (0x101), WRITE (0x8), and READ (0x2).

```sh
# get part of flag from write/notify
$ gatttool -b 08:3A:F2:AA:12:AA --char-write-req -a 0x54 -n 69 --listen
Characteristic value was written successfully
Notification handle = 0x0054 value: 30 37 65 34 61 30 63 63 34 38
^C
$ echo "30 37 65 34 61 30 63 63 34 38" | xxd -p -r; echo
07e4a0cc48

# get another part of flag from read
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x54 | cut -d: -f2 | xxd -p -r; echo
fbb966958f

# submit flag (read part + notify part)
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "fbb966958f07e4a0cc48" | xxd -p)
Characteristic value was written successfully
```




# Flag 20 - 0x0056 - OSINT the author!

[Hints](https://github.com/hackgnar/ble_ctf/blob/master/docs/hints/flag20.md)

Figure out the authors twitter handle and do what 0x0056 tells you to do!

The author's GitHub username is "hackgnar", and his name is Ryan Holeman (I got the challenge files/hints from his GitHub). Googling Twitter for his real name, I don't find any results that look promising. But googling Twitter for "hackgnar" turns up his profile: [@hackgnar](https://twitter.com/hackgnar).

```sh
# get instructions
$ gatttool -b 08:3A:F2:AA:12:AA --char-read -a 0x56 | cut -d: -f2 | xxd -p -r; echo
md5 of author's twitter handle

# get md5sum of his handle
$ echo -n "@hackgnar" | md5sum
d953bfb9846acc2e15eecd5b467a79aa  -

# extract first 20 chars of md5
$ echo -n "@hackgnar" | md5sum | cut -f1 -d' ' | head -c 20
d953bfb9846acc2e15ee

# submit flag
$ gatttool -b 08:3A:F2:AA:12:AA -a 0x2c --char-write-req -n $(echo -n "d953bfb9846acc2e15ee" | xxd -p)
```

