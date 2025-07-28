# Bluefield Rshim Service

Author: Sean Zhang

Created: July 28, 2025

## Problem Description

The objective of this feature is to enhance the Rshim ownership grant from BMC.

With this feature, users can enable Rshim on BMC by force.

## Background and References
Previous we have OEM action to enable Rshim on BMC, but the host need disable Rshim before the action, otherwise BMC can not grant the Rshim.

This design will introduce a new options to allow BMC grant Rshim ownership even if the Rshim is already enabled by host.

Beside above, we want to move rshim related service (LFWP) to use the new added rshim service.

1. API change

Add a new parameter for Oem/Nvidia as below:
```
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -XPATCH -d '{
      "BmcRShim": {
        "BmcRShimEnabled": true,
        "Force": true,
      }
}' https://$bmc/redfish/v1/Managers/Bluefield_BMC/Oem/Nvidia
```

With the 'Force' set true, the bmc will force grant the rshim ownership even host have the rshim ownership.

The below OEM action can also be used.
```
curl -k -X GET https://bmc-ip/redfish/v1/Managers/Bluefield_BMC/
{
  "@odata.id": "/redfish/v1/Managers/Bluefield_BMC",
  "@odata.type": "#Manager.v1_15_0.Manager",
  "Actions": {
    ...
    "Oem": {
      ...
      "#NvidiaManager.SetRshim": {
        "Rshim": {
          "@Redfish.AllowableValues": [
            "Enabled",
            "Disabled",
            "Forced"
          ]
        },
        "target": "/redfish/v1/Managers/Bluefield_BMC/Actions/Oem/NvidiaManager.SetRshim"
      }
    }
  }
  ...
}
curl -k -X POST -d '{"Rshim": "Forced"}' https://bmc-ip/redfish/v1/Managers/Bluefield_BMC/Actions/Oem/NvidiaManager.SetRshim
{
  "@Message.ExtendedInfo": [
    {
      "@odata.type": "#Message.v1_1_1.Message",
      "Message": "The request completed successfully.",
      "MessageArgs": [],
      "MessageId": "Base.1.18.1.Success",
      "MessageSeverity": "OK",
      "Resolution": "None."
    }
  ]
}
```

No change needed for LFWP API.
```
curl -k -H "X-Auth-Token: $token" -X POST -d '{"LFWP": "Enabled"}' https://${bmc}/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/LFWP.Set
```

2. DBUS change

Add a new dbus interface for enabling/disabling Rshim ownership in BMC defined as below:

```
phosphor-dbus-interfaces$ cat yaml/com/nvidia/BF/Rshim.interface.yaml
description: >
    Implement to manage the Rshim.
properties:
    - name: State
      type: enum[self.State]
      description: >
          The Rshim state.
      default: Disabled
    - name: LfwpEnabled
      type: boolean
      description: >
          The LFWP is enabled or not.
      default: false

enumerations:
    - name: State
      description: >
          Possible Rshim State settings
      values:
          - name: Enabled
            decription: >
                The Rshim is enabled.
          - name: Disabled
            decription: >
                The Rshim is disabled.
          - name: Forced
            decription: >
                The Rshim is forced to be enabled.
```

The property `State` is enumeration for enable/disable/force enable Rshim ownership flag and the `LfwpEnabled` property is flag for LFWP.

A new object `/com/nvidia/software/rshim` will be added as below with the previous defined interface `com.nvidia.BF.Rshim`.

```
busctl tree com.Nvidia.Software.Rshim
`- /com
  `- /com/nvidia
    `- /com/nvidia/software
      `- /com/nvidia/software/rshim
busctl introspect com.Nvidia.Software.Rshim /com/nvidia/software/rshim
NAME                                TYPE      SIGNATURE  RESULT/VALUE                         FLAGS
com.nvidia.BF.Rshim                 interface -          -                                    -
.LfwpEnabled                        property  b          false                                emits-change writable
.State                              property  s          "com.nvidia.BF.Rshim.State.Disabled" emits-change writable
org.freedesktop.DBus.Introspectable interface -          -                                    -
.Introspect                         method    -          s                                    -
org.freedesktop.DBus.ObjectManager  interface -          -                                    -
.GetManagedObjects                  method    -          a{oa{sa{sv}}}                        -
.InterfacesAdded                    signal    oa{sa{sv}} -                                    -
.InterfacesRemoved                  signal    oas        -                                    -
org.freedesktop.DBus.Peer           interface -          -                                    -
.GetMachineId                       method    -          s                                    -
.Ping                               method    -          -                                    -
org.freedesktop.DBus.Properties     interface -          -                                    -
.Get                                method    ss         v                                    -
.GetAll                             method    s          a{sv}                                -
.Set                                method    ssv        -                                    -
.PropertiesChanged                  signal    sa{sv}as   -                                    -
```


3. BMCWEB

