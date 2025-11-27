# nsmd - Nvidia System Management Daemon

Authors:  

- Gilbert Chen (<gilbertc@nvidia.com>)
- Harshit Aghera (<haghera@nvidia.com>)
- Rajat Jain (<rajatj@nvidia.com>)
- Aishwary Joshi (<aishwaryj@nvidia.com>)
- Utkarsh Yadav (<uyadav@nvidia.com>)
- Pawel Iwaneczko (<piwaneczko@nvidia.com>)

Primary assignee:

- Harshit Aghera (<haghera@nvidia.com>)

Reviewers:  

- Deepak Kodihalli (<dkodihalli@nvidia.com>)
- Shakeeb Pasha (<spasha@nvidia.com>)
- Gilbert Chen (<gilbertc@nvidia.com>)

## Capabilities

The nsmd service can discover NSM endpoint, gather telemetry data from the endpoints, and can publish them to D-Bus or similar IPC services, for consumer services like bmcweb.

## Relevant Standard Specifications

1. NVIDIA System Management API Specification
2. [DMTF MCTP Base Specification](https://www.dmtf.org/dsp/DSP0236)

## Architecture

### nsmd flow chart

```text
   ┌──────┐                       ┌───────────────┐              ┌───────────┐           ┌───────────┐     ┌────────────┐
   │ nsmd │                       │ EntityManager │              │ FruDevice │           │ mctp-ctrl │     │ NSM Device │
   └──┬───┘                       └───────┬───────┘              └─────┬─────┘           │  daemon   │     └────────────┘
      │                                   │    ┌────┐                  │                 └────┬──────┘           │       
      │                                   │    │JSON│                  │                      │                  │       
      │                                   │    └─┬──┘                  │                      │                  │       
      │                                   │ load │                    ┌┤                      │                  │       
      │                                   │◄─────┘      fruDevice PDI ││ detect EEPROM        │                  │       
      │                                   │     interfaceAdded signal ││ expose to D-Bus      │                  │       
      │                                  ┌┤◄──────────────────────────┴┤                      │                  │       
      │  config PDI InterfaceAdded signal││ Probe success              │                      │                  │       
      │             e.g. NSM_Temp        ││                            │                      │                  │       
     ┌┤  ◄───────────────────────────────││                            │                      │                  │       
     ││                                  ││                            │                      │                  │       
     ││1.get UUID for devType and inst#  ││                            │                      │                  │       
     ││2.search nsmDevice(devType, inst#)││                            │                      │                  │       
     ││3.create nsmDevice if not found   ││                            │                      │                  │       
     ││4.create sensor add to nsmDevice  ││                            │                      │                  │       
     ││5.start sensor pollign task for   ││                            │                      │                  │       
     └┤  the nsmDevice if not start yet  ││                            │                      │                  │       
      │                                  ││                            │                      │                  │       
      │             InterfaceAdded signal││                            │                      │                  │       
      │             NSM_Tble_RemapInst   ││                            │                      │                  │       
     ┌┤  ◄───────────────────────────────││                            │                      │                  │       
     ││1.get name for devType            ││                            │                      │                  │       
     ││2.get type for remapping type     ││                            │                      │                  │       
     ││3.add table to deviceManager      ││                            │                      │                  │       
     └┤                                  ││                            │                      │                  │       
      │                                  ││                            │                      │                  │       
      │             interfaceAddes singal││                            │                      │                  │       
      │             NSM_XXX              ││                            │                      │                  │       
     ┌┤  ◄───────────────────────────────┴┤                            │                      │                  │       
     ││  ...                              │                            │                      │                  │       
     └┤                                   │                            │                      │                  │       

   ┌──────┐                                                                              ┌───────────┐     ┌────────────┐
   │ nsmd │                                                                              │ mctp-ctrl │     │ NSM Device │
   └──┬───┘                                                                              │  daemon   │     └─────┬──────┘
      │                                                                                  └────┬──────┘           │       
      │                                                                                      ┌┤ EID enumerated   │       
      │                                                                                      ││ or start to support NSM  
      │                                         interfacesAdded or PropertiesChanged signal  ││                  │       
      │                                      xyz.openbmc_project.MCTP.Endpoint,Enabled=true  ││                  │       
     ┌┤  ◄───────────────────────────────────────────────────────────────────────────────────┴┤                  │       
     ││1.check if EID support message type 0x7E                                               │                  │       
     ││2.send queryDeviceIdentification                                                       │                  │       
     ││  ─────────────────────────────────────────────────────────────────────────────────────┼────────────────► │       
     ││                                                                           response devType=X Instance#=Y │       
     ││  ◄────────────────────────────────────────────────────────────────────────────────────┼───────────────── │       
     ││3.add EID to discoveredEIDs table                                                      │                  │       
     ││4.record devType,Inst# to table                                                        │                  │       
     └┤5.mark EID is online                                                                   │                  │       

   ┌──────┐                                                                              ┌───────────┐     ┌────────────┐
   │ nsmd │                                                                              │ mctp-ctrl │     │ NSM Device │
   └──┬───┘                                                                              │  daemon   │     └─────┬──────┘
      │                                                                                  └────┬──────┘           │       
      │                                                                                      ┌┤ detect EID       │       
      │                                                            propertiesChanged signal  ││ offline          │       
      │                                     xyz.openbmc_project.MCTP.Endpoint,Enabled=false  ││                  │       
     ┌┤ ◄────────────────────────────────────────────────────────────────────────────────────┴┤                  │       
     ││ search nsmDevice by EID and set isActive to false                                     │                  │       
     ││ update DiscoveredEIDs table to set EID offline                                        │                  │       
     ││                                                                                       │                  │       
     └┤                                                                                       │                  │       

   ┌──────┐                                                                              ┌───────────┐     ┌────────────┐
   │ nsmd │                                                                              │ mctp-ctrl │     │ NSM Device │
   └──┬───┘                                                                              │  daemon   │     └─────┬──────┘
      │                                                                                  └────┬──────┘           │       
┌──► ┌┤ sensor doPollingTask start                                                            │                  │       
│    ││                                                                                       │                  │       
│    ││ if(nsmDevice.isActive==false) {                                                       │                  │       
│    ││   remap the inst# if remapTable for the devType is available                          │                  │       
│    ││   search EID for devType=X,Inst#=Y                                                    │                  │       
│    ││   set isActive=true if EID is online                                                  │                  │       
│    ││ }                                                                                     │                  │       
│    └┤                                                                                       │                  │       
│    ┌┤                                                                                       │                  │       
│    ││ if(nsmDevice.isActive==false) {                                                       │                  │       
│    ││   sleep and continue sensor polling loop                                              │                  │       
│    ││ }                                                                                     │                  │       
│    └┤                                                                                       │                  │       
│    ┌┤                                                                                       │                  │       
│    ││ if(nsmDevice.isActive==true) {                                                        │                  │       
│    ││   foreach sensor in nsmDevice              send NSM command to get reading            │                  │       
│    ││   call sensor.update()       ─────────────────────────────────────────────────────────┼────────────────► │       
│    ││   handle response                          response                                   │                  │       
│    ││   update PDI                 ◄────────────────────────────────────────────────────────┼────────────────  │       
│    ││ }                                                                                     │                  │       
│    └┤ sleep                                                                                 │                  │       
└─────┤                                                                                                                  
```

### nsmd instanceNumber remapping

```text
┌──────────────┐                        ┌───────────────┐                                                                                   
│SensorManager │                        │ DeviceManager │                                                                                   
├──────────────┴───┐                    ├───────────────┴────────────┐                                                                      
│NsmDevice[]   ────┼───┐                │ DiscoveredEIDs[]   ────────┼───┐                                                                  
│...               │   │                │ mapInstToInst[DevType] ────┼───┼────────────────────────────────┐                                 
├──────────────────┤   │                │ mapUuidToInst[DevType]  ───┼───┼─────────────────────────────┐  │                                 
│doPollingTask()   │   │                │ mapEidToInst[DevType]      │   │                             │  │                                 
│                  │   │                │ ...                        │   │                             │  │                                 
└──────────────────┘   │                ├────────────────────────────┤   │                             │  │                                 
                       │                │ SearchEIDs()               │   │                             │  │          ┌──────────────────┐   
                       │                │ ...                        │   │                             │  └────────► │mapInstToInst[GPU]│   
                       │                └────────────────────────────┘   │                             │             ├──────────────────┴─┐ 
                       │  ┌─────────────┐                                │  ┌────────────────┐         └──────────┐  │ 4 -> 0             │ 
                       └─►│ NsmDevice[] │                                └─►│DiscoveredEIDs[]│                    │  │ 5 -> 1             │ 
                          ├──────┬──────┴─────────┐                         ├────┬───────────┴──────────────────┐ │  │ 6 -> 2             │ 
                          │index#│ DevType:Inst#  │                         │EID │ DevType, Inst#, Uuid, active │ │  │ 7 -> 3             │ 
                          ├──────┴────────────────┤                         ├────┴──────────────────────────────┤ │  │ 0 -> 4             │ 
                          │[0]     FPGA     0     │                         │ 12    FPGA     0     abc1   true  │ │  │ 1 -> 5             │ 
                          │[1]     ERoT     0     │                         │ 13    ERoT     0->0  abc2   true  │ │  │ 2 -> 6             │ 
                          │[2]     ERoT     1     │                         │ 14    ERoT     0->1  abc3   true  │ │  │ 3 -> 7             │ 
                          │[3]     ERoT     2     │                         │ 15    ERoT     0->2  abc4   true  │ │  └────────────────────┘ 
                          │[4]     SWITCH   0     │                         │ 22    SWITCH   0     abc5   true  │ │  ┌───────────────────┐  
                          │[5]     BRIDGE 255     │   matched after         │ 23    BRIDGE 255     abc6   true  │ └─►│mapUuidToInst[ERoT]│  
                          │[6]     GPU      0     │ ◄────────────────────►  │ 28    GPU      4->0  abc7   true  │    ├───────────────────┴─┐
                          │[7]     GPU      1     │   inst# was remapped    │ 29    GPU      5->1  abc8   true  │    │ abc2 -> 0           │
                          └───────────────────────┘   via mapXXXtoInst      └───────────────────────────────────┘    │ abc3 -> 1           │
                                                                                           //remap inst#             │ abc4 -> 2           │
                                                                                           //if table is available   └─────────────────────┘
```

### End to End data path of OpenBMC service block diagram

```text
                    ┌──────────────────┐
                    │    Redfish       │
                    └──────┬───────────┘
    Async DBus Calls       │   ▲
                           ▼   │
                    ┌──────────┴───────┐
                    │      D-Bus       │
                    └──────┬───────────┘
       D-Bus req/Res       │  ▲
                           ▼  │
         ┌────────────────────┴───────────────────┐
         │  NSMD                                  │
         │ ┌──────────┐ ┌──────────┐┌──────────┐  │
         │ │coroutine1│ │coroutine2││coroutine3│  │
         │ └──────────┘ └──────────┘└──────────┘  │
         │                                        │
         └──────┬───────────┬────────────┬────────┘
  Unix Socket   │  ▲        │  ▲         │  ▲
                ▼  │        ▼  │         ▼  │
              ┌────┴───────────┴────────────┴──┐
              │ MCTP demux daemon              │
              └─┬───────────┬────────────┬─────┘
MCTP over PCIe  │  ▲        │  ▲         │  ▲
                ▼  │        ▼  │         ▼  │
              ┌────┴───────────┴────────────┴──┐
              │       FPGA                     │
              └─┬────────────┬───────────┬─────┘
                │  ▲         │  ▲        │  ▲
 MCTP over I2C  ▼  │         ▼  │        ▼  │
              ┌────┴──┐    ┌────┴─┐    ┌────┴──┐
              │  CX7  │    │ QM3  │    │ GB100 │
              └───────┘    └──────┘    └───────┘
```

### Event Loop Responsibilities in NSMD Service

The NSMD event loop (based on `sdeventplus::Event`) handles the following specific tasks:

1. **MCTP Socket Communication:**
   - Handles incoming MCTP messages through socket file descriptors
   - Manages socket connections and disconnections
   - Processes VDM (0x7e) message type registrations

2. **DBus Interface Management:**
   - Monitors for new interface additions
   - Handles interface registration and object creation
   - Manages DBus object paths and service names

3. **Coroutine Management:**
   - Resumes suspended coroutines
   - Manages coroutine semaphores
   - Handles coroutine scheduling and prioritization
   - Manages timer-based coroutine operations
   - Controls sleep/wake cycles for coroutines
   - Handles request retry timeouts

4. **DBus Property Operations:**
   - Handles DBus property operations (get/set)
   - Manages property notifications and updates
   - Processes DBus method calls and responses
   - Handles DBus errors and timeouts

5. **Event Type Handlers:**
   - Processes Event Type 0 handlers
   - Processes Event Type 1 handlers
   - Processes Event Type 3 handlers
   - Manages long-running event responses
   - Handles event acknowledgments

6. **Resource Management:**
   - Manages socket file descriptors
   - Handles instance ID expiration
   - Controls request queue management

---

```mermaid
graph TD
    A[Start Event Loop] --> B{Event Occurred?}
    B -->|No| C[Wait or Sleep]
    B -->|Yes| D[Identify Event Type]
    D --> D1{Event Type}
    D1 -->|MCTP Socket IO| E[Handle MCTP Socket Messages]
    D1 -->|DBus Interface Added| F[Handle New Interface Registration]
    D1 -->|Coroutine Resume| G[Handle Coroutine Operations]
    D1 -->|DBus Property Operation| H[Handle DBus Property Operations]
    D1 -->|Event Type Handlers| I[Process Event Type 0/1/3]
    E --> J[Continue Event Loop]
    F --> J
    G --> J
    H --> J
    I --> J
    J --> B
```

### Sensor polling loop

#### Main Flow
The main polling loop starts with the StartPolling event and proceeds through the following steps:
1. Update NSMDevices event initialization
2. Main polling loop execution
3. Update NSM Devices and EIDs for all devices
4. Update all priority sensors for all NsmDevices
5. Time check to determine if state machine should continue
6. State machine execution for non-priority sensors
7. Check devices readiness after state machine completion
8. Sleep for 20ms before next iteration

#### Update NSM Devices and EIDs
This process handles device initialization and updates:
1. List all currently available devices
2. For each device:
   - Check if device is active
   - If not active, search for EID
   - Update device if EID is found
   - Refresh command matrix if device is active
   - Move to next device
3. Continue until all devices are processed

#### State Machine
The state machine handles non-priority sensor updates:
1. Check if circular queue has sensors
2. If sensors exist:
   - Check if current sensor needs update
   - Update sensor if needed
   - Move circular iterator to next sensor
3. Continue until time limit is reached

The polling state machine prioritizes sensor types based on update frequency and processing cost.

#### Check Devices Readiness
After all sensors are updated:
1. Check device readiness state
2. Update device state under either of the following conditions:
  - Device is currently not ready (`!isDeviceReady`)
  - Device qualifies for a readiness re-check (`isReadyForReadinessCheck`)
3. Call checkAllDevicesReady when appropriate

### Main Flow Diagram
```mermaid
flowchart TD
    %% Start Section
    Start[StartPolling Event] --> StartDeviceTasks[Start Device Task for new NsmDevice if not running]
    StartDeviceTasks --> DeviceLoop[Device Loop]
    
    %% Device Task Flow
    DeviceLoop --> CheckPollingRunning{Polling Running?}
    CheckPollingRunning -->|No| End[End Task]
    CheckPollingRunning -->|Yes| GetTime[Get Current Time t0]
    
    %% Device Activation Check
    GetTime --> CheckDeviceActive{Device Active?}
    CheckDeviceActive -->|No| TryActivate[Try Activate Device]
    TryActivate --> CheckDeviceActive2{Device Active?}
    CheckDeviceActive -->|Yes| CheckCommands{All Command Codes Retrieved?}
    
    %% Command Matrix Check
    CheckCommands -->|No| RefreshMatrix[Refresh Command Matrix]
    CheckCommands -->|Yes| CheckDeviceActive2
    RefreshMatrix --> CheckDeviceActive2
    
    CheckDeviceActive2 -->|No| Sleep[Sleep for remaining time]
    CheckDeviceActive2 -->|Yes| PriorityPolling[Poll Priority Sensors]

    %% Priority and Non-Priority Polling
    PriorityPolling --> NonPriorityPolling[Poll Non-Priority Sensors]
    NonPriorityPolling --> Sleep
    Sleep --> DeviceLoop
```

#### Device Activation Flow
```mermaid
flowchart TD
    %% Try Activate Device
    Start[Try Activate Device] --> SearchEID[Search EID using DeviceManager]
    SearchEID --> EIDFound{EID Found?}
    
    EIDFound -->|Yes| SetOnline[Set Device Online]
    SetOnline --> UpdateDevice[Update NsmDevice in DeviceManager]
    UpdateDevice --> Sleep20ms[Sleep 20ms]
    Sleep20ms --> Return[Return Success]
    
    EIDFound -->|No| SleepInactive[Sleep for Inactive Time]
    SleepInactive --> Return
    
    Return --> End[End Activation]
```

#### Priority Sensors Polling
```mermaid
flowchart TD
    %% Priority Sensors Polling
    Start[Poll Priority Sensors] --> InitQueue[Initialize LimitedSensorQueue with prioritySensors]
    InitQueue --> CheckSensors{Has Sensors to Update?}
    
    CheckSensors -->|Yes| UpdateSensor[Update Current Sensor]
    UpdateSensor --> NextSensor[Move to Next Sensor]
    NextSensor --> CheckSensors
    
    CheckSensors -->|No| End[End Priority Polling]
```

#### Non-Priority Sensors Polling
```mermaid
flowchart TD
    %% Non-Priority Sensors Polling
    Start[Poll Non-Priority Sensors] --> InitQueues[Initialize Queue Dictionary with all sensor types: GPM, LongRunning, Static, RoundRobin]
    InitQueues --> GetTime[Get Current Time t1]
    GetTime --> CalculateRemainingTime[Calculate Remaining Time with t1 - t0]
    CalculateRemainingTime --> CheckTime{Remaining Time < 150ms - allowedBuffer?}
    
    CheckTime -->|No| End[End Non-Priority Polling]
    CheckTime -->|Yes| CheckSensors{Has Sensors to Update in any Queue?}
    
    CheckSensors -->|No| MarkReady[Mark Device Ready]
    MarkReady --> End
    
    CheckSensors -->|Yes| NextState[Next Polling State GPM➜LongRunning➜Static➜RoundRobin]
    NextState --> GetQueue[Get Sensors Queue by Polling State]
    GetQueue --> CheckQueue{Queue has sensors?}
    
    CheckQueue -->|No| NextIteration[Move to Next Iteration]
    CheckQueue -->|Yes| GetSensor[Get Current Sensor]
    GetSensor --> CheckNeedsUpdate{Sensor needs update?}
    
    CheckNeedsUpdate -->|No| NextIteration
    CheckNeedsUpdate -->|Yes| CheckLongRunning{Is LongRunning Type?}
    
    CheckLongRunning -->|Yes| StartLongRunningTask[Start LongRunning Task if not running]
    StartLongRunningTask --> NextIteration
    
    CheckLongRunning -->|No| UpdateSensor[Update Sensor]
    UpdateSensor --> MarkRefreshed[Mark Sensor as Refreshed and set last updated time]
    MarkRefreshed --> CheckStatic{Is Static Type?}
    
    CheckStatic -->|Yes| RemoveFromQueue[Remove from Static Queue]
    CheckStatic -->|No| NextIteration
    RemoveFromQueue --> NextIteration
    
    NextIteration --> GetTime
```

#### Polling State Machine Flow
```mermaid
flowchart TD
    Start[Start Polling State Machine] --> GPM[GPM Sensors State]
    GPM -->  GPMProcess[Process 1 GPM Sensor]
    GPMProcess --> LongRunning[LongRunning Sensors State]
    LongRunning --> LongRunningProcess[Process 1 LongRunning Sensor]
    LongRunningProcess --> Static[Static Sensors State]
    Static --> StaticProcess[Process 1 Static Sensor]
    StaticProcess --> RoundRobin[RoundRobin Sensors State]
    RoundRobin --> RoundRobinProcess[Process 1 RoundRobin Sensor]
    RoundRobinProcess --> GPM
```

### Sensor Polling Sequence Diagram

The following diagram illustrates the sensor polling flow for 2 devices, each with 2 priority sensors and 1 sensor in each non-priority queue:

```mermaid
sequenceDiagram
    participant SM as SensorManager
    participant DT1 as DeviceTask1
    participant DT2 as DeviceTask2
    participant GPU1 as GPU Device 1
    participant GPU2 as GPU Device 2

    Note over SM: Start polling cycle
    SM->>DT1: Start deviceTask(Device1)
    SM->>DT2: Start deviceTask(Device2)
    
    Note over DT1,DT2: Concurrent device activation
    DT1->>DT1: Check device activation
    DT2->>DT2: Check device activation
    DT1-->>SM: Device 1 Active
    DT2-->>SM: Device 2 Active
    
    Note over DT1,DT2: Concurrent command matrix refresh
    DT1->>DT1: Check allCommandCodesAreRetrieved()
    DT2->>DT2: Check allCommandCodesAreRetrieved()
    DT1->>GPU1: refreshCommandMatrix()
    DT2->>GPU2: refreshCommandMatrix()
    GPU1-->>DT1: Command Matrix Refreshed
    GPU2-->>DT2: Command Matrix Refreshed
    
    Note over DT1,DT2: Concurrent priority sensor polling
    DT1->>GPU1: sensor->update() (Priority Sensor 1)
    DT2->>GPU2: sensor->update() (Priority Sensor 1)
    GPU1-->>DT1: Priority Sensor 1 Updated
    GPU2-->>DT2: Priority Sensor 1 Updated
    
    DT1->>GPU1: sensor->update() (Priority Sensor 2)
    DT2->>GPU2: sensor->update() (Priority Sensor 2)
    GPU1-->>DT1: Priority Sensor 2 Updated
    GPU2-->>DT2: Priority Sensor 2 Updated
    
    Note over DT1,DT2: Concurrent non-priority polling
    Note over DT1: GPM State
    DT1->>GPU1: sensor->update() (GPM Sensor)
    Note over DT2: GPM State
    DT2->>GPU2: sensor->update() (GPM Sensor)
    GPU1-->>DT1: GPM Sensor Updated
    GPU2-->>DT2: GPM Sensor Updated
    
    Note over DT1: LongRunning State
    DT1->>GPU1: sensor->update() (LongRunning Sensor)
    Note over DT2: LongRunning State
    DT2->>GPU2: sensor->update() (LongRunning Sensor)
    Note over DT1,DT2: LongRunning sensors processing in background
    
    Note over DT1: Static State
    DT1->>GPU1: sensor->update() (Static Sensor)
    Note over DT2: Static State
    DT2->>GPU2: sensor->update() (Static Sensor)
    GPU1-->>DT1: Static Sensor Updated, Remove from queue
    GPU2-->>DT2: Static Sensor Updated, Remove from queue
    
    Note over DT1: RoundRobin State
    DT1->>GPU1: sensor->update() (RoundRobin Sensor)
    Note over DT2: RoundRobin State
    DT2->>GPU2: sensor->update() (RoundRobin Sensor)
    GPU1-->>DT1: RoundRobin Sensor Updated
    GPU2-->>DT2: RoundRobin Sensor Updated
    
    Note over DT1,DT2: LongRunning responses processed asynchronously
    GPU1-->>DT1: LongRunning Sensor Updated (delayed response)
    GPU2-->>DT2: LongRunning Sensor Updated (delayed response)
    
    Note over DT1,DT2: Concurrent sleep
    DT1->>DT1: Sleep(pollingTimeInUsec)
    DT2->>DT2: Sleep(pollingTimeInUsec)
    
    Note over SM: Next polling cycle begins
    SM->>DT1: Continue deviceTask(Device1)
    SM->>DT2: Continue deviceTask(Device2)
```

### Sensor Queue Types

The polling system manages sensors in different queues based on their priority and characteristics:

- **Priority Sensors**: Polled every 150ms with highest priority
- **GPM Sensors**: GPU Performance Monitoring sensors polled every 1000ms
- **Long Running Sensors**: Sensors that require extended processing time
- **Static Sensors**: One-time polled sensors removed after a successful update
- **Round Robin Sensors**: Non-priority sensors polled in round-robin fashion

### Polling Flow Summary

1. **Device Activation**: Each device task first checks if the device is active.
2. **Priority Polling**: All priority sensors are updated first for each device.
3. **Non-Priority Polling**: Handled in state machine order:
   - GPM (1s) → Long Running (15s) → Static (once) → Round Robin (30s)
4. **Time Management**: Ensures polling finishes within time budget.
5. **Sleep**: Device waits until next polling cycle.


---
## Design

### **Sensors Insertion to Avoid Duplication**  

To prevent duplicate sensor creation and ensure received data is propagated correctly to multiple PDI paths, use `NsmInterfaceContainer<IntfType>` instead of defining a PDI as a direct class member. This approach helps maintain structured and efficient sensor management.  

#### **When to Use `NsmGroupSensor` vs. `NsmInterfaceContainer`**  

- **Use `NsmInterfaceContainer`** when a sensor's data should be sent and received only once, but the decoded result needs to be propagated to multiple PDI paths. This ensures that a single sensor instance efficiently updates all relevant interfaces without redundant communication.  

- **Use `NsmGroupSensor`** as an additional abstraction layer over `NsmInterfaceContainer` when multiple `NsmSubSensor` instances need to handle the same received data differently. While `NsmInterfaceContainer` ensures that data is shared across PDIs, `NsmGroupSensor` calls `handleResponse` for each `NsmSubSensor`, allowing them to process the data in a specific way. This is useful in cases where differentiation is needed, such as handling GPU instance IDs within a shared response.

#### Class Diagrams

- Core Sensor Structure:

```mermaid
classDiagram
direction TB

    class NsmDevice { 
        + addStaticSensor(sensor: std::shared_ptr[SensorType]&) void
        + addSensor(sensor: std::shared_ptr[SensorType]&, priority: bool, isLongRunning: bool) void
        - addSensorBase(sensor: std::shared_ptr[NsmObject]&, priority: bool, isLongRunning: bool) void
    }

    class NsmSensor { 
        + NsmSensor(name: std::string, type: std::string)
        + NsmSensor(copy: NsmObject)
        + genRequestMsg(eid: eid_t, instanceId: uint8_t) std::optional[Request] virtual = 0
        + handleResponseMsg(responseMsg: const nsm_msg*, responseLen: size_t) uint8_t virtual = 0
        + update(manager: SensorManager&, eid: eid_t) requester::Coroutine virtual
        + equals(other: const NsmSensor&) bool virtual
        + operator==(other: const NsmSensor&) bool
    }

    NsmDevice --> NsmSensor

```

- Interface Management:

```mermaid
classDiagram
direction TB

    class NsmInterfaces {
        + using IntfType_t = IntfType
        + using Interfaces = std::unordered_map[std::path, std::shared_ptr[IntfType]]
        + interfaces: Interfaces
        + pdi() IntfType&
        + moveInterfaces(container: NsmInterfaces&) void
    }

    class NsmInterfacesContainer {
        + NsmInterfaceContainer(provider: NsmInterfaceProvider)
    }

    class NsmInterfaceProvider {
        + NsmInterfaceProvider(name: std::string, type: std::string, objectsPaths: dbus::Interfaces)
        + createInterfaces(objectsPaths: dbus::Interfaces) Interfaces
    }

    NsmInterfaces <|-- NsmInterfaceProvider
    NsmInterfaces <|-- NsmInterfacesContainer
```

- Group and Sub Sensors:

```mermaid
classDiagram
direction TB

    class NsmSubSensor {
        + handleResponse(responseMsg: const nsm_msg*, responseLen: size_t) uint8_t virtual = 0
    }

    class NsmGroupSensor {
        + sensors: std::vector[std::shared_ptr[NsmSubSensor]]
        - handleResponseMsg(responseMsg: const nsm_msg*, responseLen: size_t) uint8_t override final
    }

    NsmSubSensor  <|-- NsmGroupSensor
    NsmSensor  <|-- NsmGroupSensor
```

- Sensor Types and Specializations:

```mermaid
classDiagram
direction TB

    class GroupSensorType { 
        + genRequestMsg(eid: eid_t, instanceId: uint8_t) std::optional[Request] override
        + handleResponse(responseMsg: const nsm_msg*, responseLen: size_t) uint8_t override
    }

    class SensorType { 
        + genRequestMsg(eid: eid_t, instanceId: uint8_t) std::optional[Request] override
        + handleResponseMsg(responseMsg: const nsm_msg*, responseLen: size_t) uint8_t override
    }

    class NsmInventoryProperty
    class NsmPCIeLinkSpeed
    class NsmMemoryCapacityUtil
    class NsmWriteProtectedControl
    class NsmGpuPresenceAndPowerStatus
    class NsmPowerSupplyStatus

    NsmGroupSensor <|-- GroupSensorType 
    NsmInterfacesContainer <|-- GroupSensorType 
    NsmInterfacesContainer <|-- SensorType 
    SensorType <|-- NsmInventoryProperty 
    SensorType <|-- NsmPCIeLinkSpeed 
    SensorType <|-- NsmMemoryCapacityUtil 
    GroupSensorType <|-- NsmWriteProtectedControl 
    GroupSensorType <|-- NsmGpuPresenceAndPowerStatus 
    GroupSensorType <|-- NsmPowerSupplyStatus 
```

#### Flowchart diagram

```mermaid
flowchart TD
    n1(["addSensor"]) --> n2["Find same objects by PDI, final class type, and request comparing"]
    n2 --> n3{"Same sensor exists?"}
    n3 -- No --> n5["addSensorBase"]
    n5 --> n6(["Sensor added or inserted"])
    n4["moveInterfaces"] --> n6
    n3 -- Yes --> n7{"Is NsmGroupSensor"}
    n7 -- No --> n4
    n7 -- Yes --> n8["groupSensors"]
    n8 --> n6
```

## Interaction with other services and relevant D-Bus APIs

nsmd interacts with other services listed below, using D-Bus IPC mechanism. In OpenBMC framework D-Bus Interfaces sometimes are referred as Phosphor D-Bus Interfaces (or PDI for short), and hence both the terms are used interchangeably in this document.

1. MCTP demux and control daemons
2. Entity Manager
3. PLDM daemon

## Platform Enablement

Following sections outline steps to be followed to enable telemetry acquisition from an NSM endpoint using nsmd. In addition these sections can also be referred to understand various nsmd capabilities and its dependencies on other services and involved D-Bus Interfaces.

### Discovering NSM endpoint

nsmd detects creation of D-Bus Interface xyz.openbmc_project. MCTP. Endpoint at object path /xyz/openbmc_project/mctp. To identify whether an MCTP endpoint support NSM, nsmd check if 0x7E (VDM-PCI) is present in Property SupportedMessageTypes of this Interface. Similar exercise can also be carried out manually, to check whether an MCTP endpoint supports NSM or not.

nsmd uses D-Bus IPC service, to publish information for each device endpoints that supports NSM. D-Bus Interface - herein referred as FRU PDI - xyz.openbmc_project. FruDevice will be published at object path /xyz/openbmc_project/FruDevice/{DeviceType}_{InstanceNumber} by nsmd.

#### FRU Device PDI Properties

#### FRU Device PDI Properties

List of Properties of FRU Device PDI created by nsmd. The list is not exhaustive.

| Property                | Type    | Mandatory/Optional | NSM Command used to get Value            | Use                         |
|-------------------------- |---------- |------------------- |----------------------------------------- | --------------------------- |
| BOARD_PART_NUMBER        | string  | Mandatory          | Type 3 Get Inventory Information (0x11)  | For debugability.           |
| DEVICE_TYPE             | byte   | Mandatory          | Type 0 Query Device Identification (0x09)| To determine list of inventories to be published. |
| INSTANCE_NUMBER           | byte    | Mandatory          | Type 0 Query Device Identification (0x09)| To determine list of inventories to be published. |
| SERIAL_NUMBER             | string    | Mandatory          | Type 3 Get Inventory Information (0x11)  | For debugability.           |
| UUID                   | string    | Mandatory          | NA (Populated by MCTP Control Daemon)    | To uniquely identify a device and EID lookup.     |

#### Example 1

FruDevice D-Bus object (for exposition purpose only)

```text
root@e4869:~# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/FruDevice/30
NAME                                TYPE      SIGNATURE RESULT/VALUE         FLAGS
xyz.openbmc_project.FruDevice       interface -         -                    -
.BOARD_PART_NUMBER                  property  s         "MCX750500B-0D00_DK" emits-change
.DEVICE_TYPE                        property  y         2                    emits-change
.EID                                property  y         30                   emits-change
.INSTANCE_NUMBER                    property  y         0                    emits-change
.SERIAL_NUMBER                      property  s         "SN123456789"        emits-change
.UUID                               property  s         "550e8400-e29b-41d4- emits-change
```

### Enabling an NSM endpoint and gathering of telemetry values from it

nsmd looks out for creation of certain D-Bus Interfaces at /xyz/openbmc_project/inventory Object path, to start requesting telemetry value from an NSM endpoint.

These interfaces should also contains individual telemetry specific details like which NSM command to use and its content. These interfaces are herein referred to as Configuration PDIs (Phosphor D-Bus Interfaces).

However, nsmd doesn't create Configuration PDIs itself and rely on Entity Manager for their creation. nsmd has configuration driven design and since Configuration PDIs are static, Entity Manager is available as part of OpenBMC infrastructure to host Configuration PDIs on D-Bus. Entity Manager detects the existence of D-Bus Interface xyz.openbmc_project. FruDevice (could have been published by nsmd as mentioned in previous section), on any of the D-Bus services and publishes Configuration PDI based on its content. Entity Manager uses JSON configuration file to determine list of Configuration PDI and their content to be published, upon detection of certain type of FRU Device. Please refer Entity Manager design documents for in depth understanding of the process of publishing Configuration PDI from FRU device D-Bus Interface.

### Structure for EM configs for NSM with examples to follow

As of now NSM support following devices:

```text
typedef enum {
 NSM_DEV_ID_GPU = 0,
 NSM_DEV_ID_SWITCH = 1,
 NSM_DEV_ID_PCIE_BRIDGE = 2,
 NSM_DEV_ID_BASEBOARD = 3,
 NSM_DEV_ID_UNKNOWN = 0xff,
} NsmDeviceIdentification;
```

- Now based on MCTP discovery, for MCTP endpoints which are NSM endpoints, NSM service will create a fruDevice object for each instance of device found.
- We fire a few nsmd commands for FRU and inventory details for the device. We then expose properties required for EM configuration on the PDI.

e.g.

```text
:# busctl tree xyz.openbmc_project.NSM
`-/xyz
  `-/xyz/openbmc_project
    `-/xyz/openbmc_project/FruDevice
      `-/xyz/openbmc_project/FruDevice/31
# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/FruDevice/31
NAME                                TYPE      SIGNATURE RESULT/VALUE                           FLAGS
org.freedesktop.DBus.Introspectable interface -         -                                      -
.Introspect                         method    -         s                                      -
org.freedesktop.DBus.Peer           interface -         -                                      -
.GetMachineId                       method    -         s                                      -
.Ping                               method    -         -                                      -
org.freedesktop.DBus.Properties     interface -         -                                      -
.Get                                method    ss        v                                      -
.GetAll                             method    s         a{sv}                                  -
.Set                                method    ssv       -                                      -
.PropertiesChanged                  signal    sa{sv}as  -                                      -
xyz.openbmc_project.FruDevice       interface -         -                                      -
.BOARD_PART_NUMBER                  property  s         "MCX750500B-0D00_DK"                   emits-change
.DEVICE_TYPE                        property  y         2                                      emits-change
.INSTANCE_NUMBER                    property  y         0                                      emits-change
.SERIAL_NUMBER                      property  s         "SN123456789"                          emits-change
.UUID                               property  s         "c13e2b99-68e4-45f1-8686-409009062aa8" emits-change
```

- As we can see CX7 was identified, we created object /xyz/openbmc_project/FruDevice/31 and interface “xyz.openbmc_project. FruDevice” , exposing UUID, DEVICE_TYPE, INTANCE_NUMBER etc on fru device interface.

#### PCIE BRIDGE DEVICE EM config

Here is the basic example for the device pcie bridge EM json. It also contains sensors to be assumed by pldm type 2.

```text
{
        "Exposes": [
            {
                "Name": "HGX_Driver_NVLinkManagementNIC_$INSTANCE_NUMBER",
                "Type": "NSM_NVLinkManagementSWInventory",
                "UUID": "$UUID",
                "Manufacturer": "Nvidia",
                "Priority": false
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 1,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Temp_0"]
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_0_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 8,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_0_Temp_0"]
            },
            {
                "Name": "HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_1_Temp_0",
                "Type": "SensorAuxName",
                "SensorId": 9,
                "AuxNames": ["HGX_NVLinkManagementNIC_$INSTANCE_NUMBER_Port_1_Temp_0"]
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 2})",
        "Name": "NVLinkManagementNIC_$INSTANCE_NUMBER",
        "Type": "NetworkAdapters",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/chassis/Baseboard_0",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "Nvidia",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Item.Chassis": {
            "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Component"
        },
        "xyz.openbmc_project.Common.UUID": {
            "UUID": "$UUID"
        },
        "xyz.openbmc_project.Inventory.Item.NetworkInterface": {},
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
        }
     },
```

- As soon as the probe gets true , we expose 1 sensor here, for cx7 software inventory related to the driver version.
- “Why UUID”: This uuid will be passed on from fru interface on device objects in NSM. It is required because UUID will be used to uniquely identify the EID/device we are running the nsmd command for. EID is not unique , may change across restarts, after dropping from mctp network and rediscover  etc.
- “Significance of Priority” -  It reflects that the sensor is dynamic. Need to be updated in polling coroutine. Now it has value true, its put in priority sensor list, if its false it is put in round robin list.

On Entity Manager we have:

```text
`-/xyz/openbmc_project/inventory/system/networkadapters
          `-/xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0
|-/xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0/HGX_Driver_NVLinkManagementNIC_0
```

```
root@umbriel:~# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/networkadapters/NVLinkManagementNIC_0/HGX_Driver_NVLinkManagementNIC_0NAME                                                              TYPE      SIGNATURE RESULT/VALUE                           FLAGS
org.freedesktop.DBus.Introspectable                               interface -         -                                      -
.Introspect                                                       method    -         s                                      -
org.freedesktop.DBus.Peer                                         interface -         -                                      -
.GetMachineId                                                     method    -         s                                      -
.Ping                                                             method    -         -                                      -
org.freedesktop.DBus.Properties                                   interface -         -                                      -
.Get                                                              method    ss        v                                      -
.GetAll                                                           method    s         a{sv}                                  -
.Set                                                              method    ssv       -                                      -
.PropertiesChanged                                                signal    sa{sv}as  -                                      -
xyz.openbmc_project.Configuration.NSM_NVLinkManagementSWInventory interface -         -                                      -
.Manufacturer                                                     property  s         "Nvidia"                               emits-change
.Name                                                             property  s         "HGX_Driver_NVLinkManagementNIC_0"     emits-change
.Priority                                                         property  b         false                                  emits-change
.Type                                                             property  s         "NSM_NVLinkManagementSWInventory"      emits-change
.UUID                                                             property  s         "a007f776-7805-4e00-0000-0000482e4f00" emits-change
```

After NSMD consumes it:
- It creates /xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0 object path which contains sensor information for driver version.
- It is kept in priority round robin polling loop because priority property was false in EM config.

```text
`-/xyz
  `-/xyz/openbmc_project
    |-/xyz/openbmc_project/FruDevice
    | `-/xyz/openbmc_project/FruDevice/31
    `-/xyz/openbmc_project/inventory_software
      `-/xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0
```

```text
root@umbriel:~# busctl introspect xyz.openbmc_project.NSM /xyz/openbmc_project/inventory_software/HGX_Driver_NVLinkManagementNIC_0
NAME                                                  TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                   interface -         -                                        -
.Introspect                                           method    -         s                                        -
org.freedesktop.DBus.Peer                             interface -         -                                        -
.GetMachineId                                         method    -         s                                        -
.Ping                                                 method    -         -                                        -
org.freedesktop.DBus.Properties                       interface -         -                                        -
.Get                                                  method    ss        v                                        -
.GetAll                                               method    s         a{sv}                                    -
.Set                                                  method    ssv       -                                        -
.PropertiesChanged                                    signal    sa{sv}as  -                                        -
xyz.openbmc_project.Association.Definitions           interface -         -                                        -
.Associations                                         property  a(sss)    0                                        emits-change writable
xyz.openbmc_project.Inventory.Decorator.Asset         interface -         -                                        -
.BuildDate                                            property  s         ""                                       emits-change writable
.Manufacturer                                         property  s         "a0f7f076-7805-4200-0000-0000482e4300"   emits-change writable
.Model                                                property  s         ""                                       emits-change writable
.Name                                                 property  s         ""                                       emits-change writable
.PartNumber                                           property  s         ""                                       emits-change writable
.SKU                                                  property  s         ""                                       emits-change writable
.SerialNumber                                         property  s         ""                                       emits-change writable
.SparePartNumber                                      property  s         ""                                       emits-change writable
.SubModel                                             property  s         ""                                       emits-change writable
xyz.openbmc_project.Software.Version                  interface -         -                                        -
.Purpose                                              property  s         "xyz.openbmc_project.Software.Version... emits-change writable
.SoftwareId                                           property  s         ""                                       emits-change writable
.Version                                              property  s         "MockDriverVersion 1.0.0"                emits-change writable
xyz.openbmc_project.State.Decorator.OperationalStatus interface -         -                                        -
.Functional                                           property  b         true                                     emits-change writable
.State                                                property  s         "xyz.openbmc_project.State.Decorator.... emits-change writable
```

This is the general pattern we follow for sensor creation. For nsmd we are assuming everything as a sensor.

```text
|                              STATIC SENSORS                                    |               DYNAMIC SENSORS             |
|--------------------------------------------------------------------------------|-------------------------------------------|
| No nsm command trigger required.   |   NSM command need to be triggered once.  |    Priority           |     Round Robin   |
| E.g. Manufacturer NVIDIA           |                                           |                       |                   |
|                                    |                                           |                       |                   |
|                                    |                                           |                       |                   |
|-----------------------------------------------------------------------------------------------------------------------------
|                      Separate update function()                                |      Separate polling coroutine           |
```

#### GB100 DEVICE EM config

```text
{
        "Exposes": [
            {
                "Name": "GPU_$INSTANCE_NUMBER + 1 Processor",
                "Type": "NSM_Processor",
                "UUID": "$UUID",
                "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                "MIGMode": {
                    "Type": "NSM_MIG",
                    "UUID": "$UUID",
                    "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                    "Priority": false
                },
                "ECCMode": {
                    "Type": "NSM_ECC",
                    "UUID": "$UUID",
                    "InventoryObjPath": "/xyz/openbmc_project/inventory/system/processors/GPU_SXM_$INSTANCE_NUMBER + 1",
                    "Priority": false
                }
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 0})",
        "Name": "GPU_$INSTANCE_NUMBER + 1",
        "Type": "Processor",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/chassis/Baseboard_0",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "Nvidia",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
        }
    }
