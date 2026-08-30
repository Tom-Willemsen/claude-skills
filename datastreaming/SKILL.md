---
name: datastreaming
description: General high-level information about the data-streaming project: new components of IBEX which will replace legacy ISISICP infrastructure related to neutron data acquisition with a modern Kafka-based system. Includes processes such as `kafka_dae_control`, `event_udp_to_kafka`, `kafka_event_aggregator`, `kafka_dae_diagnostics`, `filewriter`, `forwarder`, `kafka_forwarder_configurer`.
---

# Datastreaming

The datastreaming project is replacing the ISISICP component of IBEX with a set of loosely-coupled cooperating processes. This is being done in a phased way; the first instruments to use this new stack will be HRPD-X (commissioning September 2027) and SANDALS-2 (commissioning March 2028).

The responsibilities of the ISISICP, and therefore the streaming project, include:
- Low-level communication to data acquisition hardware: setting parameters, and reading back data
- Converting from the electronics representation (raw channels, clock-cycles) to an engineering representation (pixel or spectrum, time-of-flight).
- Writing Nexus files, including the detector data, metadata, and sample-environment parameters from IBEX (EPICS) IOCs. This includes triggering archiving, and writing metadata ("journal") records of all historic runs.

## Kafka

Kafka is the central message broker used by data streaming components to exchange data. Several topics are in use for each instrument; the topic name convention is `<instrument>_topicName`. For example on HRPD-X the topics would include `HRPD-X_runInfo` and `HRPD-X_events`.