Bmcweb can use this dbus service to for rshim enable/disable like below:
```
    if (BmcRShimEnabled && Force)
    {
        state = "com.nvidia.BF.Rshim.State.Forced";
    }
    else if (BmcRShimEnabled)
    {
        state = "com.nvidia.BF.Rshim.State.Enabled";
    }
    else if (!BmcRShimEnabled)
    {
        state = "com.nvidia.BF.Rshim.State.Disabled";
    }
    ...
    sdbusplus::asio::setProperty(
        *crow::connections::systemBus, "com.Nvidia.Software.Rshim",
        "/com/nvidia/software/rshim",
        "com.nvidia.BF.Rshim", "State",
        state,
        [asyncResp](const boost::system::error_code ec) {
        if (ec)
        {
            BMCWEB_LOG_ERROR("DBUS response error Rshim State setProperty {}",
                             ec);
            messages::internalError(asyncResp->res);
            return;
        }
    });
```

For LFWP, change the the related service, object and property is dbus.

From:
```
const PropertyInfo lfwpInfo = {
    .intf = "xyz.openbmc_project.Object.Enable",
    .prop = "Enabled",
    .dbusToRedfish = {{"true", "Enabled"}, {"false", "Disabled"}},
    .redfishToDbus = {{"Enabled", "true"}, {"Disabled", "false"}},
    .isPropBool = true};

DpuActionSetAndGetProp
    lfwp({{"LFWP",
           {.service = "xyz.openbmc_project.Software.DPU.Version",
            .obj = "/xyz/openbmc_project/control/lfwp",
            .propertyInfo = lfwpInfo,
            .required = true}}},
         lfwpTarget);
```

To:
```
const PropertyInfo lfwpInfo = {
    .intf = "com.nvidia.BF.Rshim",
    .prop = "LfwpEnabled",
    .dbusToRedfish = {{"true", "Enabled"}, {"false", "Disabled"}},
    .redfishToDbus = {{"Enabled", "true"}, {"Disabled", "false"}},
    .isPropBool = true};

DpuActionSetAndGetProp
    lfwp({{"LFWP",
           {.service = "com.Nvidia.Software.Rshim",
            .obj = "/com/nvidia/software/rshim",
            .propertyInfo = lfwpInfo,
            .required = true}}},
         lfwpTarget);
```

4. DPU manager

To make it work, a new systemd service `xyz.openbmc_project.Software.Rshim.service` need to be added in BMC.

```
xyz.openbmc_project.Software.Rshim.service:
[Unit]
Description=DPU Rshim
After=multi-user.target
StartLimitIntervalSec=0

[Service]
Restart=always
RestartSec=10
ExecStart=/usr/bin/bf-dpu-rshim

[Install]
WantedBy=multi-user.target
```

The bf-dpu-rshim can be generated in dpu-manager by adding dpu_rshim/dpu_rshim_main.cpp. In the new added file, we can create the needed dbus object, interface and method.

When the operation is set Force for enable rshim, inside the override set function defined in the PDI, it will create the override.conf file and reload systemd before starting the rshim service. And remove the file after rshim started.

```
root@dpu-bmc:~# cat > /etc/systemd/system/rshim.service.d/override.conf
[Service]
Environment="OPTIONS=-F"
```

```
State RshimManager::state(State value)
{
    auto current = sdbusplus::com::nvidia::BF::server::Rshim::state();
    if (value == State::Enabled)
    {
        // TODO: Implement the logic to enable the Rshim
        // systemctl start rshim
        current = sdbusplus::com::nvidia::BF::server::Rshim::state(value);
    }
    else if (value == State::Disabled)
    {
        // TODO: Implement the logic to disable the Rshim
        // systemctl stop rshim
        current = sdbusplus::com::nvidia::BF::server::Rshim::state(value);
    }
    else if (value == State::Forced)
    {
        // TODO: Implement the logic to force the Rshim
        /*
         * 1. Create the override.conf file
         * 2. systemctl daemon-reload
         * 3. systemctl start rshim
         * 4. Remove the override.conf file
         * 5. systemctl deamon-reload
         */
        current = sdbusplus::com::nvidia::BF::server::Rshim::state(State::Enabled);
    }
    return current;
}
```

For the LFWP, we can change the dbus from interface `xyz.openbmc_project.Object.Enable` under `/xyz/openbmc_project/control/lfwp` to use property `LfwpEnabled` under `/com/nvidia/software/rshim`. The legacy dbus object can be removed.

We can remove the interface registration in dpu_version.cpp and add the logic to `setLfwpEnabled` override function in dpu_rshim_main.cpp.

```
    bool setLfwpEnabled(boolean lfwpEnabled) override
    {
        ...
        if (lfwpEnabled == false)
        {
            // Only trigger when switching lfwp to disabled
            if (!lfwpManager->setLfwp(false))
            {
                return false;
            }
        }
        ...
    }
```

For places using lfwpEnable property, we can get the property from the new dbus object.

And all places start Rshim service will be replaced with the new method.

From:
```
    auto method = conn->new_method_call(
        "org.freedesktop.systemd1",
        "/org/freedesktop/systemd1/unit/rshim_2eservice",
        "org.freedesktop.systemd1.Unit", "Start");
    method.append("replace");
    conn->call(method);
```

To:
```
    sdbusplus::asio::setProperty(
        *crow::connections::systemBus, "com.Nvidia.Software.Rshim",
        "/com/nvidia/software/rshim",
        "com.nvidia.BF.Rshim", "State",
        state,
        [asyncResp](const boost::system::error_code ec) {
        if (ec)
        {
            BMCWEB_LOG_ERROR("DBUS response error Rshim State setProperty {}",
                             ec);
            messages::internalError(asyncResp->res);
            return;
        }
    });
```