```

- Here we handle both scenario whether we want to index gpu from 0 or 1 .
- For hgxb it is 1 based.

```text
root@umbriel:/usr/share/entity-manager/configurations# busctl tree xyz.openbmc_project.EntityManager
`- /xyz
`- /xyz/openbmc_project
|- /xyz/openbmc_project/EntityManager
`- /xyz/openbmc_project/inventory
`- /xyz/openbmc_project/inventory/system
|- /xyz/openbmc_project/inventory/system/fabric
| `- /xyz/openbmc_project/inventory/system/fabric/Fabric_0
| `- /xyz/openbmc_project/inventory/system/fabric/Fabric_0/QM3_0
`- /xyz/openbmc_project/inventory/system/processor
`- /xyz/openbmc_project/inventory/system/processor/GPU_1
`- /xyz/openbmc_project/inventory/system/processor/GPU_1/GPU_1_Processor
```

```text
root@umbriel:/usr/share/entity-manager/configurations# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/processor/GPU_1/GPU_1_Processor
NAME                                                              TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                               interface -         -                                        -
.Introspect                                                       method    -         s                                        -
org.freedesktop.DBus.Peer                                         interface -         -                                        -
.GetMachineId                                                     method    -         s                                        -
.Ping                                                             method    -         -                                        -
org.freedesktop.DBus.Properties                                   interface -         -                                        -
.Get                                                              method    ss        v                                        -
.GetAll                                                           method    s         a{sv}                                    -
.Set
                    method    ssv       -                                        -
.PropertiesChanged                                                signal    sa{sv}as  -                                        -
xyz.openbmc_project.Configuration.NSM_Processor                   interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Name                                                             property  s         "GPU_1 Processor"                        emits-change
.Type                                                             property  s         "NSM_Processor"                          emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.ECCMode           interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_ECC"                                emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.EDPpScalingFactor interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_EDPp"                               emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
xyz.openbmc_project.Configuration.NSM_Processor.MIGMode           interface -         -                                        -
.InventoryObjPath                                                 property  s         "/xyz/openbmc_project/inventory/syste... emits-change
.Priority                                                         property  b         false                                    emits-change
.Type                                                             property  s         "NSM_MIG"                                emits-change
.UUID                                                             property  s         "c13e2b99-68e4-45f1-8686-409009062aa8"   emits-change
```

