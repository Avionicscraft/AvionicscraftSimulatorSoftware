# Avionicscraft Simulator Software

Avionicscraft Simulator Software connects Ethernet-based flight simulator panels controlled by a microcontroller (MCU) to [DCS-BIOS](https://github.com/DCS-Skunkworks/dcs-bios). It provides one Windows control center for discovering modules, monitoring their live state, forwarding physical controls to DCS, and distributing simulator output back to the panels.

The software is designed for modular home cockpit hardware. Panels obtain an IP address through DHCP and automatically locate DCS-LINK, so fixed panel IP addresses are not required.

Website: [avionicscraft.com](https://avionicscraft.com/)

## Main features

- Automatic MCU-based module discovery over Ethernet
- DHCP-based networking without fixed panel addresses
- One DCS-BIOS connection shared by all connected modules
- Live module list with aircraft, module name, MAC address and IP address
- Detailed module status showing component names, GPIO pins, labels and current values
- LED backlight testing from the module status window
- Automatic reconnection after temporary network interruptions
- Optional automatic startup with Windows
- Configurable activity log retention
- Trusted MAC address filtering
- Mission input synchronization

o all connected modules.

## Mission input synchronization

DCS can reset cockpit controls while a mission is loading. A physical switch may therefore differ from its virtual cockpit switch even though the physical switch has not moved.

DCS-LINK solves this by storing the latest numeric value of every connected module input:

- All known input states are sent immediately after DCS-BIOS connects.
- The complete input state is sent again once per second.
- Switches, selectors and potentiometers remain authoritative during the mission.
- Indicator lights, LED feedback and other outputs are excluded from synchronization.

If a virtual control is changed with the mouse while its physical control remains in another position, Mission Sync restores the physical position within approximately one second.

## Requirements

### Windows application

- Windows 10 or Windows 11
- [.NET 6 Desktop Runtime](https://dotnet.microsoft.com/download/dotnet/6.0), or the .NET 6 SDK when building from source
- DCS World with DCS-BIOS installed and running
- A private Ethernet network that allows local UDP broadcast traffic

### Panel hardware

- A compatible microcontroller (MCU) board with an Ethernet interface
- Arduino `Ethernet` library
- `DCS-BIOS Arduino Library`
- A unique MAC address for every module

## Quick start

1. Install and configure DCS-BIOS on the DCS computer.
2. Start `AvionicscraftSimulatorSoftware.exe`.
3. Enter the IP address of the computer running DCS and DCS-BIOS.
   - Use `127.0.0.1` when DCS-LINK runs on the same computer.
   - Use the DCS computer's LAN address when it runs on another computer.
4. Keep the default DCS-BIOS port `7778` unless your configuration uses another port.
5. Select **DCS-BIOS** as the output mode.
6. Click **Start**.
7. Power the panel MCUs. They should appear automatically under **Connected Modules**.

The connection light indicates the DCS-BIOS state:

| Colour | Meaning |
|---|---|
| Green | Connected to DCS-BIOS |
| Orange | Connecting or retrying |
| Red | DCS-LINK is stopped or DCS-BIOS is unavailable |

Only one copy of Avionicscraft Simulator Software can run DCS-LINK at a time. A second copy cannot use the same discovery and panel ports.

## Output modes

- **DCS-BIOS** is the available output mode.
- **Virtual joystick (Coming Soon!)** is visible in the interface but disabled. It cannot be selected or started.

## Module status

Double-click a connected module, or right-click it and select **View module status**. The status window displays:

- Component and control name
- Input or output type
- GPIO connection
- Hardware label number
- Current switch, selector or potentiometer state
- Brightness percentage where applicable
- Time of the latest update

Three-position controls are shown as **UP position**, **MID position** and **DOWN position**. Module firmware normally sends a complete status snapshot every 100 ms.

Modules with a PWM backlight can use **Push to test LED strip**. Hold the button to request 100% brightness and release it to restore the DCS-controlled value. Use a suitable transistor or MOSFET driver for an LED strip; an MCU GPIO must not power the strip directly.

## Panel firmware

The `DcsPanelHub` folder contains firmware for several compatible panel configurations. Specific panel names are intentionally omitted from this overview because the same DCS-LINK protocol can be used for different aircraft, panels and control layouts.

Before uploading firmware, verify its GPIO assignments and give the module a unique MAC address. Duplicated MAC addresses cause unreliable discovery and reconnection behaviour.

## Network ports

| Port | Protocol | Direction | Purpose |
|---:|---|---|---|
| 7778 | TCP | DCS-LINK to DCS-BIOS | DCS-BIOS command and export stream |
| 39000 | UDP | MCU to DCS-LINK | Automatic module discovery |
| 39001 | TCP | MCU to DCS-LINK | Persistent panel data connection |
| 39002 | UDP | DCS-LINK to MCU | Module commands such as LED testing |

Allow the application through Windows Firewall on private networks. Panel MCUs and DCS-LINK must normally be on the same LAN or VLAN because discovery broadcasts do not usually cross routers.

## Module protocol

The current text-based protocol is intentionally simple so additional modules can be developed independently.

### Discovery request

```text
DCS_PANEL_DISCOVER|MAC_ADDRESS|PANEL_NAME
```

### Discovery response

```text
DCS_PANEL_HUB|39001
```

### Panel identity

```text
DCS_PANEL_HELLO|MAC_ADDRESS|AIRCRAFT|MODULE
```

### DCS-BIOS input command

```text
CONTROL_IDENTIFIER VALUE
```

Example:

```text
CONTROL_NAME 2
```

### Live component state

```text
PANEL_STATE|COMPONENT_ID|VALUE|DISPLAY_NAME|IO_TYPE|GPIO|LABEL_NUMBER
```

The metadata fields allow the Windows application to build the module status window without module-specific UI code.

## Build from source

From the repository root:

```powershell
dotnet run --project .\DcsPanelHub\DcsPanelHub.csproj
```

Create a framework-dependent Release build:

```powershell
dotnet build .\DcsPanelHub\DcsPanelHub.csproj -c Release
```

Create a self-contained Windows x64 release:

```powershell
dotnet publish .\DcsPanelHub\DcsPanelHub.csproj -c Release -r win-x64 --self-contained true
```

Application settings are stored in:

```text
%LOCALAPPDATA%\AvionicscraftSimulatorSoftware\settings.json
```

## Troubleshooting

### A module does not appear

- Confirm that DCS-LINK is started.
- Close other copies of Avionicscraft Simulator Software.
- Check the Ethernet cable, PoE supply and DHCP server.
- Confirm that the module and computer are on the same LAN or VLAN.
- Verify that every module has a unique MAC address.
- Allow UDP 39000 and TCP 39001 on private networks.
- Open Serial Monitor at 115200 baud and check the DHCP and discovery messages.

### The DCS-BIOS light stays orange

- Confirm that DCS-BIOS is installed and running.
- Verify the DCS computer IP address and TCP port 7778.
- Use `127.0.0.1` only when DCS and DCS-LINK run on the same computer.
- Check Windows Firewall on the DCS computer.

### A module connects but its controls do not work

- Select **DCS-BIOS** output mode.
- Confirm that the correct aircraft module is active in DCS-BIOS.
- Verify the DCS-BIOS control identifiers in the firmware.
- Check the GPIO assignments and switch wiring.
- Ensure switch common terminals are connected to GND when using `INPUT_PULLUP`.

### An LED strip does not respond

- Use **Push to test LED strip** in the module status window.
- Check that the module reports `LED_PWM_GPIO43`.
- Verify the LED power supply, common ground and external driver.
- Check whether the driver expects active-high or active-low PWM.

### The physical and virtual controls do not match after mission load

- Wait for the DCS-BIOS status light to turn green.
- Confirm that the module reports its inputs in **View module status**.
- Mission Sync should restore all known numeric input states within approximately one second.

## Windows startup

Click **Enable Windows startup** to launch the application automatically after signing in. The same button changes to **Disable Windows startup** when the feature is enabled. The DCS computer address, port, output mode and log retention setting are saved locally.
