# UDP Connection Test

A dependency-free C++17 receiver for the JSON telemetry emitted by
`URocketUdpBridge`. It proves that live simulation data can cross a physical
Ethernet connection to a separate machine.

## Test layout

| Device | Ethernet IPv4 address | Prefix/mask |
|---|---:|---:|
| Windows simulation PC | `192.168.50.1` | `/24` (`255.255.255.0`) |
| Mac or PC receiver | `192.168.50.2` | `/24` (`255.255.255.0`) |

The Windows computer runs Unreal and sends UDP telemetry. The Mac (or a second
Windows computer) runs this receiver and listens on UDP port `5762`.

```text
Unreal on Windows (192.168.50.1)  --UDP/5762-->  Receiver (192.168.50.2)
```

No router or internet connection is required. A direct Ethernet cable is
sufficient; modern Ethernet adapters automatically handle cable crossover.

## Complete setup: Windows simulator to Mac receiver

### 1. Connect the computers

Connect the Windows computer and Mac with an Ethernet cable. A USB-to-Ethernet
adapter is fine. Keep Wi-Fi enabled if either computer still needs internet
access; the static addresses below apply only to the Ethernet interfaces.

### 2. Configure the Windows simulation computer

The Ethernet adapter may not appear as connected until the cable is plugged in.

1. Press `Win+R`, enter `ncpa.cpl`, and press Enter.
2. Identify the wired **Ethernet** adapter. Unplugging and reconnecting the cable
   is a quick way to confirm which adapter changes state.
3. Right-click that adapter and choose **Properties**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** and choose **Properties**.
5. Select **Use the following IP address** and enter:

   ```text
   IP address:      192.168.50.1
   Subnet mask:     255.255.255.0
   Default gateway: leave blank
   DNS servers:     leave blank
   ```

6. Click **OK**, then **Close**.

To confirm the setting, open PowerShell and run:

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object IPAddress -Like "192.168.50.*"
```

The result should show `192.168.50.1` with `PrefixLength` equal to `24` on the
wired adapter.

### 3. Configure the Mac receiver

1. Open **System Settings > Network** and select the wired Ethernet adapter. For
   a USB adapter, this may appear under its chipset name, such as `AX88179B`.
2. Open **Details > TCP/IP**.
3. Set **Configure IPv4** to **Manually** and enter:

   ```text
   IP address:  192.168.50.2
   Subnet mask: 255.255.255.0
   Router:      leave blank
   ```

4. Apply the changes. The interface should show **Connected**.

To identify the interface and verify its address from Terminal:

```sh
networksetup -listallhardwareports
ifconfig en14 | grep "inet "
```

Replace `en14` with the device name shown for your Ethernet adapter. The output
should include `192.168.50.2`.

### 4. Test the Ethernet link

From the Windows simulation computer, run:

```powershell
ping 192.168.50.2
```

Replies prove Windows can reach the receiver through the Ethernet subnet. You do
not need to run both pings simultaneously. A ping from the Mac to Windows may
time out because Windows commonly blocks inbound ICMP; that does not prevent
Windows from sending UDP telemetry to the Mac.

If Windows cannot ping the Mac, confirm that both wired interfaces show a link
and that their addresses are exactly `192.168.50.1/24` and
`192.168.50.2/24`.

### 5. Configure Unreal on Windows

1. Open `UnrealSpaceProgram.uproject` with Unreal Editor and open
   `Content/Levels/Map_Rocket`. This is also the project's default startup map.
2. In the World Outliner, select the `BP_Rocket` actor. If the level does not
   contain one, follow `Docs/ROCKET_SETUP.md` to create and place it.
3. In the actor's Components panel, select **UdpBridge**.
4. In **Details > HIL Bridge**, configure:

```text
bAutoStart = true
TelemetryAddress = 192.168.50.2
TelemetryPort = 5762
TelemetryRateHz = 30
```

5. Leave `ListenPort` at `5761`. That port is for commands sent back to Unreal;
   this receive-only connection test does not use it.
6. Compile and save `BP_Rocket` if you edited the Blueprint defaults, then save
   the level. Leave the flight controller in **Auto** mode for a hands-off run.

When Play starts, Unreal's Output Log should contain a message similar to:

```text
RocketUdpBridge: listening on UDP 5761, telemetry -> 192.168.50.2:5762 @ 30 Hz.
```

### 6. Build the receiver on the Mac

From the root of the `unreal-space-program` repository:

```sh
cd UDP-Connection
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

