# Smartphone localization

The standard implementation of indoor localization of smartphone is by scanning for BLE messages and subsequently using the received signal strength indicator (RSSI) values to do the positioning, either by (1) trilateration, (2) fingerprinting, or (3) simultaneous localization and mapping (SLAM).

It is also possible to have the phone broadcasting BLE messages instead and having a network of nodes scanning for devices that transmit these messages. In that case the indoor localization is performed in-network. In the case of Apple hardware, devices that run iOS have the capability to run in peripheral mode.

The problem is running as a peripheral in the background. In that case the Apple documentation says:

> ### The bluetooth-peripheral Background Execution Mode
>
>[T]he Core Bluetooth framework allows your app to advertise while in the background state. That said, you should be aware that advertising while your app is in the background operates differently than when your app is in the foreground. In particular, when your app is advertising while in the background:
> 
>* The CBAdvertisementDataLocalNameKey advertisement key is ignored, and the **local name** of peripheral is **not advertised**.
>* All **service UUIDs** contained in the value of the CBAdvertisementDataServiceUUIDsKey advertisement key are placed in a special **"overflow"** area; they can be discovered only by an iOS device that is **explicitly scanning** for them.
>* If all apps that are advertising are in the background, the frequency at which your peripheral device sends advertising packets may **decrease**.

## What now?

Some of the questions are... 

* What is this **overflow** area? 
* Why can it only be discovered by an **iOS** device? Or do they mean that they can only be discovered by a (iOS or non-iOS) device in **explicit** scanning mode (searching for these particular service UUIDs)?

To first find out if there is something fishy going on between iOS devices we broadcast as a beacon from one iOS device and try to scan the messages it is sending out from the console on a Linux machine:

    sudo hcitool lescan --duplicates
    sudo hcidump -x -R | grep -A1 'BD D4 51' | grep -v '>'

The messages arrive on the Linux machine just fine. So, this means that it's not the case that they mean that only iOS devices can discover the BLE messages. Moreover, we see that there are single messages sent out. The information is not spread out over multiple messages. The messages also look very straightforward. They do seem to hash/encrypt the service UUIDs, so our next question:

* How are the service UUIDs encoded in the background BLE advertisements?

At PagePinner... To parse the hashed bit, they use the identity `14ff4c0001`:

* `14`     -> length (0x14, hence 20 bytes)
* `ff`     -> custom advertisement type
* `4c0001` -> apple defined

If we look at a message:

    02011a11067f00000079e3e78e82319dfbf045bdd209084c69676874426c7514ff4c00010000000000000000000000000000000200000000000000000000

    02011a11067f00000079e3e78e82319dfbf045bdd209084c69676874426c75XXXXXXXXXX0102030405060708090a0b0c0d0e0f1011121314000000000000

We count till the 20th (0x14) byte.

## Scan response

Is this data actually in the advertisement or does it reside in the scan response? 

Dave at Maxwell Walker (see resources) has the following hypothesis:

    It's not explicitly documented, but there are only so many options provided by the Bluetooth specification that would provide for such a feature. Most likely, the "overflow" area iOS refers to is a scan response message. This is a polled message that advertising devices can listen for and provide additional data without a connection. Scan responses allow for whitelisting, which would let iOS filter and ignore any scan requests it receives from any devices except another iOS device.

If this is actually true, we would need to mimick a scanning iOS device from the Crownstone. 

If we have something in the foreground. In this case with local name `Fred` and service UUID `0xC007` the advertisement looks like this:

    > 04 3E 19 02 01 00 01 8B 89 6B 29 A5 69 0D 02 01 1A 03 03 07 C0 05 09 46 72 65 64 D9 
    > 04 3E 0C 02 01 04 01 8B 89 6B 29 A5 69 00 D9 

You see `07 C0` somewhere on the first line: the service UUID is just sent in the advertisement packet. 
The advertisement is a so-called **connectable undirected advertising frame** (0x00 on the 6th position).
The second line is a scan response. It does not contain additional information.
The **scan reponse** can be identified similarly (0x04 on the 6th position). 
The size of the advertisement is 0x19 (data length) and that of the scan response 0x0C.

If we now go to the background it becomes:

    > 04 3E 24 02 01 00 01 8B 89 6B 29 A5 69 18 02 01 1A 14 FF 4C 00 01 00 00 00 00 10 00 00 00 00 00 00 00 00 00 00 00 DA 
    > 04 3E 0C 02 01 04 01 8B 89 6B 29 A5 69 00 DA 

