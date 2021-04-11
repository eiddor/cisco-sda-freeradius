# Using FreeRADIUS with Cisco SD-Access

### Introduction
This repo contains most of what you will need in order to configure [FreeRADIUS](https://freeradius.org/) for authentication and authorization with a [Cisco Software-Defined Access](https://www.cisco.com/c/en/us/solutions/enterprise-networks/software-defined-access/index.html) network.

**Note:** [Cisco Identity Services Engine](https://www.cisco.com/c/en/us/products/security/identity-services-engine/index.html) (ISE) is still required for policy in Cisco SD-Access.

Inspired from this [post](https://community.cisco.com/t5/networking-documents/how-to-use-group-based-policies-with-3rd-party-radius-using/ta-p/3930041) and this [video](https://www.youtube.com/watch?v=ZoNKa9X1Xjk&list=UUHDBMhhEDalzaiFGXOhHJDg&index=1).

**Note:** _These are sample files and are only appropriate for a lab environment.  **They are by no means secure** and your production implementation will look much different._

### "Instructions"

The important files in this repository:

- `freeradius/users` - Sample users file for FreeRADIUS 
 
- `freeradius/clients.conf` - Sample clients.conf file for FreeRADIUS 

Although this example is for FreeRADIUS, you can extend this same functionality to other RADIUS servers such as [ForeScout](https://www.forescout.com/) or [ClearPass](https://www.arubanetworks.com/products/security/network-access-control/).

I have also included a `docker-compose.yml` file that will allow you to run FreeRADIUS in a Docker container along with these files.

### TODO
* The configuration files are mostly documented within, but I will document them here in the future.  
 
* Device template files for Cisco switches running IOS-XE.  These would be pushed to the Fabric Edge switches by Cisco DNA Center in order to tell them to use FreeRADIUS for authentication and authorization. 
 
* Better documentation overall

If you have any questions about these files or the purpose of them, please open an [issue](https://github.com/eiddor/cisco-sda-freeradius/issues).