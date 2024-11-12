# Shower Controller
A responsive application for controlling my Mira Mode shower.


# Bluetooth Protocol
This file documents my current understanding of the bluetooth messages passed between the client application and the shower device.  This information has been reverse engineered by sniffing bluetooth data and experimenting.

I would also like to give credit to [python-miramode](https://github.com/alexpilotti/python-miramode) which acted as a starting point for my investigations.

## Device Information Service
The standard device information service can be queried for information below using either 16 bit or 128 bit UUIDs.

The relevant 16 bit UUIDs are:
|Service / Characteristic|UUID|
|--|--|
|SERVICE_DEVICE_INFORMATION|180a|
|CHARACTERISTIC_MODEL_NUMBER|2a24|
|CHARACTERISTIC_SERIAL_NUMBER|2a25|
|CHARACTERISTIC_HARDWARE_REVISION|2a26|
|CHARACTERISTIC_FIRMWARE_REVISION|2a27|
|CHARACTERISTIC_MANUFACTURER_NAME|2a29|


## MIRA specific
An additional service with 2 characteristics is also available (names are my own).

|Service / Characteristic|UUID|
|--|--|
|SERVICE_DEVICE_MIRA|bccb0001-ca66-11e5-88a4-0002a5d5c51b|
|CHARACTERISTIC_WRITE|bccb0002-ca66-11e5-88a4-0002a5d5c51b|
|CHARACTERISTIC_NOTIFICATIONS|bccb0003-ca66-11e5-88a4-0002a5d5c51b|

When interacting with the MIRA service, commands are written to the WRITE characteristic, and resulting changes are read from the NOTIFICATIONS service.
Applications should subscribe to the NOTIFICATIONS service as some notification payloads exceed the maximum permitted length and are therefor split across multiple notifications.

When a client pairs with a device, it sends a 4 byte clientId (I've been generating a random clientId for each application instance), and the device allocates a single byte identifier for the client (I call this the clientSlot) which it returns in the subsequent notification.
This clientSlot is included in all subsequent commands and notifications, and the clientId is used when calculating the CRC (see below).

A certain number of presets can be configured in the device.  The way the protocol works suggets that these are limited to 16 as a single bit is set in one of two bytes to indicate that a slot is populated.

### Commands
The basic structure of commands is:

    [ clientSlot, commandTypeId, payloadLength, [ payload ], [ CRC ] ]

The CRC is 2 bytes calculated for the sequence of bytes:

    [ clientSlot, commandTypeId, payloadLength, [ payload ], [ 4 byte clientId ] ]

|Field|Description|
|--|--|
|clientId|4 (random?) bytes chosen by the client and registered with the device during pairing.| 
|clientSlot|Unique identifier for the client within the device - allocated by the device during pairing.|
|commandTypeId|Unique identifier for the type of command being sent|
|payloadLength|The number of bytes of payload that will follow immediately after.  Zero if there is no payload.|
|payload|A sequence of bytes that is exactly payloadLength bytes long|
|crc|A two byte CRC.  Uses a standard algorithm referred to under one of these names: **CRC-16/IBM-3740**, **CRC-16/AUTOSAR**, **CRC-16/CCITT-FALSE**|

If the overall command length exceeds 20 bytes, it should be sent as a sequence of chunks of up to 20 bytes each.

TBC: Looks like 'update' and 'request' commandTypeId for a given property are 128 bits apart.

|command|commandTypeId|payloadLength|payload|expected notification type|
|--|--|--|--|--|
|Pair\*|0xeb|24|[ [ clientId ], [ clientName ] ] NOTE: As the device is not aware of the client at the time this request is made, it uses 0x00 for the clientSlot, and uses a pairing-specific id for CRC calculation instead of the clientId.  It's unclear whether this id is hardcoded in the official application or looked up/discovered.  I have not documented this id, but it is easy to brute force from sniffed pairing requests|SuccessOrFailure|
|Unpair\*|0xeb|1|[ clientSlot of client to unpair ]|SuccessOrFailure|
|RequestClientSlots|0x6b|1|[ 0x00 ]|Slots|
|RequestClientDetails|0x6b|1|[ 0x10 + clientSlot ]|ClientDetails|
|RequestNickname|0x44|0|No payload|Nickname|
|UpdateNickname\*|0xc4|16|[ deviceNickname ]|SuccessOrFailure|
|RequestDeviceState|0x07|0|No payload|DeviceState|
|RequestDeviceSettings|0x3e|0|No payload|DeviceSettings|
|UpdateWirelessRemoteButtonSettings\*|0xbe|2|[ 0x01, wirelessRemoteButtonSettingBits ]|SuccessOrFailure|
|UpdateDefaultPresetSlot\*|0xbe|2|[ 0x02, presetSlot ]|SuccessOrFailure|
|UpdateControllerSettings\*|0xbe|2|[ 0x03, controllerSettingBits ]|SuccessOrFailure|
|RequestPresetSlots|0x30|1|[ 0x80 ]|Slots|
|RequestPresetDetails|0x30|1| [ 0x40 + presetSlot ] |PresetDetails|
|UpdatePresetDetails\*|0xb0|24|[ presetSlot, 0x01, targetTemperature, 0x64, duration, 0x02, 0x00, 0x00, [ presetName ] ]|SuccessOrFailure|
|DeletePresetDetails\*|0xb0|24|[ presetSlot, 0x01, [ 22 bytes of 0x00 ] ]|SuccessOrFailure|
|StartPreset|0xb1|1|[ presetSlot ]|ControlsOperated|
|OperateOutlets|0x87|5|[ timerState, 0x01, targetTemperature, outletState1, outletState2 ]|ControlsOperated|
|RequestOutletSettings|0x0f or 0x10 depending on outlet|0|No payload|OutletSettings|
|UpdateOutletSettings\*|0x8f ot 0x90 depending on outlet|11|[ outletIdentifier, outletIdentifier, 0x08, 0x64, maximumDuration, 0x01, maximumTemperature, 0x01, 0x2c, 0x01, successfulUpdateCommandCounter ]|SuccessOrFailure|
|RestartDevice\*|0xf4|1|[ 0x01 ]|SuccessOrFailure|
|RequestTechnicalInformation|0x32|1|[ 0x01 ]|TechnicalInformation|
|UnknownRequestTechnicalInformation|0x41|0|No payload|UnknownTechnicalInformation|
|UnknownRequestTechnicalInformation2|0x40|1|[ 0x01 ]|[ 0x00 re[eated 18 times ]|
|UnknownRequestRequest|0x40|0|No payload|[ TBC, TBC, TBC, TBC ]|

\* NOTE: Many of the update type commands will fail while the timer is running (i.e. a non-zero timerState), and also for 5 seconds afterwards.  The official application tends to use a popup warning and stops the device outlets before executing these commands.


### Notifications
The basic structure of notifications is:

    [ clientSlot + 0x40, TBC (seems to always be 0x01), payloadLength, [ payload ] ]

|Field|Description|
|--|--|
|clientSlot|Unique identifier for the client within the device - allocated by the device during pairing.|
|payloadLength|The number of bytes of payload that will follow immediately after.|
|payload|A sequence of bytes that is exactly payloadLength bytes long.|

If the overall notification length is sufficiently large, the notification will be received in chunks.  The payloadLength can be used to understand whether it is necessary to wait for another chunk to arrive before the notification has been received in full.
There is nothing in the notification to indicate which command type triggered it, but so far it appears that the payloadLength can be used to differentiate between notification types (the first byte of the payload is sometimes used to further define the structure of the remaining payload).

|notificationType|payloadLength|payload|
|--|--|--|
|SuccessOrFailure|1|[ varies.  0x80 for failure, clientSlot for pairing success, 0x01 for other success]|
|Slots|2|[ Each bit set indicates an occupied slot.  The slot type (preset or client) depends on the command that triggered the notification ]|
|DeviceSettings|4|[ TBC, wirelessRemoteButtonSettingBits, defaultPresetSlot, controllerSettingBits ]|
|DeviceState|10|[ timerState, TBC, targetTemperature, TBC, actualTemperature, outletState1, outletState2, secondsRemainingPart1, secondsRemainingPart2, successfulUpdateCommandCounter ]|
|ControlsOperated|11|[ 0x01 (command made a change) or 0x80 (e.g. no change because controls already stopped), timerState, TBC, targetTemperature, TBC, actualTemperature, outletState1, outletState2, secondsRemainingPart1, secondsRemainingPart2, successfulUpdateCommandCounter ]|
|OutletSettings|11|[ outletIdentifier, TBC, TBC, TBC, minimumDurationSeconds, TBC, maximumTemperature, TBC, TBC, TBC, TBC, successfulUpdateCommandCounter ]|
|Nickname|16|[ deviceNickname ]|
|ClientDetails\*|20|[ clientName ]|
|PresetDetails|24|[ presetSlot, TBC, targetTemperature, TBC, durationSeconds, TBC, TBC, TBC, [ presetName ] ]|
|TechnicalInformation|16|[ 0x00, valveType, 0x00, valveSoftwareVersion, 0x00, uiType, 0x00, uiSoftwareVersion, 0x00, 0x00, 0x00, 0x00, 0x00, TBC, 0x00, bluetoothSoftwareVersion ]|
|UnknownTechnicalInformation|4|[ TBC, TBC, TBC, TBC ]

\* NOTE: The ClientDetails notification does not include information about which clientSlot the name relates to.  It appears that an application needs to link the notification to the command it sent that requested details for a specific clientSlot. 



### Payload data description / conversion information
|PayloadData|Number of Bytes|Description|
|--|--|--|
|clientId|4|4 (random?) bytes chosen by the client and registered with the device during pairing.  Subsequently used for generating CRC so the device can validate that requests came from the registeered client| 
|controllerSettingBits|1|bitmask (???????1=swapped top button outlet, ??????1?=standby lighting off)|
|wirelessRemoteSettingBits|1|bitmask (???????1=outlet1 enabled, ??????1?=outlet2 enabled)|
|targetTemperature, actualTemperature, maximumTemperature|1|celcius = (256 + payloadByte) / 10|
|duration, maximumDuration|1|seconds = payloadByte \* 10|
|secondsRemaining|2|seconds = a 16 bit unsigned integer split across 2 bytes.  First byte are the most significant bits|
|timerState|1|0x00 = timer stopped, 0x01 = timer running, 0x03 = lockout timer running.  Must be set consistently with the outlets - i.e. timer cannot be set to stopped when one of the outlets is on.  The lockout timer appears to run for 5 minutes after the shower is turned off by the physical controls.  May be a safety thing to prevent changes in case someone is still in the shower and just turns it back on|
|outletState|1|0x64 = running.  0x00 = not running|
|outletIdentifier|1|0x00 for first outlet, 0x04 for second outlet|
|successfulUpdateCommandCounter|1|Appears to cycle between 0x09 and 0x0f incrementing by one after each sucessfull command unless following an UpdateOutletSettings command where the value from the command is reflected in the resulting notification|
|deviceNickname|16|Nickname of device in utf-8 bytes, padded with trailing 0x00 to 16 bytes|
|clientName|20|Name of the client application in utf-8 bytes padded with trailing 0x00 to 20 bytes|
|presetName|16|Name of the preset in utf-8 bytes, padded with trailing 0x00 to 16 bytes|
