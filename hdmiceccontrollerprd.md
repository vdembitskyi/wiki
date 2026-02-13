# **HDMI-CEC VSL Layer**

This document outlines the API requirements for the Hardware Abstraction Layer (HAL) to support the HDMI-CEC Controller.

#### **Review of Existing Solutions **

A review of the AOSP source code identified a standard HDMI-CEC API supported by all vendors. Since Android is the industry standard, using this API accelerates time-to-market and avoids the complexity of custom solutions.

To demonstrate the complexity of the HDMI-CEC feature, the complete Android HDMI-CEC (controller) stack is detailed below, including Lines of Code (LOC) metrics.
###### Core logic
1. **HdmiControlService.java** (5,284 lines): CRITICAL. The central hub. Handles client app registration, settings, power state, and coordinates everything.
2. **HdmiCecController.java** (1,799 lines): CRITICAL. The bridge to the HAL. Handles the "dumb pipe" transmission and address allocation logic.
3. **HdmiCecNetwork.java** (1,027 lines): Manages the map of the CEC network (which device is connected to which port).

###### Device Type
1. **HdmiCecLocalDevicePlayback.java** (803 lines): Logic for Streaming Sticks/Set-top boxes.
2.  **HdmiCecLocalDeviceTv.java** (1,822 lines): Logic specific to TVs (System Audio control, ARC handling).
3.  **HdmiCecLocalDevice.java** (1,551 lines): Base class for all devices.
4.  **HdmiCecLocalDeviceAudioSystem.java** (1,402 lines): Logic for Soundbars/AVRs.


###### Protocol layer
1. **HdmiCecMessageValidator.java** (1,194 lines): Ensures outgoing messages are spec-compliant.
2. **HdmiCecMessageBuilder.java** (722 lines): Factory methods for creating raw byte arrays.
3. **Constants.java** (677 lines): definitions of OpCodes and operands.

###### Actions
1. **DeviceDiscoveryAction.java** (535 lines): Finding other devices on the bus.
2. **OneTouchPlayAction.java** (258 lines): The "Active Source" logic (Switch input when Play is pressed).
4. **RoutingControlAction.java** (126 lines): Managing input switching to suspend the OTT device whenever a user transitions to Vizio Home/other inputs.
5. **VolumeControlAction.java** (209 lines): Sending/Receiving Vol+/Vol- commands.

The main complexity stems from the slow, unreliable nature of the CEC bus and interoperability issues across vendors like Samsung, LG, Sony, Vizio, and Philips. However, with the available architecture and handling examples, achieving a working solution is simply a matter of technical execution and unlimited AI tokens.

##### **Vizio HDMI-CEC stack**
hal-cec problems:
1. Broken Abstraction & Coupling: The platform driver bypasses the intended hardware interface and directly implements the high-level service contract, while also relying on a global singleton to find itself during callbacks, making the code fragile and impossible to test in isolation.
2. System Freeze Risk: The hardware driver performs synchronous sleep operations (up to ~1.2 seconds) on the main event thread when the bus is busy, which will block the entire service, causing unresponsiveness and potential timeouts.
3. To reuse this codebase for OTT, rewriting is necessary. However, achieving robust interoperability will still be difficult, as the current implementation relies on a proprietary custom architecture instead of the standard AOSP flow.