- Now if we compare EM config and the results we can see that main configuration PDI is "xyz.openbmc_project. Configuration. NSM_Processor".
- ECCMode, MIGMode which are created in sub blocks of EM json. here are created as kind of secondary PDI's, e.g. "xyz.openbmc_project. Configuration. NSM_Processor. ECCMode" etc.

#### PCIeRetimer DEVICE EM config

- This is a special scenario. Retimer is not a device which is directly supported by nsmd. We get all its info from FPGA.
- So here we tightly couple retimer with fpga EM json.
- As soon as fpga is up we create all retimers supported.

```
{
        "Exposes": [
            {
                "Name": "HGX_PCIeRetimer_0",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 0,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_0_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_0"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_1",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 1,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_1_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_1"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_2",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 2,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_2_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_2"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_3",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 3,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_3_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_3"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_4",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 4,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_4_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_4"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_5",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 5,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_5_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_5"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_6",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 6,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_6_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_6"
                ]
            },
            {
                "Name": "HGX_PCIeRetimer_7",
                "Type": "NSM_PCIeRetimer",
                "UUID": "$UUID",
                "INSTANCE_NUMBER": 7,
                "Association": [
                    "all_sensors",
                    "chassis",
                    "/xyz/openbmc_project/sensors/temperature/HGX_PCIeRetimer_7_TEMP_0",
                    "parent_chassis",
                    "all_chassis",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
                    "fabrics",
                    "chassis",
                    "/xyz/openbmc_project/inventory/system/fabrics/HGX_PCIeRetimerTopology_7"
                ]
            }
        ],
        "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 3})",
        "Name": "HGX_FPGA_0",
        "Type": "fpga",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
    }
```

