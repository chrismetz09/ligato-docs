# Linux L3 plugin

The Linux L3 plugin is capable of configuring **Linux ARP** entries and **Linux routes**. The Linux L3 plugin is dependent on [Linux interface plugin](Linux-Interface-plugin.md) since both configuration items are dependent on the Linux interface managed by it.

L3 configuration items support Linux namespace in the same manner as the Linux interfaces.  

- [Linux ARP entries](#linux-arps)
  * [Model](#linux-arpmodel)
  * [Configuration](#linux-arpconf)
- [Linux Routes](#linux-routes)
  * [Model](#linux-routesmodel)
  * [Configuration](#linux-routesconf)

## <a name="linux-arps">Linux ARP entries</a>  

Linux ARP is a communication protocol which helps to discover a MAC address associated with the given IP address. The plugin makes a use of the very simple proto model with interface, IP address and MAC address fields. All those fields are mandatory.

The namespace resolution is automatic and user does not need to care about it. Since the ARP entry is dependent on the associated Linux interface, the configuration is postponed till the moment the interface appears. Target namespace is derived from the interface identified by the unique name. The interface host name has to be unique within the namespace scope.   

## <a name="linux-arpmodel">Model</a>  

The northbound Linux plugin API defines ARP in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/arp.proto). Every ARP entry is defined with MAC address, IP address, and an interface. The IP address and the interface are a part of the key.

In the generated proto model, the ARP is referred to the `ARPEntry` object.

## <a name="linux-arpconf">Configuration</a> 

How to configure Linux ARP entry:

**1. Using a key-value database** put the proto-modeled data with the correct key for the Linux ARP to the database. 

Key:
```
/vnf-agent/vpp1/config/linux/l3/v2/arp/<interface>/<ip_address>
```

Example value:
```json
{  
    "interface":"veth1",
    "ip_address":"130.0.0.1",
    "hw_address":"46:06:18:DB:05:3A"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1 '{"interface":"veth1","ip_address":"130.0.0.1","hw_address":"46:06:18:DB:05:3A"}'
```

To remove configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/arps
```

**3. Using GRPC:**

* Prepare the Linux ARP data:
```go
linuxArp := &linuxL3.ARPEntry{
		Interface: "linuxIf1",
		IpAddress: "130.0.0.1",
		HwAddress: "46:06:18:DB:05:3A",
	}
```

* Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			ArpEntries []*l3.ARPEntry {    
			    linuxArp,
			},	
		}
	}
```

The config data can be combined with any other Linux or VPP configuration.

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## <a name="linux-routes">Linux Routes</a>  

A routing table is a data table listing the routes to particular network destinations and metrics (distances) associated with routes. The route interface is mandatory (the namespace is derived from the interface, the same principle as for the Linux ARP). If source and gateway IP addresses are set, they must be valid and the gateway IP address must be evaluated as reachable.
Every route defines a scope, the options are 'Global', 'Site', 'Link' and 'Host'. The scope is also expected to be defined the route config.  

## <a name="linux-routesmodel">Model</a>

The northbound Linux plugin API defines Route in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/route.proto). 
In the generate file, the route is reffered as `Route`.

## <a name="linux-routesconf">Configuration</a> 

How to configure a Linux Route:

**1. Using a key-value database** put the proto-modeled data with the correct key for the Linux route to the database. 

Key:
```
/vnf-agent/vpp1/config/linux/l3/v2/route/<dst_network>/<outgoing_interface>
```

Example configuration:
```json
{  
    "outgoing_interface":"linuxIf1",
    "dst_network":"10.0.2.0/24",
    "metric":100
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1 '{"outgoing_interface":"veth1","dst_network":"10.0.2.0/24","metric":100}'
```

To remove configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/routes
```

**3. Using GRPC:**

Prepare the Linux route data:
```go
linuxRoute := &linuxL3.Route{
		DstNetwork:        "10.0.2.0/24",
		OutgoingInterface: "veth1",
		Metric:            100,
		Scope:             linuxL3.Route_HOST,
	}
```

* Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			Routes: []*l3.Route  {    
			    linuxRoute,
			},	
		}
	}
```

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```