Messages are encoded using flatbuffers schemas. Each schema has a 4-character identifier. The authoritative list of schemas is [here](https://github.com/ISISComputingGroup/streaming-data-types/tree/master/schemas). The list of topics and schemas which they may contain is [here](https://raw.githubusercontent.com/ISISComputingGroup/ibex_developers_manual/refs/heads/master/doc/specific_iocs/datastreaming/Datastreaming-Topics.md).

## Components

### [`kafka_dae_control`](https://github.com/isiscomputingGroup/kafka_dae_control)

This is a Python-based 'control' process, which reads and writes registers on the low-level FPGA hardware, via UDP. No 'fast' data travels via this process.

It emits to Kafka:
- `<instrument>_runInfo`: run start (`pl72`) and run stop (`6s4t`) messages, notifying other processes that a run has been started or stopped.
- `<instrument>_vetoConfig`: veto-configuration (`vc00`) messages, notifying other processes about a change in user-selected veto enablement.

It exposes read and write EPICS PVA PVs via a `p4p` server to allow the streaming hardware to be controlled via IBEX.

### [`event_udp_to_kafka`](https://github.com/isiscomputingGroup/event_udp_to_kafka)

This is a Rust-based process which is the first software step on the 'fast' (neutron event) data path. It:
- Listens for UDP messages from the streaming hardware containing neutron events.
- Performs some low-level manipulations on raw hardware packets to extract the data of interest and convert it to a useable representation (for example, converting clock cycles into time-of-flight offsets).
- Reformats the incoming data as a `pu00` message describing the veto flags, protons-per-pulse, and period number for this message, followed by zero or more `ev44` messages containing the event data.
- Sends the flatbuffers to an `<instrument>_rawEvents` Kafka topic.

### [`kafka_event_aggregator`](https://github.com/isiscomputingGroup/kafka_event_aggregator)

This is a Rust-based process which forms full ISIS frames from the data pushed to the `<instrument>_rawEvents` Kafka topic.

After combining events and veto-flags from all data from the same pulse, it re-emits a single `pu00` header for the entire frame followed by `ev44` messages describing the neutron data in the frame to the `<instrument>_events` topic.

It aggregates events assuming that all events in a frame arrive in Kafka within a configurable timeout of each other.

### [`kafka_dae_diagnostics`](https://github.com/isiscomputingGroup/kafka_dae_diagnostics)

This is a Python-based process, with some Rust library functions via PyO3. This process serves 'diagnostics' derived from Kafka topics to IBEX as EPICS PVs. It exposes these PVs using a `p4p` PVA server.

It listens to:
- `<instrument>_runInfo` for notification of run start and stop
- `<instrument>_vetoConfig` for notification about changes in veto configuration (names, enablement)
- `<instrument>_events` for the actual neutron event data

The types of diagnostics it exposes are:
- Counters for how many frames, events, or micro-amp hours have been collected in the current run
- Diagnostic histograms of the data collected so far in the current run. For performance, this is implemented in a Rust library, called from Python.
- Veto diagnostics displaying information about which vetoes are active and blocking data collection
- Run-state readback, which can be for example SETUP (idle), RUNNING, VETOING (attempting to run but data is being vetoed), PAUSED (user paused the acquisition), WAITING (attempting to run, but data collection is suspended due to a parameter out-of-range).
  * `kafka_dae_control` publishes a *hardware* run state readback, but distinguishing between RUNNING and VETOING or WAITING requires the `pu00` headers and therefore subscribing to the `_events` stream.

### `filewriter`

This process is not yet implemented. It will read from a number of Kafka topics, and write out Nexus files for consumption by data-reduction applications such as Mantid or GudRun.

### [`forwarder`](https://github.com/isiscomputingGroup/forwarder)

This process listens to updates of EPICS PVs from IBEX IOCs, and forwards those updates to an `<instrument>_sampleEnv` Kafka topic using the `f144` schema. This is used by the filewriter, so that the filewriter can write time-series logs of interesting EPICS PVs into the Nexus file.

Most EPICS PVs are scalars and update relatively slowly: typically 1Hz, occasionally 10-100Hz. Array PVs are possible but rare.

'Fast' sample environment, needing to log at significant data rates, will go via a direct route to Kafka. This route is not currently implemented.

The set of PVs to be forwarded is modified by `fc00` messages on the `<instrument>_forwarderConfig` topic.

### [`kafka_forwarder_configurer`](https://github.com/isiscomputingGroup/kafka_forwarder_configurer) (also known as `BSKAFKA`)

This process *reconfigures* the `forwarder` process, ensuring that the PVs the forwarder is listening to match the set defined as interesting in the IBEX configuration. It monitors the IBEX configuration via EPICS and reconfigures the forwarder when it changes.

The reconfiguration is done by emitting an `fc00` flatbuffers schema onto the `<instrument>_forwarderConfig` Kafka topic.

## Consumers

### EPICS (PVA)

Several downstream processes may consume PVs served by `kafka_dae_control` or `kafka_dae_diagnostics`:
- IBEX GUI - interactive users, UI readbacks. Implemented using CS-Studio OPIs in a dedicated "Streaming DAE" perspective within the IBEX GUI.
- `genie_python` - scripted Python interface
- `ibex_bluesky_core` - a different Python interface, used by instruments which perform complicated "scans"
- Command-line clients - a rare use-case, only used by developers while debugging
- Monitoring, for example Nagios or DSG's Grafana instance

### Kafka

The Kafka streams contain the full data. The main "end" consumers are:
- The filewriter, to write out those events to persistent Nexus files
- `kafka_dae_diagnostics`, to serve live data back to IBEX as PVs
- Mantid, in its live-listening configuration, to auto-reduce live data

## Boundaries of responsibilities

### Detector Systems Group (DSG)

- Maintain and program the FPGA hardware and firmware
- Define register mappings and UDP event-data communication schemes

### Experiment Controls Group (you are acting on behalf of this team)

- **All software components listed above**
- Everything that conceptually lies between an event-data UDP packet being received by a computer, and a readable Nexus file accessible on long-term storage.
- All interaction with the IBEX control system and EPICS PVs

### Scientific Computing Department

- Data reduction software (including Mantid, GudRun) which may consume Nexus files or the Kafka streams