We see `14 FF 4C 00 01` (see above). The size of the advertisement is now 0x24 and the byte array is:

    00 00 00 00 | 10 00 00 00 | 00 00 00 00 | 00 00 00 00

The last byte (in this case `DA`) changes from one packet to the next (this is actually the RSSI value
).

If we change to other values (for example service UUID `0xC008`) the byte array changes to:

    00 00 00 00 | 00 00 00 08 | 00 00 00 00 | 00 00 00 00

So, how is this encoded? Is there some specific hash so we can go from the values above to the service UUIDs? 
Interestingly enough, there are hash collisions, so it won't be possible to distinguish every 32-bit UUID. For 
example, `0x1001` and `0x3333` map to the following sequence:

With UUID `0x1001`:

    > 04 3E 24 02 01 00 01 8C 91 A0 AD 8F 40 18 02 01 1A 14 FF 4C 00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 00 D3 
    > 04 3E 0C 02 01 04 01 8C 91 A0 AD 8F 40 00 D3 

With UUID `0x3333`:

    > 04 3E 24 02 01 00 01 8C 91 A0 AD 8F 40 18 02 01 1A 14 FF 4C 00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 00 D3 
    > 04 3E 0C 02 01 04 01 8C 91 A0 AD 8F 40 00 D3 

Days later the same device with again service UUID `0x1001` and `0x3333` respectively:

    > 04 3E 24 02 01 00 01 6F 7B 65 0A 60 50 18 02 01 1A 14 FF 4C 00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 00 C1 
    > 04 3E 24 02 01 00 01 6F 7B 65 0A 60 50 18 02 01 1A 14 FF 4C 00 01 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00 00 BE 

Note that there is a different (virtual) MAC address used: `50:60:0A:65:7B:6F` instead of `40:8F:AD:A0:91:8C` (the bytes
are shown in the reverse order).

## Conclusions

Hence, what does this mean? How can we use this?

1. Have device identifiers encoded into the service UUIDs. Check beforehand which service UUIDs are unique. This means
we can track a subset of the 32-bit space (might be less than 128 items for example). 
2. There is no encryption possible. Any person who can craft a BLE packet with the above layout will be treated by
the Crownstone scanners in the same way.
3. It might be necessary to check so now and then if this person is indeed who he transmits to be. This can be done
for example by tag-to-toggle functionality (an NFC-like function using BLE only) if active involvement of the user
is required. It can also be done in the background by regularly setting up a connection to the BLE device that the 
device identifier is mapped to. 
4. Random other iPhones broadcasting an advertisement that interferes with the iPhones that need to be tracked is a
possibility. For this it is necessary that these iPhones are advertising (not so likely). One solution is to have a
setup procedure to the iPhones to be tracked that can rotate their 'service' UUID to a different identifier as soon
as collisions are discovered. The collision discovery process would require point 3 (a failing authentication check).
5. There will still be issues if a to-be-tracked iPhone and one not to be tracked are all the time next to each other.
For that it might be useful to regularly rotate the 'service' UUID anyway to something not heard for quite a while. 
This will solve most problems.
6. Main issue left is the number of iPhones that can be tracked.

## To check

1. Generate all UUIDs in the background on a iPhone, 1 every second for example and register all corresponding byte 
array values. This will immediately give the upper bound on the number of iPhones that can be tracked.
2. We need to establish if the hash from service UUID to the byte array differs from phone to phone. This is not likely
because the MAC address is rotated. 
3. If a phone specific code is used that is not transmitted within the advertisement, it needs to be obtained by the 
receiving party in another way. Are there some even more special messages advertised when a new phone gets near and 
sends a scan request?
4. Are there other scan responses send when an iPhone is scanning? Until now I have just been listening in all the 
time and this does not seem the case. All the scan responses are all the time exactly the same.


# Resources

* [Core Bluetooth background processing for iOS apps, Apple](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html)
* [Making iOS7 BLE peripheral to work in the background, Stackoverflow](https://stackoverflow.com/questions/20915249/making-ios7-ble-peripheral-to-work-in-background)
* [CBAdvertisementDataOverflowServiceUUIDsKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdataoverflowserviceuuidskey?language=objc)
* [startAdvertising:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393252-startadvertising?language=objc)
* [How to get BLE overflow hash bit from IOS peripheral? PagePinner](http://www.pagepinner.com/2014/04/how-to-get-ble-overflow-hash-bit-from.html)
* [Bluetooth UUIDs and interoperable advertisements, Maxwell Walker](http://www.maxwellwalker.com/s/post/1786/2016/01/04/bluetooth-uuids-and-interoperable-advertisements.html)