The build step only needs to be repeated after the receiver code changes.

### 7. Run the test

1. On the Mac, start the receiver before pressing Play in Unreal:

   ```sh
   cd /path/to/unreal-space-program/UDP-Connection
   ./build/udp_receiver --bind 192.168.50.2 --port 5762 --raw
   ```

2. Confirm it prints `Listening for Unreal rocket telemetry on
   192.168.50.2:5762`.
3. On Windows, press **Play** in Unreal.
4. Confirm the Mac prints numbered telemetry records and raw JSON packets.
5. Stop the receiver with `Ctrl+C` when the test is complete.

Successful reception proves all of the following:

- Unreal started the UDP bridge.
- Windows routed packets through the direct Ethernet connection.
- The receiver obtained and decoded live simulation telemetry.

The rocket's flight behavior is separate from this connection test. Even a
packet reporting `liftoff:false` proves the UDP path is working.

By default, every packet is also appended to `telemetry.jsonl`. The file is
ignored by Git and can be retained as test evidence.

## Using a second Windows computer as the receiver

Assign the receiver computer `192.168.50.2/24` using the same adapter procedure
above. Install Visual Studio 2022 with the **Desktop development with C++**
workload, then open Developer PowerShell in `UDP-Connection`:

```powershell
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
.\build\Release\udp_receiver.exe --bind 192.168.50.2 --port 5762 --raw
```

If prompted, allow the receiver through Windows Defender Firewall on private
networks. A manual rule can be added from an Administrator PowerShell:

```powershell
New-NetFirewallRule -DisplayName "Unreal Rocket UDP Telemetry" -Direction Inbound -Protocol UDP -LocalPort 5762 -Action Allow -Profile Private
```

## Troubleshooting

- **No Ethernet adapter:** connect the cable first, then reopen `ncpa.cpl`.
- **Windows can ping the Mac, but no telemetry appears:** verify `bAutoStart`,
  `TelemetryAddress`, and `TelemetryPort` on the actual `BP_Rocket` instance in
  `Map_Rocket`; then check Unreal's Output Log for `RocketUdpBridge`.
- **Receiver reports a bind error:** another process may already be using UDP
  port `5762`, or `192.168.50.2` is not assigned to that machine. Binding to
  `0.0.0.0` is a useful diagnostic: `./build/udp_receiver --raw`.
- **Mac-to-Windows ping fails:** test Windows-to-Mac instead. Windows firewall
  often blocks ping replies and outbound Unreal telemetry can still work.
- **Packets arrive over Wi-Fi:** both static `192.168.50.x` addresses exist only
  on Ethernet, so targeting `192.168.50.2` selects that subnet. Disable Wi-Fi
  temporarily if you want an additional physical-link demonstration.
- **Need packet-level proof:** capture on the receiver's Ethernet interface in
  Wireshark with the display filter `udp.port == 5762`.

## Options

```text
--bind ADDRESS   Local address to listen on (default 0.0.0.0)
--port PORT      UDP port to listen on (default 5762)
--log PATH       Capture raw packets at PATH
--no-log         Disable capture-file output
--raw            Print the complete JSON packet
--help           Show usage
```

The parser is separated from the desktop socket layer so its telemetry model
and validation behavior can later be reused with a microcontroller Ethernet
stack such as lwIP or a W5500 driver.