- Each retimer has type "NSM_PCIeRetimer".
- the advantage of this is future exposes on all pcieretimer now can we done wirth single json block.

e.g. Look at the probe, it gets true for all retimer devices.

```
{
        "Exposes": [
            {
                "Name": "HGX_FW_PCIeRetimer_$INSTANCE_NUMBER",
                "Type": "NSM_PCIeRetimer_FWInventory",
                "Association": [
                    "inventory",
                    "activation",
                    "/xyz/openbmc_project/inventory/system/chassis/HGX_PCIeRetimer_$INSTANCE_NUMBER",
                    "software_version",
                    "updateable",
                    "/xyz/openbmc_project/software"
                ],
                "UUID": "$UUID",
                "Manufacturer": "Nvidia",
                "INSTANCE_NUMBER": "$INSTANCE_NUMBER"
            }
        ],
        "Probe": "xyz.openbmc_project.Configuration.NSM_PCIeRetimer({})",
        "Name": "PCIeRetimer_$INSTANCE_NUMBER",
        "Type": "chassis",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
    }
```

#### HSC Device

- Device for which no chassis schema is applicable.

```
{
           "Name": "HGX_Chassis_0_HSC_0_Temp_0",
           "Type" : "NSM_Temp",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 192,
           "Priority": true
        },
{
           "Name": "HGX_Chassis_0_HSC_0_Power_0",
           "Type" : "NSM_Power",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 128,
           "AveragingInterval": 0,
           "Priority": true
        },
 {
           "Name": "HGX_Chassis_0_StandbyHSC_0_Power_0",
           "Type" : "NSM_Power",
           "Associations": [
               {
                   "Forward" : "chassis",
                   "Backward" : "all_sensors",
                   "AbsolutePath" : "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0"
               }
           ],
           "UUID": "$UUID",
           "Aggregated": true,
           "SensorId": 138,
           "AveragingInterval": 0,
           "Priority": true
        },
],
      "Probe": "xyz.openbmc_project.FruDevice({'DEVICE_TYPE': 3})",
      "Name": "HGX_FPGA_0",
      "Type": "chassis",
      "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/HGX_Chassis_0",
      "xyz.openbmc_project.Inventory.Decorator.Asset":
      {
        "Manufacturer": "Nvidia",
        "Model": "$BOARD_PRODUCT_NAME",
        "PartNumber": "$BOARD_PART_NUMBER",
        "SerialNumber": "$BOARD_SERIAL_NUMBER"
      },
      "xyz.openbmc_project.Inventory.Item.Chassis": {
         "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Module"
      },
      "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": "$INSTANCE_NUMBER"
      }
   }
```

