# HDMICecController

What API do we need to implement? It is a service not a HAL module.

Android split:
* base/services/core/java/com/android/server/hdmi (Controller-HAL bridge)
    * HdmiCecController.java - It acts as the bridge/intermediary between the high-level Android framework (HdmiControlService) and the low-level HDMI-CEC Hardware Abstraction Layer (HAL).



The HAL (Hardware Abstraction Layer) is generally "dumb" regarding CEC protocol logic. Its job is mostly to put bytes on the wire and take bytes off the wire. The Controller (the software layer above the HAL) is responsible for the "intelligence," which includes Logical Address Allocation


#### HdmiCecController:
* API:
    * requestActiveDisplay (oneTouchPlay related)
        * Called when a user presses the Home or Cast.
        * Success: The TV switches input to the OTT device and shows the content.
        * Error: The action is ignored and an error is reported.
    * requestStandby (key press related)
        * Initiates power-off. Broadcasts to all connected HDMI devices by default, or to a single device if specified.
        * Success: The TV/devices power down.
    * 



#### HdmiCecNetwork - cache layer
* Description:
    * Holds information about the current state of the HDMI CEC network. It is the sole source of
    * truth for device information in the CEC network.
* API:
    * getHdmiVersion
    * getVendorId
    * getPhysicalAddress

#### VSL API:

* onHotplug (plug/unplug)
    * Called when a hotplug event is detected
    * Args: bool connected
    * Return: None
* onCecMessage
    * Args:
        * src, dst, body - all bytes
* sendMessage
    * Args:
        * src  [byte]
        * dst  [byte]
        * body [byte]
* addLogicalAddress
    * Description:
        * Tell hardware to ACK any messages destined for <address>
* clearLogicalAddress
    * Description: 
        * Tells the hardware to stop responding to <address>

Android code flow:
* onHotplug:
    * mHdmiCecNetwork:initPortInfo
    *
