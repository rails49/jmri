# How control programs reach many command stations, and what it means for rails49

The decision went the other way from section 4's recommendation. The JMRI
translator is parked, and a station other than DCC-EX gets a translator of its
own when a layout needs one
([rails49/.github#83](https://github.com/rails49/.github/issues/83)).

Answers four questions. How JMRI, Rocrail and iTrain support so many command
stations, and whether a few wire protocols cover most of them. Whether those
projects buy and test every station. How much code each protocol takes in
JMRI, and which connections are thin layers over another. And which route
rails49 should take past DCC-EX: through JMRI, through protocols of its own,
or through one common LAN protocol.

[command-stations.md](command-stations.md) maps each capability control
expects onto DCC-EX and onto JMRI's core interfaces, and lists what JMRI's
JSON servlet cannot carry. This note does not repeat that; its Gaps section is
what option (a) below inherits. The planned translator is #1 to #7, and #2
writes those gaps down as ADR-0001.

Sources are pinned:

- `JMRI/JMRI@4ccc481a3a53af1df7d74f7bc2318d1c7ffd36b5` (master, 2026-11-24),
  cloned shallow and read only. Paths without a repository name are into this
  checkout: source under `java/src/jmri/`, tests under `java/test/jmri/`, help
  pages under `help/en/html/`.
- `rails49/dccex@d36d127` (main, 2026-11-24). `src/dccex/commands.py`,
  `replies.py`, `translator.py`.
- `rails49/CommandStation-EX@5828d3c` (2026-10-22), searched for `Z21` and
  `WiThrottle` only.
- Rocrail driver sources from `schelli04/Rocrail@be76c08` (2015-01-30), a
  GitHub mirror of Rocrail's source from before it closed. The
  `rocrail/Rocrail` repository on GitHub is empty ("This repository is empty",
  GitHub API, 2026-11-26). The Rocrail wiki answered 403 to every fetch.
- iTrain's list of digital systems,
  <https://berros.eu/en/itrain/systems.php> (read 2026-11-26). iTrain is
  closed source.
- Z21 LAN Protocol Specification, document version 1.13 en, 06.11.2023,
  [z21.eu PDF](https://www.z21.eu/media/Kwc_Basic_DownloadTag_Component/root-en-main_47-1652-959-downloadTag-download/default/d559b9cf/1558675126/z21-lan-protokoll-en.pdf).
  Cited by section and page.
- Digitrax LocoNet support page and its terms,
  <https://www.digitrax.com/support/loconet/>; LocoNet Personal Edition,
  <https://www.digitrax.com/support/loconet/loconetpersonaledition.pdf> (not
  read beyond its terms).
- YaMoRC YD7010 manual,
  [PDF](https://cdn.abicart.com/shop/578/art53/221851953-d9cded-YD7010.en.pdf),
  page 4. ZIMO MX10 manual, <https://www.zimo.at/web2010/documents/MX10_EN.pdf>,
  chapter 9, page 43.

Line counts are physical lines of `.java` (JMRI), `.c`/`.h` (Rocrail) or `.py`
(rails49) files, comments included. JMRI's "code" column drops blank and
comment lines. "GUI" is the `swing` subpackages: monitors, configuration
tools and throttles windows, none of which a translator would use.

## 1. A few protocols, many brands

JMRI groups code by the wire protocol, not by the brand. Its system structure
page says the code for one general type goes in one package under
`jmri.jmrix`, with each interface or connection mode a subpackage
(`help/en/html/doc/Technical/SystemStructure.shtml:30-52`). Brands are a label
on top: each package registers a `ConnectionTypeList` naming the
manufacturers whose menus list its connection classes. Several brands point
at another brand's package:

- "Roco" lists the Lenz XpressNet connections, then Z21
  (`jmrix/roco/RocoConnectionTypeList.java:22-33`).
- "Atlas" is a second name in the Lenz list
  (`jmrix/lenz/LenzConnectionTypeList.java:16-33`).
- "Uhlenbrock" lists LocoNet connections, with the Intellibox as a LocoNet
  adapter (`jmrix/loconet/uhlenbrock/UhlenbrockConnectionTypeList.java:24-28`;
  `IntelliboxAdapter extends LocoBufferAdapter`,
  `jmrix/loconet/Intellibox/IntelliboxAdapter.java:17`).
- "Hornby" is a subpackage of `lenz`
  (`jmrix/lenz/hornbyelite/EliteConnectionTypeList.java:15-22`).
- "Wangrow" is one serial adapter that extends NCE's
  (`jmrix/wangrow/serialdriver/SerialDriverAdapter.java:22`).
- "Digikeijs" has no package at all. Its help page tells the user to pick
  "ESU / ECoS via network" or "Digitrax / LocoNet over TCP" or "Digitrax /
  LocoNet LocoBuffer-USB" (`help/en/html/hardware/digirails/index.shtml:41-81`).

The table maps each protocol to the stations that speak it and to the JMRI
code behind it. "Tests" is the matching package under `java/test/jmri/jmrix/`.
"Simulator" is a connection a user can select without hardware; "scaffold" is
the test double described in section 2.

| Protocol and owner | Command stations that speak it | JMRI package | Source files / lines (code) / of which GUI | Test files / lines | Simulator, scaffold |
| --- | --- | --- | --- | --- | --- |
| LocoNet (Digitrax). Personal Edition public, for non-commercial use; a licence for products | Digitrax DCS50, 51, 52, 100, 200, 210, 210+, 240, 240+, DB150 (`jmrix/loconet/LnCommandStationType.java:42-228`); Uhlenbrock Intellibox I and II (`:289`, `:309`); Fleischmann Twin Centre (`hardware/index.shtml:746-750`); Digikeijs DR5000 and YaMoRC stations over LocoNet over TCP (`digirails/index.shtml:54-70`); YaMoRC YD7010 over USB and LAN (manual p. 4); black Z21's LocoNet port (`roco/z21/index.shtml:67`) | `loconet` | 368 / 71,585 (44,703) / 10,096 | 327 / 40,756 | `hexfile` simulator; `LocoNetInterfaceScaffold` |
| XpressNet (Lenz) | Lenz LZ100, LZV100, LZV200, Compact, LH200; OpenDCC; Paco NanoX; Roco LocoMaus 2 and multiMaus; Hornby Elite; Atlas Commander; ZTC511; listed as "untested": ZTC521, Hornby Select, Viessmann Commander, Roco multiZENTRALpro (`hardware/XPressNet/index.shtml:135-212`); YaMoRC YD7010 (manual p. 4) | `lenz` (Hornby Elite in `lenz/hornbyelite`, 15 files, 1,226 lines) | 210 / 25,367 (14,959) / 3,682 | 211 / 19,021 | `xnetsimulator`; `XNetInterfaceScaffold` |
| Z21 LAN (Roco). Public spec, UDP 21105 (§1.1, p. 8) | Roco Z21 and z21, also sold as Fleischmann (`roco/z21/index.shtml:17-20`); YaMoRC YD7010 (manual p. 4: "Z21® (Network)"); ZIMO MX10 with the Roco Z21 app (MX10 manual ch. 9, p. 43). DR5000: reported by a magazine review, not checked against a primary source | `roco/z21` | 95 / 10,131 (6,238 for all of `roco`) / 1,152 | 97 / 6,970 (all of `roco`) | `z21/simulator`; `Z21InterfaceScaffold` |
| Märklin CAN (Märklin) | Central Station 2 and 3, Mobile Station 2; CAN-digital-Bahn CC-Schnitte (`hardware/marklin/index.shtml:17-36`) | `marklin` | 122 / 17,165 (10,050) / 3,255 | 79 / 9,836 | `simulation`; `MarklinTrafficControlScaffold` |
| ECoS (ESU) | ESU ECoS (`hardware/ecos/index.shtml:20`); Digikeijs DR5000 on port 15471 (`digirails/index.shtml:41-47`) | `ecos` | 58 / 9,939 (6,925) / 2,590 | 55 / 2,941 | none; `EcosInterfaceScaffold` |
| DCC-EX native (DCC-EX) | DCC-EX EX-CommandStation | `dccpp` | 108 / 21,146 (14,016) / 5,789 | 87 / 9,069 | `simulator`; `DCCppInterfaceScaffold` |
| NCE binary (NCE) | NCE Power Pro, PowerCab (`hardware/index.shtml:854-856`); Wangrow System One | `nce`, `wangrow` | 87 / 22,683 (15,372) / 246 in `swing`, more tools in other subpackages | 80 / 4,756 | `simulator`; `NceInterfaceScaffold` |
| P50X (Uhlenbrock, binary) | TAMS MasterControl (`jmrix/tams/TamsPowerManager.java:10`); iTrain also lists Intellibox and TAMS on P50X | `tams` | 47 / 5,516 (3,347) / 1,536 | 42 / 1,859 | `simulator`; `TamsInterfaceScaffold` |
| LCC / OpenLCB (NMRA) | TCS CS-105 (`hardware/index.shtml:161`) | `openlcb` plus the CAN adapters in `can`, plus `lib/openlcb.jar` 0.8.6 (`pom.xml:30`) | 106 / 21,190 (14,487) / 14,179 | 78 / 9,646 | CAN loopback |
| CBUS (MERG) | MERG command station, SPROG Generation 5 (`jmrix/sproggen5/SprogGen5ConnectionTypeList.java:23-27`) | `can` (CBUS in `can/cbus`) | 333 / 57,403 (34,885) | 311 / 37,721 | `can/cbus/simulator`; `CbusTrafficControllerScaffold` |
| BiDiB (open, bidib.org) | BiDiB command stations and nodes | `bidib` plus `lib/jbidibc-*.jar` | 69 / 15,154 (9,189) / 1,846 | 64 / 3,284 | `simulator`; `BiDiBInterfaceScaffold` |
| SRCP (open, a server protocol) | srcpd and other SRCP servers, not stations | `srcp` | 37 / 3,503 (2,009) | 36 / 3,465 | none; `SRCPTrafficControlScaffold` |
| One brand each | SPROG (`sprog`, 87 / 12,495), Zimo MX-1 (`zimo`, 42 / 4,976), MRC (`mrc`, 40 / 5,685), EasyDCC (`easydcc`, 43 / 4,304), Lionel TMCC (`tmcc`, 35 / 5,764) | | | | each has a simulator |

All of `jmrix` is 2,948 files and 456,840 lines; its tests are 219,661 lines.
That is a third of JMRI's 1,345,659 lines under `java/src`. The shared base
in `jmrix/*.java` is 54 files and 15,022 lines, among them
`AbstractMRTrafficController` (1,509), `AbstractThrottleManager` (1,099) and
`AbstractThrottle` (1,193).

The other two programs show the same shape.

- Rocrail's 2015 source has one driver per protocol under `rocdigs/impl/`.
  The XpressNet driver has one file per interface: `atlas.c`, `elite.c`,
  `li101.c`, `liusb.c`, `lieth.c`, `opendcc.c`, `xntcp.c` under
  `rocdigs/impl/xpressnet/`. The LocoNet driver has one per transport:
  `lbserial.c`, `lbtcp.c`, `lbudp.c`, `lbserver.c`, `ulni.c` under
  `rocdigs/impl/loconet/`. In all, 52 drivers and 75,828 lines.
- iTrain's page lists 34 command stations and 7 other devices. Where it names
  a protocol, the same few recur: "Intellibox (P50X)", "Intellibox II, III,
  Basic and COM" and "LocoNet (serial, USB and TCP/IP)" on LocoNet, "Lenz
  XpressNet (serial, USB and LAN/Ethernet)", "Roco Z21 (including the white
  z21)", BiDiB, DCC-EX. It does not say how its code is organised.

So the answer to the first question is yes. Five protocols carry most
commercial stations: LocoNet, XpressNet, Z21 LAN, Märklin CAN and ECoS, with
NCE for much of the North American market and DCC-EX for the DIY one. A
newer station often speaks several of them at once. The YD7010 offers
LocoNet, XpressNet, Z21 and WiThrottle (manual p. 4). The DR5000 offers ECoS
and LocoNet over TCP to JMRI. The Z21 carries XpressNet inside its own
packets and tunnels LocoNet (section 3). The long tail is single-brand
protocols: Zimo MX-1, MRC, EasyDCC, TMCC, SPROG, Selectrix and the others on
iTrain's list.

## 2. Who tests against which hardware

Nothing found in JMRI's repository describes tests against real stations.
Its tests run against software stand-ins, and its help pages say which
stations someone has run.

- Unit tests replace a connection's traffic controller with a scaffold.
  `LocoNetInterfaceScaffold` extends `LnTrafficController`, keeps every
  message the code under test sends in a public `outbound` vector, and lets
  the test inject replies with `sendTestMessage`
  (`java/test/jmri/jmrix/loconet/LocoNetInterfaceScaffold.java:14`, `:30`,
  `:69`). 47 such scaffolds exist under `java/test/jmri/jmrix/`, one or more
  per connection, ECoS, Z21, Märklin, NCE, DCC-EX and XpressNet among them.
  These check that JMRI builds the bytes it means to; they cannot check what
  a station does with them.
- The simulators are thin. The XpressNet one says "Most XpressNet commands
  are still unsupported in this implementation"
  (`jmrix/lenz/xnetsimulator/XNetSimulatorAdapter.java:28`); the DCC-EX one
  says the same of DCC-EX (`jmrix/dccpp/simulator/DCCppSimulatorAdapter.java:36`)
  and was based on the XpressNet one (`:47`). NCE's simulator copies its
  command set from the Power Pro System Reference Manual
  (`jmrix/nce/simulator/SimulatorAdapter.java:13-15`).
- The acceptance tests start the applications with simulator profiles only:
  LocoNet, TMCC, Grapevine, SPROG and EasyDCC
  (`java/acceptancetest/features/apps/ApplicationStart.feature:11-60`;
  `help/en/html/doc/Technical/ContinuousIntegration.shtml:286-289`).
- The help pages state what was run on hardware, by whom, and what was not.
  XpressNet sorts stations into Full, Partial and Minimal and lists four as
  "untested, but should be at least partially supported"
  (`hardware/XPressNet/index.shtml:135-212`). MRC "has only been tested and
  developed with the Advanced2" (`hardware/mrc/index.shtml:28-32`). Märklin
  "was developed and tested on CS2 firmware version 2"
  (`hardware/marklin/index.shtml:55-56`). Digikeijs: "Please report your
  results" (`hardware/digirails/index.shtml:26`). Z21 credits "Paul Bender and
  others" (`hardware/roco/z21/index.shtml:182`), and the Z21 connection and
  its simulator are still marked `// experimental` in the menu list
  (`jmrix/roco/RocoConnectionTypeList.java:32-33`).
- Authorship follows the hardware. Paul Bender is on 154 `@author` lines in
  `lenz` and 81 in `roco`; Mark Underwood leads `dccpp` with 61; B. Milhaupt
  is second in `loconet`; Ken Cameron is second in `nce`; Kevin Dickerson
  leads `tams` and is second in `marklin` and `ecos`. Bob Jacobsen appears
  in each package counted, which fits the guide's worked example, which
  starts by copying an existing system
  (`help/en/html/doc/Technical/NewSystem.shtml:37`).

The pattern is that a connection exists because someone who owns the station
wrote it, and works as far as that person and later users have run it. The
project does not hold a set of stations. This is read from the code and help
pages; I did not read JMRI's mailing lists or ask its maintainers.

Rocrail and iTrain could not be checked. The Rocrail wiki was not reachable,
and its current source is not public. iTrain's systems page makes no
statement about testing, beta status or who wrote what.

## 3. How much code, and which connections sit on another

The table above gives the totals. What a translator would replace is the
non-GUI part: `loconet` 61,489, `lenz` 21,685, `nce` 22,437, `dccpp` 15,357,
`marklin` 13,910, `bidib` 13,308, `roco/z21` 8,979, `ecos` 7,349, `tams`
3,980, `srcp` 2,839 lines. A JMRI connection carries far more than driving:
programming, consists, monitors, slot tables, configuration of the station's
own peripherals. LocoNet's size is mostly that: its subpackages include
`ds64`, `bdl16`, `pm4`, `se8`, `locoio`, `lnsvf2`, `sdfeditor`,
`soundloader`, `downloader`.

Rocrail's 2015 drivers, which do driving, accessories, feedback and some
programming, are smaller: `z21` 1,687, `mcs2` (Märklin CAN) 1,679, `ecos`
2,837, `xpressnet` 3,498, `loconet` 8,497, `srcp` 1,667, `nce` 611 lines of
C.

Connections that sit on another protocol's code:

- Z21 over XpressNet and LocoNet. The Z21 protocol carries XpressNet bytes
  inside its own records: `LAN_X_…` commands have an X-header and an XOR
  byte (spec §2.13, p. 15). JMRI runs its whole Lenz stack on top.
  `Z21XPressNetTunnel` pipes Z21 records into an `XNetStreamPortController`
  (`jmrix/roco/z21/Z21XPressNetTunnel.java:16-19`, `:47-67`), and 1,565 lines
  of `Z21XNet*` classes subclass the Lenz ones: `Z21XNetThrottle extends
  RocoXNetThrottle extends XNetThrottle`, likewise turnout, programmer,
  packetizer, message and reply (`jmrix/roco/z21/Z21XNet*.java`,
  `jmrix/roco/RocoXNetThrottle.java:16`). `Z21LocoNetTunnel` (325 lines) and
  `Z21LnStreamPortController` do the same with LocoNet
  (`Z21SystemConnectionMemo.java:178-181`). The Z21 memo registers only a
  `CommandStation`, a `SensorManager` for R-Bus and CAN detectors and a
  `ReporterManager` for RailCom of its own; throttles, turnouts and power
  come from the tunnelled XpressNet connection.
- Hornby Elite over XpressNet: every class in `lenz/hornbyelite` extends a
  Lenz one (`EliteXNetThrottle.java:16`, `EliteXNetTurnout.java:12`,
  `EliteAdapter extends XNetSerialPortController`, `EliteAdapter.java:14`).
- Roco multiMaus, Atlas Commander, Viessmann Commander, ZTC: no code of their
  own; they are entries in the Lenz menu.
- Uhlenbrock Intellibox over LocoNet: `IBLnPacketizer extends LnPacketizer`,
  `IntelliboxAdapter extends LocoBufferAdapter`
  (`jmrix/loconet/Intellibox/*.java`, 7 files, 593 lines).
- Digikeijs DR5000 and YaMoRC: no code; they use the ECoS or LocoNet
  connections unchanged.
- Wangrow over NCE: one adapter, 6 files, 411 lines.
- SPROG Generation 5 over CBUS: its connection classes live in
  `can/adapters/gridconnect/sproggen5`.

The core interfaces each connection implements are in `java/src/jmri/`:
`ThrottleManager` (`ThrottleManager.java:30`) handing out a `DccThrottle`
(`DccThrottle.java:20`, `Throttle.java:32`); `PowerManager`
(`PowerManager.java:39`); `TurnoutManager` (`TurnoutManager.java:44`);
`SensorManager` (`SensorManager.java:23`); `CommandStation`, whose one job is
`sendPacket(byte[], int)` for raw DCC packets (`CommandStation.java:21`,
`:32`); `Programmer` (`Programmer.java:52`) through `GlobalProgrammerManager`
and `AddressedProgrammerManager` (`GlobalProgrammerManager.java:37`,
`AddressedProgrammerManager.java:40`); plus `LightManager`,
`ReporterManager` and `ConsistManager`. A connection's
`SystemConnectionMemo` (`SystemConnectionMemo.java:19`) registers the ones it
has, as DCC-EX's does at `jmrix/dccpp/DCCppSystemConnectionMemo.java:89-183`.

Not every connection has every one. Read by class name in each package:

| Package | Throttle | Power | Turnout | Sensor | Reporter | CommandStation | Programmer |
| --- | --- | --- | --- | --- | --- | --- | --- |
| loconet | yes | yes | yes | yes | yes | yes (`SlotManager`) | yes |
| lenz | yes | yes | yes | yes | – | yes | yes |
| roco/z21 | through XpressNet | through XpressNet | through XpressNet | yes | yes | yes | through XpressNet |
| dccpp | yes | yes | yes | yes | – | yes | yes |
| nce | yes | yes | yes | yes | – | yes | yes |
| ecos | yes | yes | yes | yes | yes | – | yes |
| marklin | yes | yes | yes | yes | – | – | – |
| openlcb | yes | yes | yes | yes | yes | yes | yes |
| tams | yes | yes | yes | yes | – | – | yes |
| zimo | yes | yes | yes | – | – | yes | yes |
| srcp | yes | yes | yes | yes | – | – | yes |

A missing `CommandStation` means no DCC signal mast on that connection
(command-stations.md, row 6). How a connection fills an interface differs too.
JMRI's Märklin power manager reads both "system stop" and "system halt" as
OFF (`jmrix/marklin/MarklinPowerManager.java:67-71`), so a halt with the
rails live is not told apart from power off. ECoS has no consists
(`hardware/ecos/index.shtml:38`). `PowerManager.IDLE` is implemented by
XpressNet, by Z21 through it, and by five LocoNet stations only
(command-stations.md, row 8).

## 4. Options for rails49

What a native driver costs is known from `rails49/dccex`. The protocol itself
is `commands.py` (224 lines) and `replies.py` (401). `translator.py` (1,514)
holds the bus side, the link poll, restart detection, the railroad's script
and the journal. The whole package is 3,314 lines, with 4,703 lines in
`tests/dccex/`. DCC-EX is a text protocol and rails49 owns the firmware, so
this is a lower bound for a binary protocol on a station rails49 does not
control.

### (a) Through JMRI, by the planned translator

Cost: issues #1 to #7, which are written. One driver, against JMRI's JSON
websocket, that is the same for every station. JMRI runs on the box with its
profile, a Jython startup script for the link (#5) and another for the
station setup (#6).

What it reaches: every connection in section 1 that registers throttle,
power and turnout managers, and sensors where the connection has them.
Programming stays in DecoderPro.

What it loses: the Gaps section of command-stations.md, on every station. No
emergency stop with the rails live over JSON, even where the connection
implements `IDLE`. No host timeout; #4 stands in for `layout` going quiet,
and nothing covers the translator or JMRI dying. No tripped district. No
station restart. Link only through the script in #5. On top of that, each
connection is as good as its contributor made it (section 2), and rails49
cannot test a station it does not own any better than JMRI can. Two more
processes sit between `layout` and the rails.

A variant closes part of the gap: run the translator's JMRI side as a Jython
script inside JMRI, against the core interfaces rather than the JSON
servlet. That reaches `PowerManager.IDLE` where a connection has it and
`ConnectionStatus` directly. It ties rails49 code to JMRI's Java API and its
release cycle, and needs JMRI running to test. #5 and #6 already put two
such scripts in the profile.

### (b) Native drivers for a few protocols

Cost: per protocol, a commands and replies pair, a session with its own link
and restart rules, tests against a fake station, and one real station to run
it on. From DCC-EX's 625 protocol lines and Rocrail's 1,700 to 3,500 lines
of C for Z21, Märklin CAN, ECoS and XpressNet, a rough estimate is 1,000 to
2,500 lines of Python per protocol plus tests. That is an estimate, not a
measurement.

What it reaches, if the five are done: most stations in section 1. Each
protocol's own features are reachable. For Z21: `LAN_X_SET_STOP`, which
stops the locomotives with the track voltage on (§2.13, p. 15), and its
broadcast when a handset triggers it (§2.14); a system state with main and
programming current, track voltage, and separate internal and external short
circuit flags (§2.18, p. 18); and broadcasts of changes other clients make
(§2.16).

What it loses: breadth. Each protocol is a purchase and a maintenance load.
LocoNet adds a legal question. Digitrax's Personal Edition "may not be used
for products that are intended for distribution to others, with or without
profit", and commercial use needs a licence
(<https://www.digitrax.com/support/loconet/>). Whether that binds free
software such as JMRI or rails49 I could not find out. None of the protocols
read here has a host timeout that cuts track power: Z21 drops a client that
has been silent for a minute from its list of participants and says nothing
about power (§1.1, p. 8). The rails49 `<G n>` stays a DCC-EX fork feature.

### (c) One common LAN protocol

Z21 LAN is the one protocol several brands now speak on the network: Roco and
Fleischmann Z21s, YaMoRC YD7010 and ZIMO MX10, with the DR5000 reported. Its
spec is public, versioned and current (1.13, 2023), and ZIMO publishes a
server-side implementation (`zimo-elektronik/z21` on the
[ESP Component Registry](https://components.espressif.com/components/zimo-elektronik/z21/versions/0.4.2/readme)).
One native driver would reach those, with the Z21 features listed under (b).

It does not reach Digitrax (LNWI speaks WiThrottle), NCE, ESU ECoS, Märklin
or DCC-EX. The rails49 DCC-EX fork has no Z21 code; a search for `Z21` finds
nothing. Not checked: how far YaMoRC's and ZIMO's Z21 servers follow the spec
beyond what the Roco app uses.

WiThrottle is the other protocol many stations serve: JMRI, DCC-EX, Digitrax
LNWI and YaMoRC YD7010. It carries throttles, turnouts, routes, track power
and the fast clock (`jmrit/withrottle/DeviceServer.java:208-365`), and no
sensors and no signals. Its heartbeat sends an emergency stop to the
device's throttles when it lapses (`DeviceServer.java:73-76`, `:483-500`),
which is a per-client stop, not a power cut. It is a throttle protocol, too
narrow for control.

### Recommendation

Build (a) as planned, and add native drivers only for a station that is on a
real layout and needs a capability JMRI does not carry. When that happens,
start with Z21 LAN.

- (a) is already specified in #1 to #7, costs one driver, and covers every
  station JMRI covers, at the level JMRI's contributors brought it to. No
  other route gets that breadth for that cost.
- Its gaps are known and written down in #2. They matter most for safety:
  the host timeout and the emergency stop with the rails live. Those are
  what DCC-EX native gives today, and DCC-EX stays the reference station.
- A native driver pays off only when someone runs that station. JMRI shows
  that support without the hardware stays partial: simulators that say most
  commands are unsupported, help pages that say "untested". rails49 would be
  in the same position with any protocol it writes without a station.
- Z21 LAN is the best second protocol because one public spec reaches
  several current brands on the network, and it carries what JMRI's JSON
  drops: emergency stop with the rails live, short circuits, current. LocoNet
  reaches more stations. The connection is USB, as for DCC-EX: the DCS52,
  DCS210+, DCS240 and DCS240+ have a USB port that acts as a LocoNet
  interface (`help/en/html/hardware/loconet/DCS52.shtml:94-96`;
  `jmrix/loconet/LnCommandStationType.java:410-458`), and older stations
  take a separate adapter such as a PR3, PR4 or LocoBuffer-USB. LocoNet's
  costs are the licence question and its size: 61k lines in JMRI against 9k
  for Z21.
  Märklin CAN and ECoS each reach one
  brand plus the DR5000.

## What I could not verify

- iTrain's internals, its test practice, and which of its 34 stations its
  author owns. The systems page says nothing about this.
- Rocrail's current code and its statements on testing. The wiki returned
  403 and the source is no longer public. The line counts are from a 2015
  mirror and may differ from Rocrail today.
- JMRI's hardware testing outside the repository: mailing lists, maintainers'
  own layouts.
- The Lenz XpressNet, ESU ECoS and NCE specifications. I did not find
  official copies to read. The rows for those protocols rest on JMRI's code
  and help pages. NCE's simulator quotes the Power Pro System Reference
  Manual, which I did not read.
- Märklin's CAN specification (`cs2CAN-Protokoll-2_0.pdf`, linked from
  Märklin forums), and whether its "system halt" keeps the rails live.
- The DR5000's Z21 support (one review), and whether LocoNet Personal
  Edition's terms bind free software.
