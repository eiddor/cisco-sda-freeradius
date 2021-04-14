# Using FreeRADIUS with Cisco Software-Defined Access

### Introduction
This repo contains most of what you will need in order to configure [FreeRADIUS](https://freeradius.org/) for authentication and authorization with a [Cisco Software-Defined Access](https://www.cisco.com/c/en/us/solutions/enterprise-networks/software-defined-access/index.html) network.

**Note:** _[Cisco Identity Services Engine](https://www.cisco.com/c/en/us/products/security/identity-services-engine/index.html) (ISE) is still required for policy in Cisco SD-Access._

Inspired from this [post](https://community.cisco.com/t5/networking-documents/how-to-use-group-based-policies-with-3rd-party-radius-using/ta-p/3930041) and this [video](https://www.youtube.com/watch?v=ZoNKa9X1Xjk&list=UUHDBMhhEDalzaiFGXOhHJDg&index=1). 


Although this example is for FreeRADIUS, you can extend this same functionality to other RADIUS servers such as [ForeScout](https://www.forescout.com/) or [ClearPass](https://www.arubanetworks.com/products/security/network-access-control/). 

**Note:** _These are sample files and are only appropriate for a lab environment.  **They are by no means secure** and your production implementation will look much different._

### Getting Started

The important FreeRADIUS files in this repository are:

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

The two key settings that you should change to match your environment are:

`ipaddr` - IP address of the client device(s) that will be querying RADIUS.  In our example we're allowing any device.

`secret` - The RADIUS secret that the client device will use to authenticate.

The official documentation for this file can be found [here](https://freeradius.org/radiusd/man/clients.html) 

See [here](https://github.com/FreeRADIUS/freeradius-server/blob/v3.0.x/raddb/clients.conf) for a well-documented example.

#### `freeradius/users`

The FreeRADIUS `users` file contains both the authentication credentials and authorization result information about users logging into the network.  In a production environment your RADIUS server would usually point to a database, LDAP, or AD server for the authentication credentials.

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

#### Running in Docker

I've included a `docker-compose.yml` file that should allow you to easily run FreeRADIUS in a Docker container along with these files. 

```yaml
version: '3'

services:
     freeradius:
         image: roddie/cisco-sda-freeradius:latest
         container_name: freeradius
         ports:
             - "1812-1813:1812-1813/udp"
         volumes:
             - "./freeradius/clients.conf:/etc/raddb/clients.conf"
             - "./freeradius/users:/etc/raddb/mods-config/files/authorize"
         restart: unless-stopped
```

This file expects our custom files to exist in `./freeradius` on the host.  Feel free to modify the `volume:` section to match your environment.

You can start the container running in the background with: `docker-compose up -d freeradius`

You can also run it directly with:

`docker run --name freeradius -p 1812-1813:1812-1813/udp -v "/$(pwd)/freeradius/clients.conf:/etc/raddb/clients.conf" -v "/$(pwd)/freeradius/users:/etc/raddb/mods-config/files/authorize" roddie/cisco-sda-freeradius:latest`

The image referenced in `docker-compose.yml` is a [custom Docker image](https://hub.docker.com/r/roddie/cisco-sda-freeradius) for use with this project which includes a fix to push the RADIUS attributes to the switch inside of the tunneled Access-Accept message (see note in non-Docker section below). 

I have also included the `Dockerfile` and accompanying files in the `docker/` directory in case you want to build your own image.
#### Running outside of Docker

If you have an existing FreeRADIUS implementation to use or would like to run FreeRADIUS natively on your server, you should be able to incorporate the information from the examples in this repository easily.

---
**Note:** If you are using a tunneled authentication method such as EAP, you will need to tell FreeRADIUS to include the RADIUS attributes with the Access-Accept message inside of the tunnel. 

In the version used in this repository, that setting is `use_tunneled_reply = yes`, however it may vary depending on your version of FreeRADIUS and how you have it setup.  This is already enabled in the Docker image discussed above.

See [this post](https://community.cisco.com/t5/switching/catalyst-2960-ignores-radius-attributes-to-set-vlan/m-p/1792633/highlight/true#M192521) for a better explanation.

---
### Testing

Before deploying the Cisco IOS-XE templates and testing with fabric users, you should probably run a local test using the `radtest` command (available in the container or in the `freeradius-utils` package).  This will test your RADIUS configuration and allow you to see the authorization results that RADIUS will return to your switches.

```console
roddie@testubuntu ~% radtest user1 password 127.0.0.1 10 radiussecret
Sent Access-Request Id 184 from 0.0.0.0:57731 to 127.0.0.1:1812 length 75
        User-Name = "user1"
        User-Password = "password"
        NAS-IP-Address = 127.0.1.1
        NAS-Port = 10
        Message-Authenticator = 0x00
        Cleartext-Password = "password"
Received Access-Accept Id 184 from 127.0.0.1:1812 to 127.0.0.1:57731 length 103
        Service-Type = Framed-User
        Tunnel-Private-Group-Id:0 = "1021"
        Tunnel-Type:0 = VLAN
        Tunnel-Medium-Type:0 = IEEE-802
        Cisco-AVPair = "cts:vn=Campus"
        Cisco-AVPair = "cts:security-group-tag=0016-00"
```

We can see above that we authenticate successfully with our credentials as our RADIUS server is sending back an **Access-Accept**.  Inside of the Access-Accept message, the RADIUS server is also sending our configured attributes back to tell the switch which VLAN (1021) we should be placed on along with our SGT and optional VN.


### Usage

The file `cisco/cat9k-template.txt` is a sample of what should be pushed to a Cisco Catalyst 9000 switch in order to redirect the authentication and authorization function to your FreeRADIUS server.  This template should be pushed using [Cisco DNA Center](https://www.cisco.com/c/en/us/products/cloud-systems-management/dna-center/index.html and it will not remove the existing configuration that is pointing to ISE for policy.

The file is mostly documented, but I will add more information here, too.
### TODO

* Modify `docker-compose.yml` to provide logging from the container.
 
* Better documentation overall.

If you have any questions about these files or the purpose of them, please open an [issue](https://github.com/eiddor/cisco-sda-freeradius/issues).