###### API contract between Controller and HAL-CEC
1. Requests (Controller -> HAL):
    1. Core CEC:
        *   `msgFromCtrl::cec::device::Startdd` (Start Discovery)
        *   `msgFromCtrl::cec::settings::EnableCec` (Enable/Disable CEC) MISSING in your list
        *   `msgFromCtrl::cec::routing::SelectDev` (Switch Input)
        *   `msgFromCtrl::cec::routing::SendRoutingChange` (Route change 0x80)
        *   `msgFromCtrl::cec::routing::RequestActiveSource` (Request Active Source)
        *   `msgFromCtrl::cec::power::StandbyAll` (Standby all devices)
        *   `msgFromCtrl::cec::power::StandbyDev` (Standby specific device)
    2. Audio:
        *   `msgFromCtrl::cec::audio::SetVol`
        *   `msgFromCtrl::cec::audio::SetMute`
        *   `msgFromCtrl::cec::audio::SetSysMode` (System Audio Mode Request)
        *   `msgFromCtrl::cec::audio::SbMode` (Soundbar Mode Enable/Disable
    3. Remote Control:
        *   `msgFromCtrl::cec::ctrl_pass_through::SetUserCtrlPressed` (Key Press)
        *   `msgFromCtrl::cec::ctrl_pass_through::SetUserCtrlReleased` (Key Release)
    4. HDMI/Hardware:
        *   `msgFromCtrl::hdmi::edid::ModePort` (Set HDMI Mode: Auto/Standard/Compatibility)
        *   `msgFromCtrl::hdmi::ToggleHpd` (Toggle Hot Plug Detect)
2. Events (HAL -> Controller)
    1. Core CEC:
        *   `msgFromSrvc::cec::device::DevListChanged` (Device Discovery results)
        *   `msgFromSrvc::cec::device::DeviceDiscoveryInProgress`
        *   `msgFromSrvc::cec::device::DeviceDiscoveryFinished`
        *   `msgFromSrvc::cec::routing::ActiveSource` (Input switching events)
    2. Power:
        *   `msgFromSrvc::cec::power::ImgViewOn` (Image View On)
        *   `msgFromSrvc::cec::power::TxtViewOn` (Text View On)
        *   `msgFromSrvc::cec::power::Standby` (Standby received from network)
    3. Audio:
        *   `msgFromSrvc::cec::audio::Sad` (Short Audio Descriptor)
        *   `msgFromSrvc::cec::audio::Volume`
        *   `msgFromSrvc::cec::audio::Mute`
        *   `msgFromSrvc::cec::audio::SbStatus` (Soundbar Status)
        *   `msgFromSrvc::cec::vizio_sb::ReportSettingDescr` (Vizio Soundbar Settings)
        *   `msgFromSrvc::cec::vizio_sb::AudioFormat` (Vizio Soundbar Audio Format)
    4. HDMI/Hardware:
        *   `msgFromSrvc::hdmi::packets::Spd` (HDMI Source Product Description)
        *   `msgFromSrvc::hdmi::device::SourceReady`
        *   `msgFromSrvc::input::signal::Status` (HDMI Signal Status)
        *   `msgFromSrvc::input::signal::Statistics` (HDMI Signal Stats)

##### **AOSP HDMI-CEC API**
Android separates these functions: the controller manages protocol parsing and messaging, while the HAL handles raw data transmission.

1. Core Messaging APIs
    *   `sendMessage(CecMessage)`
        *   Input: Source Address, Destination Address, Body (Opcode + Operands).
        *   Expected Behavior: Transmit the frame to the hardware. Must return `SUCCESS`, `NACK`, or `BUSY`.
    *   `onCecMessage(CecMessage)`
        *   Input: 
            * `initiator` - The 4-bit Logical Address (0-15) of the device that sent the message.
            * `destination` - The 4-bit Logical Address (0-15) of the device that the message is intended for
            * `body[]` - The raw payload of the CEC message, excluding the header byte (src/dst)
        *   Expected Behavior: HAL invokes this callback when it receives a CEC frame from the hardware bus.
    *   `onHotplugEvent(portId, connected)`
        *   Input:
            *   `portId`: This maps to the port IDs returned by getPortInfo().
            *   `connected`: true/false
2. Address Management APIs
    *   `addLogicalAddress(logicalAddress)`
        *   Input: The Logical Address (0-15) Java wants to claim.
        *   Expected Behavior: The hardware should start ACKing messages sent to this address.
    *   `clearLogicalAddress()`
        *   Input: None.
        *   Expected Behavior: The hardware should stop ACKing all addresses (except Broadcast 15). Used when the service is disabled or resetting.
3. Hardware Info & Configuration APIs
    *   `getPhysicalAddress()`
        *   Expected Behavior: Return the 16-bit Physical Address (e.g., 0x1000) read from the EDID of the sink (TV).
        *   Note: If the device is a TV, this usually returns 0x0000.
    *   `getPortInfo()`
        *   Expected Behavior: Return an array describing all HDMI ports (Port ID, Input/Output type, CEC support, ARC support).
    *   `getVendorId()`
        *   Expected Behavior: Return the 24-bit IEEE OUI (Manufacturer ID) stored in the hardware/driver config.
    *   `getCecVersion()`
        *   Expected Behavior: Return the CEC version supported by the hardware (usually CEC 1.4 or 2.0).
    *   `isConnected(portId)`
        *   Expected Behavior: Return true if a cable is plugged into the specific HDMI port.
4. Control & Settings APIs
    *   `enableCec(enabled)`
        *   Expected Behavior: Global switch. If false, the CEC hardware should completely power down or ignore the bus.
    *   `enableSystemCecControl(enabled)`
        *   Expected Behavior: Controls who handles messages.
            *   true: Forward messages to Android (Java).
            *   false: The Hardware/Firmware handles messages internally (if applicable).
    *   `enableWakeupByOtp(enabled)`
        *   Expected Behavior: If true, the hardware should wake the system from suspend when it receives a "One Touch Play" (<Image View On>) message.
    *   `setLanguage(language)`
        *   Input: 3-letter ISO language code (e.g., "eng").
        *   Expected Behavior: Some CEC controllers store the language to auto-reply to <Get Menu Language>.
5. Audio Return Channel (ARC) APIs
    *   `enableAudioReturnChannel(portId, enabled)`
        *   Expected Behavior: Enable or disable the electrical ARC pins on the specified HDMI port.
    
    
##### **Vizio HAL-VSL API (new stack)**
The existing Vizio HAL requires reimplementation to support OTT device. Furthermore, the current architecture lacks scalability and risks delaying time-to-market because the HDMI-CEC core logic resides in the HAL layer rather than the service (controller). This design implies that onboarding new vendors will be time-consuming, as they must implement a complex set of APIs. Additionally, specific vendor implementations may introduce subtle bugs, causing behavior to vary between SoCs. This inconsistency significantly increases maintenance costs and poses a risk to ViziOS stability.

Adopting an architecture that mimics the AOSP HDMI-CEC stack provides a proven, scalable foundation supported by all major vendors. This strategy avoids the pitfalls often associated with custom implementations. Although this requires handling complex management logic, the AOSP codebase serves as a reliable reference, ensuring the system behaves predictably across different SoCs
    
###### **VSL API**
The new VSL API defines 6 messages that cover all necessary areas for an OTT device.
1. Sink Connected Event
```
{
    "$id": "http://vizio.com/platform/v1.0/hdmi/sink_connected",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "Notified when a valid HDMI sink is identified and its EDID is parsed.",
    "type": "object",
    "properties": {
        "physicalAddress": {
          "type": "integer"
        },
        "vendorId": {
          "type": "string"
        },
        "hdmiVersion": {
          "type": "string",
          "enum": [
             "HDMI1.4",
             "HDMI2.0",
             "HDMI2.1"
          ]
        },
        "portId": {
          "type": "integer"
        }
    },
    "required": [
        "physicalAddress",
        "vendorId",
        "hdmiVersion",
        "portId"
    ],
    "additionalProperties": false
}
```
2. Sink Disconnected Event
```
{
    "$id": "http://vizio.com/platform/v1.0/hdmi/sink_disconnected",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "Notified when the HDMI Sink is disconnected or no longer valid",
    "type": "object",
    "properties": {
        "portId": {
          "type": "integer"
        }
    },
    "required": [
        "portId",
    ],
    "additionalProperties": false
}
```
3. CEC Message Received Event
```
{
    "$id": "http://vizio.com/platform/v1.0/controller/hdmi/cec_message_received",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "Notified for every valid CEC frame received on the bus",
    "type": "object",
    "properties": {
        "initiator": {
          "type": "integer",
          "description": "The logical address of the sender"
        },
        "destination": {
          "type": "integer",
          "description": "The logical address of the destination (0-15) or 15 for Broadcast"
        },
        "opcode": {
          "type": "integer",
          "description": "CEC Opcode"
        },
        "body": {
          "type": "string",
          "description": "The raw parameters/operands associated with the opcode. Empty if the message has no arguments. **Base64 encoded**"
        }
    },
    "required": [
        "initiator",
        "destination",
        "opcode",
        "body"
    ],
    "additionalProperties": false
}
```
4. Add Logical Address
```
{
    "$id": "http://vizio.com/platform/v1.0/controller/hdmi/add_logical_address",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "Start acknowledging (ACK) messages sent to this address",
    "type": "object",
    "properties": {
        "address": {
          "type": "integer",
        }
    },
    "required": [
        "address",
    ],
    "additionalProperties": false
}
```
5. Forget Logical Address
```
{
    "$id": "http://vizio.com/platform/v1.0/controller/hdmi/forget_logical_address",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "After this call, the device will strictly act as a follower (or remain silent) and will no longer acknowledge messages directed to the <address>",
    "type": "object",
    "properties": {
        "address": {
          "type": "integer",
        }
    },
    "required": [
        "address",
    ],
    "additionalProperties": false
}
```
6. Send CEC Message
```
{
    "$id": "http://vizio.com/platform/v1.0/controller/hdmi/send_cec_message",
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "description": "Transmits a CEC message to the bus",
    "type": "object",
    "properties": {
        "src": {
          "type": "integer",
        },
        "dst": {
          "type": "integer",
        }
        "body": {
          "type": "string",
          "description": "The raw byte array containing the opcode and operands. **Base64 encoded**"
        }
    },
    "required": [
        "src",
        "dst",
        "body"
    ],
    "additionalProperties": false
}
```
NOTE: The VSL transport relies on JSON, which is not natively suitable for binary data. Consequently, payloads must be encoded for reliable transfer. While this introduces a serialization overhead, the low data volume typical of the HDMI-CEC protocol renders this bottleneck negligible. However, should performance requirements change, the API architecture could be refactored to pass a file descriptor. In that model, the VSL would serve only as an event signal (onCecMessage), while actual data transfer would occur via direct reads and writes to the descriptor. This optimization is currently out of scope.
