# Interface plugin

**Written for: v2.0-vpp18.10**

The VPP interface plugin is a base plugin able to setup various types of **VPP interfaces** and also manages **the interface status** (state data, optionally published back to the database) and **interface and DHCP lease notifications**. The VPP provides multiple interface types, following are supported in the VPP-agent:

  - Sub-interface
  - Software loopback
  - DPDK (physical) interface
  - Memory interface (memif)
  - Tap (version 1 and 2)
  - Af-packet interface
  - VxLAN tunnel
  - IPSec tunnel
  - VmxNet3 interface

Other types are specified as undefined for the vpp-agent purpose.
  
All the interfaces except DPDK (physical) can be created or removed directly in the VPP if all necessary conditions are met (for example an Af-packet interface requires a VEth in the host in order to attach to it ( more in [Interface types](#ifs-model-types) section). The DPDK interfaces (PCI, physical) can be only configured, cannot be added or removed.
  
- [Interfaces](#ifs)
  * [Model](#ifs-model)
    * [Unnumbered interfaces](#ifs-model-unnumbered)
    * [Maximum transmission unit](#ifs-model-mtu)
    * [Interface types](#ifs-model-types)
  * [Configuration](#ifs-config)
- [Interface status](#ifstate)
  * [Model](#ifstate-model)
  * [Configuration](#ifstate-config)
  * [Limitations](#ifstate-limit)
  
## <a name="ifs">Interfaces</a>

The VPP uses multiple commands and/or binary APIs to create and configure shared boundaries. The vpp-agent is simplifying this processes, providing the single model with specific extensions depending on required interface type. The northbound configuration is translated to a sequence of binary API calls using GOVPP library. All interface types except the DPDK (physical) interface can be directly created or removed in the VPP. Physical interfaces can be only configured. The Af-packet interface is dependent on host-interface of the VEth type, and cannot exist without it. 

### <a name="ifs-model">Model</a>

The interface plugin defines following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto). The model is divided to two parts - the first, which contains fields common for all interface types, and the second, exclusive for given interface type. Note that there are even exceptions in the common part, where not all fields can be defined on every interface type. For example, a physical address cannot be set to Af-packet or to DPDK interface despite the field exists in the definition. If this is done, the value is silently ignored.

Mandatory fields are **interface type** and **logical name**. The name is limited to 63 characters. The interface may or may not define unique fields for the given type.

In the generated proto model, the interface is referred to the `Interface` object.

All interfaces use the same key for every type. It means the name has to be incomparable among all interfaces. The key format:

```
vpp/config/v2/config/vpp/v2/interface/<name>
```
 
The interface logical name is for the vpp-agent use only. It serves as a reference for other models (bridge domains for instance). The VPP works with indexes, which are generated when the new interface instance is created. The index is a unique integer for identification and future references. In order to parse name and index, the vpp-agent stores a tag to the VPP via binary API - it's a name-to-index entry which helps the vpp-agent to classify interface name and assign the correct index to it. 

#### <a name="ifs-model-unnumbered">Unnumbered interfaces</a>

Every interface can have assigned one or more IPv4 or IPv6 addresses or their combination. Besides this, the VPP interface can be set as unnumbered; it takes the IP address of another interface and performs as it has the same IP address as the target instance. 

To set an interface as unnumbered, omit `ip_addresses` and set `interface_with_ip`. The target interface has to exist (if not, the unnumbered interface will be postponed and configured later when the required interface appears) and must contain at least one IP address. 

#### <a name="ifs-model-mtu">Maximum transmission unit</a>

The MTU is the size of the largest protocol data unit that can be communicated in a single network layer transaction. The VPP interfaces allow setting custom MTU size. For this purpose, there is a field called `mtu` available in the interface model. If the field is left empty, the VPP uses a default value of the MTU.

The vpp-agent offers an option to automatically set the MTU of specific size to every created interface via *vpp-ifplugin* config file (example [here](https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf)). If the global MTU value is set to non-zero value, but interface definition contains its own different value, the local value is preferred before the global one.

**Note:** MTU is not available for VxLan tunnels and IPSec tunnel interfaces.

#### <a name="ifs-model-types">Interface types</a>

Type-specific interface fields in vpp-agent are defined in special structures called links. Type of the interface's link must match the type of the interface itself. Description of main differences of interface types:

**Sub-interface**

The Sub-interface is an interface type derived from other interfaces adding VLAN ID to it. The sub-interface link of type `SubInterface` requires a name of the parent (super) interface, and ID of the VLAN.

**Software loopback**

The loopback interface type does not have any special fields.

**DPDK interface**

The physical interface cannot be created or removed directly by the VPP agent. The PCI interface has to be connected to the VPP - they should be configured for use by the Linux kernel and shut down. Vpp-agent can set the interface IP, MAC or set it as enabled. No special configuration fields are defined.

**Memory interface**

Shared memory packet interface (memif) provides high-performance packet transmit and reception between VPPs. The memory interface defines additional fields in `MemifLink`. The most important is a socket filename - the VPP by default contains default socket filename with zero index which can be used to create memif interfaces (the vpp-agent needs to be resynced to register it). 

**Tap interface**

Note: The first version of the Tap interface is no longer supported by the VPP, thus cannot be managed by the vpp-agent. 

The Tap interface exists in two versions, only the latter can be configured. TAPv2 is based on virtio (it knows that runs in a virtual environment, and cooperates with the hypervisor). The `TapLink` provides several setup options available for the TAPv2. 

Field `version` sets the desired version (should be set only to 2), `host_if_name` allows to set a custom name for the TAP interface in the host OS (optional, if not set it will be autogenerated). The option `to_microservice` puts the host TAP to the desired namespace right after creation. Please refer the [interface model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto) to know which fields are defined for specific TAP interface version.

**Af-packet interface**

The Af-packet is the VPP "host" interface, its main purpose is to connect it to the host via the OS VEth interface. Because of this, the af-packet cannot exist without its host interface. If the desired VEth is missing, the configuration is postponed. If the host interface was removed, vpp-agent un-configures and caches related Af-packets. The `AfpacketLink` contains only one field with the name of the host interface (as defined in the [Linux interface plugin](Linux-Interface-plugin.md)).

**VxLan tunnel**

VxLan tunnel endpoint (VTEP) defines `VxlanLink` with source and destination address, a VXLan network identifier (VNI) and a name of the optional multicast interface.

**IPSec tunnel**

IPSec virtual tunnel interface (VTI) provides a routable interface type for terminating IPSec tunnels and an easy way to define protection between sites to form an overlay network. The IPSec tunnel defines `IPSecLink` with various options like local and remote IP address, local and remote security parameter index (SPI) or cryptographic algorithms for encryption and authentication. Available cryptographic algorithms are determined from types supported by the VPP.

**VmxNet3 interface**

VmxNet3 is a virtual network adapter designed to deliver high performance in virtual environments. VmxNet3 requires VMware tools to be used. Type-specific fields are defined in `VmxNet3Link` structure.

### <a name="ifs-config">Configuration</a>

How to configure an interface:

**1. Using key-value database** put the proto-modelled data with the correct key for interfaces. 

Key:
```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

Example value of common interface configuration, since loopback type does not use any type specific fields.
```json
{  
    "name":"loop1",
    "type":"SOFTWARE_LOOPBACK",
    "enabled":true,
    "phys_address":"7C:4E:E7:8A:63:68",
    "ip_addresses":[  
        "192.168.25.3/24",
        "172.125.45.1/24"
    ],
    "mtu":1478
}
```

Use `etcdctl` to put compacted loop interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces//loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
```

Another example with memif type interface. Note memif-specific fields in `memif` section:
```json
{  
    "name":"memif1",
    "type":"MEMIF",
    "enabled":true,
    "phys_address":"4E:93:2A:38:A7:77",
    "ip_addresses":[  
        "172.125.40.1/24"
    ],
    "mtu":1478,
    "memif":{  
        "master":true,
        "id":1,
        "socket_filename":"/tmp/memif1.sock",
        "secret":"secret"
    }
}
```

Use `etcdctl` to put compacted memif interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

**2. Using REST:**

The REST currently supports only retrieval of the existing configuration. The following command can be used to read all interfaces via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

The interface type-specific calls:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/loopback
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/memif
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```

**3. Using GRPC:**

* Prepare the interface data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/interfaces"
)

memoryInterface := &vpp.Interface{
        Name:        "memif1",
        Type:        interfaces.Interface_MEMIF,
        Enabled:     true,
        IpAddresses: []string{"192.168.10.1/24"},
        Link: &interfaces.Interface_Memif{
            Memif: &interfaces.MemifLink{
                Id:             1,
                Master:         true,
                SocketFilename: "/tmp/memif1.sock",
            },
        },
    }
```

* Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI interface commands:** 

The VPP cli has several CLI commands to verify configuration:
 - to show all configured interfaces: `show interface`
 - to show interface IP addresses: `show interface address`
 - show connected PCI interfaces: `show PCI`
 - various information about the state: `show hardware`
 - show VxLan tunnel details: `show vxlan tunnel`
 - show IPSec tunnel details: `show ipsec`

## <a name="ifstate">Interface status</a>

The interface status is a collection of interface data and identification (tagged name, internal name, index) which can be stored to an external database or send a notification.

The vpp-agent waits on several types of VPP events, like creation, deletion or counter update. The event is processed, all the data extracted and using interface index provided in the event data, vpp-agent reads the rest of the interface data from the VPP. Using all the collected information, vpp-agent builds a notification which is sent to all provided publishers (databases). All errors occurred are also stored.

The statistics readout is performed every second.

**Important note:** the interface status is disabled by default, since no default publishers are defined. Use vpp-ifplugin config file to define it (see the [example conf file](https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf))   

### <a name="ifstate-model">Model</a>

The interface state [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto) is represented by two objects: `InterfaceState` with all the status data, and `InterfaceNotification` which is a wrapper around the state with additional information required to send status as a notification.

Keys used for interface status and errors:

```
// Interface status
vpp/config/v2/status/vpp/v2/interface/<name>

// Errors
vpp/config/v2/status/vpp/v2/interface/errors/<name>
```

The interface status is meant to be read-only, published only by the VPP agent.

Data provided by interface status:
    * identification (logical and internal name, software interface index, interface type)
    * administrative and operational status
    * physical address, MTU value
    * used duplex 
    * statistics (received, transmitted or dropped packets, bytes, error count, etc)

### <a name="ifstate-config">Usage</a>

To read status data, use any tool with access to a given database (etcdctl for the ETCD, redis-cli for Redis, boltbrowser for BoltDB, etc). 

Let's read interface status data with `etcdctl`:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/
```

The command above returns status for all interfaces, which have it currently stored inside the ETCD. To read the status just for single interface, add its name to the end:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/loop1
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```

Do not forget to enable status in the config file, otherwise, the status will not be published to the database.