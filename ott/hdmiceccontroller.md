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
        * src, dst, body - all bytesO
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

    
VRR - 

    
LLM:
* Inputs:
    *   Action: When you connect the HDMI cable, the TV sends its EDID (Extended Display Identification Data) to the Android device.
    *   Data: The EDID contains an HDMI Forum VSDB (Vendor Specific Data Block).
    *   Bit Flag: Inside this block, there is a specific bit flag for Auto Low Latency Mode (ALLM) support
* HWC (has get API)
    *   AIDL/HIDL Method: getDisplayCapabilities(Display display)
    *   Value: It includes the enum DisplayCapability::AUTO_LOW_LATENCY_MODE. 
    

##### The Validator: The Kernel Driver (DRM/KMS)
*   Who: The Linux Kernel Display Driver (provided by the SoC vendor like Amlogic, Realtek, MediaTek).
*   Responsibility: This is the Most Critical Step.
*   What it does:
    1.  Reads the EDID from the TV.
    2.  Checks the SoC Hardware Capabilities (e.g., "My video processor can only do 4K @ 30Hz").
    3.  Checks the HDMI PHY Status (e.g., "Link training failed at 6Ghz, falling back to 3Ghz").
    4.  Calculates Bandwidth: It runs the math: 4K + 60Hz + 10bit + RGB = 22Gbps.
    5.  Pruning: If the link is only HDMI 2.0 (18Gbps), the Driver MUST either:
        *   Delete the 4K60 RGB mode from the list.
        *   Modify the mode to use YUV 4:2:0 (which fits in 18Gbps).
*   The Output: A list of "Validated Modes" exposed to the OS (usually via DRM ioctls).

Files:
1. drivers/gpu/drm/display/drm_hdmi_state_helper.c
2. drivers/gpu/drm/display/drm_hdmi_helper.c (implements the official HDMI specification) (drm_hdmi_compute_mode_clock)

    
```
device = new LocalDisplayDevice(displayToken, physicalDisplayId, staticInfo, dynamicInfo, modeSpecs, isFirstDisplay);
sendDisplayDeviceEventLocked(device, DISPLAY_DEVICE_EVENT_ADDED);
```
1. StaticDisplayInfo (The Identity)
This object contains the immutable properties of the physical display. These values never change as long as the device is connected.
*   Source: Read from EDID (External) or Driver Hardcoding (Internal).
*   Key Fields:
    *   connectionType: Is it Internal (Phone Screen), External (HDMI), or Virtual?
    *   densityDpi: The physical pixel density.
    *   secure: Does the hardware support DRM protection (HDCP)?
    *   deviceProductInfo: Manufacturer Name (e.g., "Samsung"), Model Year, Sink ID.
*   When it changes: Only on Hotplug (unplugging/replugging the cable).
---
2. DynamicDisplayInfo (The Capabilities)
This object contains the current capabilities of the display. This is the "Filtered List" we discussed earlier—it is what the Kernel/HAL has validated is safe to use.
*   Source: Calculated by HWC/Kernel based on Bandwidth & EDID.
*   Key Fields:
    *   J: The list of valid modes (e.g., 4K60, 1080p60). This is where bandwidth-limited modes disappear.
    *   activeDisplayModeId: Which mode is currently running.
    *   hdrCapabilities: Supported HDR types (Dolby Vision, HDR10, HLG).
    *   supportedColorModes: RGB, DCI-P3, Native.
    *   autoLowLatencyModeSupported: Boolean (True/False).
*   When it changes:
    *   On Hotplug.
    *   On Driver Renegotiation: If the HDMI link quality drops and the driver forces a fallback (e.g., from HDMI 2.1 to 2.0), this info updates to remove the high-bandwidth modes.
---
3. DesiredDisplayModeSpecs (The Command)
This is NOT a description of the hardware. It is a Configuration Request from the Android System (DisplayModeDirector) down to the HAL.
*   Source: DisplayManagerService (Calculated from App Votes, Battery Saver, User Settings).
*   Key Fields:
    *   baseModeId: The "Preferred" resolution (e.g., "I want 4K").
    *   primaryRefreshRateRange: A constraint (e.g., min=60Hz, max=120Hz).
    *   appRequestRefreshRateRange: Overrides from Game Mode (e.g., min=120Hz, max=120Hz).
    *   allowGroupSwitching: Whether the HAL is allowed to switch between different resolution groups seamlessy.
*   How it works:
    *   Android tells the HAL: "I want Mode ID 5, but I accept any refresh rate between 60 and 120."
    *   The HAL picks the best match from DynamicDisplayInfo that satisfies these specs.


    
Android code flow:
* onHotplug:
    * mHdmiCecNetwork:initPortInfo
    *

    
    
Match content:
* input: video data. (4k@60 RGB  4k60 YUV 4.2.0) 
1. if I can only pick my timing, 
4k30 4:2:2
