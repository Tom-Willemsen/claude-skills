---
name: ibex
description: IBEX control system technical overview. Use to route IBEX-specific terminology or concepts to the correct IBEX technical component, or to understand the high-level architectural layout of IBEX.
---

# About IBEX

IBEX is the EPICS-based control system used on instruments at the ISIS Neutron and Muon source.

The ISIS source is a pulsed neutron and muon source. Scientific instruments use these neutrons or muons to perform scientific experiments.

ISIS instruments have names like ALF, HIFI, SANS2D. Instruments are usually independent from one another; however, they all depend on the accelerator. IBEX may read information from the accelerator control system, for example beam currents or moderator temperatures, but does not control the accelerator.

Important IBEX-specific terminology and acronyms:
- Instrument: a neutron or muon beamline including all devices required to operate it
- Device: an individual piece of scientific hardware that IBEX can communicate with, e.g. temperature controller, motor controller
- EPICS: the control system framework on which IBEX is built
- PV: process variable - a single data point e.g. the heater power on a single channel of a temperature controller
- IOC: EPICS software component which communicates to hardware and serves all relevant PVs from that hardware
- Macros: Settings passed into an IOC. Includes comms settings (e.g. serial port, IP address) and IOC-specific configuration parameters. Edited by changing the IBEX configuration using the GUI.
- Block: An alias to a PV with a user-friendly name: e.g. the block `Cryostat_temp` may point at the PV `IN:INSTRUMENTNAME:EUROTHRM_01:A01:TEMP`. Defining a block allows:
  - Easy use with scripting, e.g. `g.cset`
  - Use with runcontrol (suspends DAE data collection if the readback is outside a defined range)
  - Display on the web dashboard
  - A log of the value to be inserted into the final data file
  - Archived values to be viewed via the "log plotter" tool
- Configuration: Settings for which IOCs are currently active (including their macros), which blocks are currently defined (including their runcontrol settings)
- Component: Configuration fragment, containing IOCs & blocks like configurations. Often used to represent several devices in an experimental setup
- OPI: operator interface screen / user interface for a specific device. Configured in either `Device screens` or `Synoptic` in IBEX. Independent from IOCs, but most IOCs have at least one OPI for that device
- Device screens: flat list of OPIs available in IBEX GUI
- Synoptic: GUI representation of all of the devices in the beamline. Configured via GUI synoptic editor, displays OPIs
- NDX: The computer on which the IBEX control system runs. e.g. NDXMERLIN is the computer for the MERLIN instrument
- DAE: data acquisition electronics - hardware which reads the physical detectors
- Journal: a metadata record from historic data collections

# Important IBEX components

## IBEX Client

The IBEX Client is a single application, which functions as a GUI for IBEX. It is implemented in Java (Eclipse RCP), using components from control system studio.

Briefly, it contains screens and menus for:
- Modifying configurations and synoptics
- Configuring run-control
- Starting and stopping IOCs
- Viewing alarms (PVs outside a defined range or not communicating)
- Viewing the ISIS Beam status
- Manually adjusting DAE configuration parameters
- Displaying operator-interface screens for individual devices
- Adjusting experiment details
- Viewing IOC Log messages
- Viewing metadata about recently-completed runs
- Displaying historic traces of block values
- An interactive python scripting console (including matplotlib plotting)
- Synoptic
- Links to documentation and bug-reporting forms

## Python

Python is the scripting language used by IBEX. Two main libraries are available: `genie_python` and `ibex_bluesky_core`.

### `genie_python`

General purpose instrument control library. Many settings that can be changed in the IBEX GUI can also be changed non-interactively via genie_python.

Example `genie_python` commands:

```python
from genie_python import genie as g
g.cset("block_name", 123.456)
g.cget("block_name")["value"]
g.begin()
g.end()
```

### `ibex_bluesky_core` / bluesky

Specialised library for controlling scanning and alignment processes.

A simple bluesky plan looks like:

```python
from ophyd_async.plan_stubs import ensure_connected
import bluesky.plans as bp
from ibex_bluesky_core.devices.block import block_r, block_mot

def my_plan(det_block_name: str, mot_block_name: str, start: float, stop: float, num: int):
    motor = block_mot(mot_block_name)
    detector = block_r(float, det_block_name)

    yield from ensure_connected(detector, motor)
    yield from bp.scan([detector], motor, start, stop, num)
```

Bluesky is only used on a small number of instruments; reflectometers, SANS and Muon instruments are among the primary users. However, those instruments still primarily
use `genie_python` for generic control scripts - bluesky is used for their scanning or alignment procedures.


## IBEX Server

The IBEX server is a collection of cooperating processes, which together control the individual devices which comprise a neutron or muon instrument. The main components of the IBEX server are:

### Configuration server

The configuration server (historically known as the blockserver) describes the current setup of an instrument:
- IOCs:
  * Which IOCs are currently running
  * Macros
  * "PV Values", which write to an arbitrary PV immediately after IOC start
- Blocks:
  * PV name, logging rates, visibility, default runcontrol settings
- Groups
- Synoptics

The settings managed by the configuration server are adjusted by the configuration and components screens in the IBEX GUI.

### ICP

The ICP (Instrument Control Program) is the software process which controls the DAE (Data Acquisition Electronics) hardware. The ICP tells the DAE when to start and stop collecting data from the instrument detectors. The ICP is also responsible for coordinating acquisition and writing NeXus files with detector data and associated metadata/logs

The configuration settings used by the ICP and DAE can be modified in the DAE perspective in the IBEX GUI, or by python scripting commands.

### IOCs

IOCs (Input/Output Controllers) are processes which control individual pieces of scientific hardware. IOCs are implemented using EPICS and translate a device-specific protocol to standard EPICS PVs. IOCs may also implement control logic for the hardware.

## Web dashboard

IBEX provides a simplified read-only view of the instrument on a public web dashboard.
