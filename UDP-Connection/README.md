# UDP Connection Test

A dependency-free C++17 receiver for the JSON telemetry emitted by
`URocketUdpBridge`. It proves that live simulation data can cross a physical
Ethernet connection to a separate machine.

## Direct Ethernet setup

| Device | Ethernet IPv4 address | Prefix/mask |
|---|---:|---:|
| Windows simulation PC | `192.168.50.1` | `/24` (`255.255.255.0`) |
| Mac or PC receiver | `192.168.50.2` | `/24` (`255.255.255.0`) |

On `BP_Rocket`, select `UdpBridge` and configure:

```text
bAutoStart = true
TelemetryAddress = 192.168.50.2
TelemetryPort = 5762
TelemetryRateHz = 30
```

## Build and run on macOS

From this `UDP-Connection` directory:

```sh
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
./build/udp_receiver --bind 192.168.50.2 --port 5762 --raw
```

Start the receiver before pressing Play in Unreal. Raw JSON packets are also
appended to `telemetry.jsonl` by default. Press Ctrl-C to stop.

## Build and run on Windows

Install Visual Studio 2022 with the **Desktop development with C++** workload,
then open Developer PowerShell in this directory:

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