#### NSM Event Configs

- json blocks are of 2 types
- applicable for all message types for each device.

```
 {
            "Name": "GlobalEventSetting",
            "Type": "NSM_EventSetting",
            "UUID": "$UUID",
            "EventGenerationSetting": 2     #push mode selected
         },
```

- applicable for each message type for each device.

```
{
            "Name": "PlatformEnvironmentalEventSetting",
            "Type": "NSM_EventConfig",
            "MessageType": 3,     #messagetype applicable
            "UUID": "$UUID",
            "SubscribedEventIDs": [ #events to subscribe for
               0,
               1
            ],
            "AcknowledgementEventIds": [   #events for ack we want
               0,
               1
            ]
         },
```

For detaisl can be found in events block of this document.

### List of Configuration PDIs of nsmd

TODO - All Configuration PDIs with their application and type and description of each of its property are to be added in this section.

#### Nvlink Port Configuration in EM Json

1. To create required number of links of type "Name", add below mentioned configuration in EM json.

| Configuration Property  | type    | Description                                                       |
|------------------------ |-------- |----------------------------------------------------------------- |
| Type                    | string  | NSM_NVLink                                                        |
| Name                    | string  | Name for EM dbus object, which is also used as port dbus object name prefix.            |
| ParentObjPath             | string  | Dbus object path of the device on which the port objects will be created.                 |
| DeviceType                | int     | Device type as per defination in NsmDeviceIdentification enum defined above.              |
| UUID                      | string   | UUID of the device.                                               |
| Priority                  | boolean   | Priority to indicate which queue to add the created sensor for polling.                   |
| Count                   | int     | The total port count on the device.<br>example: if Count=4 and Name="NVPort", then four dbus objects will be created.<br>/xyz/openbmc_project/.../Ports/NVPort_0<br>/xyz/openbmc_project/.../Ports/NVPort_1<br>/xyz/openbmc_project/.../Ports/NVPort_2<br>/xyz/openbmc_project/.../Ports/NVPort_3  |

