# Using FreeRADIUS with Cisco Software-Defined Access

### Introduction
This repo contains most of what you will need in order to configure [FreeRADIUS](https://freeradius.org/) for authentication and authorization with a [Cisco Software-Defined Access](https://www.cisco.com/c/en/us/solutions/enterprise-networks/software-defined-access/index.html) network.

**Note:** [Cisco Identity Services Engine](https://www.cisco.com/c/en/us/products/security/identity-services-engine/index.html) (ISE) is still required for policy in Cisco SD-Access.

Inspired from this [post](https://community.cisco.com/t5/networking-documents/how-to-use-group-based-policies-with-3rd-party-radius-using/ta-p/3930041) and this [video](https://www.youtube.com/watch?v=ZoNKa9X1Xjk&list=UUHDBMhhEDalzaiFGXOhHJDg&index=1). 


Although this example is for FreeRADIUS, you can extend this same functionality to other RADIUS servers such as [ForeScout](https://www.forescout.com/) or [ClearPass](https://www.arubanetworks.com/products/security/network-access-control/). 

**Note:** _These are sample files and are only appropriate for a lab environment.  **They are by no means secure** and your production implementation will look much different._

### "Instructions"

The important files in this repository:

#### `freeradius/clients.conf`

The FreeRADIUS `clients.conf` file specifies the  parameters for the _RADIUS clients_, typically  network switches.  In Cisco ISE these are known as network access devices (NADs.)  This is not the same as the user devices that will actually be logging into the network.

```sh
client switches {
  ipaddr = 0.0.0.0/0
  proto = *
  secret = radiussecret
  nas_type = cisco
}
```

The two important settings that you should change to match your environment are:

`ipaddr` - IP address of client device that will be querying RADIUS.

`secret` - The RADIUS secret that the client device will use to authenticate.

The official documentation for this file can be found [here](https://freeradius.org/radiusd/man/clients.html) 

See [here](https://github.com/FreeRADIUS/freeradius-server/blob/v3.0.x/raddb/clients.conf) for a well-documented example.

#### `freeradius/users`

The FreeRADIUS `users` file contains both the authentication credentials and authorization result information about users logging into the network.  In a production environment your RADIUS server would usually point to an LDAP or AD server for the authentication credentials.

```sh
user Cleartext-Password := "password"
	Service-Type = Framed,
	Tunnel-Private-Group-ID = "1021",
	Tunnel-Type = 13,
	Tunnel-Medium-Type = 6,
	Cisco-AVPair = "cts:vn=Campus",
	Cisco-AVPair += "cts:security-group-tag=0016-00"
```

In the example above `user` is the username and `password` is the user's password. 

Once the user is authenticated to the network, the rest of the settings are results that are pushed to the client device for authorization.

`Tunnel-Private-Group-ID` - The VLAN ID or VLAN name that the authenticated user should be placed in.

`Cisco-AVPair += "cts:security-group-tag=0016-00"` - The Scalable (or Security) Group Tag (SGT) that will be assigned to the user's traffic.  In this example, we are using `0016`.  The SGT must be in HEX format and you can view the value in Cisco DNA Center or in Cisco ISE.  The `-00` is also mandatory and is the SGT "Generation" which can be viewed in Cisco ISE.

`Cisco-AVPair = "cts:vn=Campus"` - (optional) The Cisco SD-Access Virtual Network that the user will be placed in.  This setting is currently optional as the VLAN will usually dictate which VN the user is in.

The official documentation for this file can be found [here](https://freeradius.org/radiusd/man/users.html).

See [here](https://github.com/FreeRADIUS/freeradius-server/blob/v3.0.x/raddb/mods-config/files/authorize) for a well-documented example.

I have also included a `docker-compose.yml` file that will allow you to run FreeRADIUS in a Docker container along with these files. (_documentation coming soon_)

### TODO

* Device template files for Cisco switches running IOS-XE.  These are pushed to the Fabric Edge switches by [Cisco DNA Center](https://www.cisco.com/c/en/us/products/cloud-systems-management/dna-center/index.html) in order to tell them to use FreeRADIUS for authentication and authorization. 
 
* Better documentation overall.

If you have any questions about these files or the purpose of them, please open an [issue](https://github.com/eiddor/cisco-sda-freeradius/issues).