Example json snippet:

```
    {
        "Name": "NVLinkManagement",
        "Type": "NSM_NVLink",
        "ParentObjPath": "/xyz/openbmc_project/inventory/system/chassis/HGX_NVLinkManagementNIC_0/NetworkAdapters/NVLinkManagementNIC_0",
        "DeviceType": "$DEVICE_TYPE",
        "UUID": "$UUID",
        "Priority": true,
        "Count": 2
    }
```

2. To add topology details on the created dbus port objects, there is a python script which generates the EM json from a mapping excel sheet.
More details are added in nsmd repo (nsmd/tools/topology/Readme.md).

3. To have correlation/association with available sensors in PLDM (in case of NVSwitches & NetworkAdapters) and NSM device EM json configuration is to be added for each sensor Id in PLDM providing the auxiliary name and other EM config derived details.
More details are added in nsmd repo (nsmd/tools/correlation/Readme.md).

### Steps to enable nsmd for a specific platform

To enable nsmd for a Platform, please follow steps given below.

1. Include nsmd as distro dependency for the Platform.
2. Create Entity Manager configuration file for the device that supports NSM, and configure Entity Manager to use this file for the Platform. Refer Entity Manager documentation for information on involved steps. An example file in given in earlier sections.
3. Provide Platform specific settings like request timeouts, number of retries etc, by using Meson Options (nsmd uses Meson build system). Use EXTRA_OEMESON variable from meson bbclass to provide non-default values for these settings. Refer meson_options.txt file at nsmd repo for list of all available configuration options.

## Extending Infrastructure for Device Role

The Device Role parameter has been introduced as an optional extension to the NSMd infrastructure to uniquely identify NSM devices. This addition does not impact existing use cases, as the device role parameter is optional and maintains backward compatibility.

### Rationale for Device Role

Device type and device instance alone are not sufficient to uniquely identify an NSM device due to overlapping instance numbers across different device categories. The device role parameter was introduced to resolve this ambiguity.

#### Key Design Constraints

**Device Type Limitations:** All CX-series devices (CX7, CX8, CX9, etc.) are categorized under device type 2 (PCIe Bridge). It is not feasible to introduce separate device types for each CX variant, as they share the same functional category.

**Instance Number Overlaps:** Device types 2 (PCIe Bridge) and 5 (MCTP Bridge) have overlapping instance numbers across different physical devices. These instance numbers drive the Redfish naming conventions for resources, making unique identification critical.

### Device Role Mapping

The following table illustrates how device role provides unique identification for devices with overlapping device types and instance numbers:

| Device        | Device Type       | Device instance | Device role               |
|---------------|-------------------|-----------------|---------------------------|
| GPU           | 0 (GPU)           | --              | Not required (0-RESERVED) |
| NvSwitch (QM3)| 1 (Fabric switch) | --              | Not required (0-RESERVED) |
| CX7           | 2 (PCIe Bridge)   | --              | 1 for CX7                 |
| CX8           | 2 (PCIe Bridge)   | --              | 2 for CX8                 |
| CX9           | 3 (PCIe Bridge)   | --              | 3 for CX9                 |
| SXM MCU       | 5 (MCTP Bridge)   | --              | 1 for SXM MCU             |
| CX MCU        | 5 (MCTP Bridge)   | --              | 2 for CX MCU              |
| HPM MCU       | 5 (MCTP Bridge)   | --              | 3 for HPM MCU             |

### Backward Compatibility

The device role parameter is optional in NSMRawCommand requests. When not specified in a Redfish query, the device role defaults to 0 (RESERVED), ensuring that existing use cases and integrations remain unaffected by this extension.

### UUID Configuration in Entity Manager

When defining NSM devices in Entity Manager configuration files, UUIDs can be specified using a static format that encodes device type, role, instance number, and mapping information. This is particularly useful for defining static devices configs.

#### Static UUID Format

The static UUID format follows this structure:

```text
STATIC:<deviceType&Role>:<instanceNumber>:<MappingTag>:<MappingValue>
```

**Format Components:**

- `STATIC`: Keyword indicating this is a statically generated UUID
- `<deviceType&Role>`: A 16-bit combined value encoding both device type and role
- `<instanceNumber>`: The device instance number
- `<MappingTag>`: Tag identifying the mapping mechanism (e.g., "EID", "UUID", "SLOT")
- `<MappingValue>`: The value associated with the mapping tag

#### Device Type and Role Encoding

The `<deviceType&Role>` field is a 16-bit value that combines the device type and device role into a single parameter:

```c
void getDeviceTypeAndRole(uint16_t combined, uint8_t* deviceType,
                          uint8_t* deviceRole)
{
    *deviceType = (uint8_t)(combined & 0xFF); // Type is in low byte
    *deviceRole = (uint8_t)(combined >> 8);   // Role is in high byte
}
```

**Encoding Rules:**

- **Low byte (bits 0-7)**: Contains the device type value
- **High byte (bits 8-15)**: Contains the device role value

**Examples of Combined Values:**

| Device Type | Device Role | Combined Value (hex) | Combined Value (decimal) |
|------------|-------------|---------------------|-------------------------|
| 2 (PCIe Bridge) | 1 (CX7) | 0x0102 | 258 |
| 2 (PCIe Bridge) | 2 (CX8) | 0x0202 | 514 |
| 5 (MCTP Bridge) | 1 (SXM MCU) | 0x0105 | 261 |
| 5 (MCTP Bridge) | 2 (CX MCU) | 0x0205 | 517 |

#### Example UUID Configuration

In Entity Manager JSON configuration:

```json
{
    "Name": "CX8_Device_0",
    "UUID": "STATIC:514:0:EID:30"
}
```

This configuration specifies:
- Device Type: 2 (0x02) - PCIe Bridge
- Device Role: 2 (0x02) - CX8
- Combined Value: 514 (0x0202)
- Instance Number: 0
- Mapping: EID 30

Another example for SXM MCU:

```json
{
    "Name": "SXM_MCU_1",
    "UUID": "STATIC:261:1:EID:25"
}
```

This configuration specifies:
- Device Type: 5 (0x05) - MCTP Bridge
- Device Role: 1 (0x01) - SXM MCU
- Combined Value: 261 (0x0105)
- Instance Number: 1
- Mapping: EID 25

## Reference

1. <https://www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.1_0.pdf>

2. <https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.0.pdf>

3. <https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.1.0.pdf>
