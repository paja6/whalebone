# Knot Resolver

The purpose of this task is to demonstrate how simple it is to set up Knot Resolver and start blocking malicious domains. You will use Knot Resolver's Docker image due to its installation and initial setup simplicity. However, this approach is recommended only for testing purposes. If you would like to deploy Knot Resolver in a production environment, it is recommended to install it as an ordinary application.

This guide was created with Knot Resolver 5.7.4.

## Prerequisites

* A computer with a Linux operating system
* Docker

## Get Knot Resolver

Download the latest Knot Resolver using the following command:

```bash
docker pull cznic/knot-resolver
```

## Run the Docker Image

The command below runs the Docker image and displays the command prompt that allows you to configure the application.

```bash
docker run -it --network=host cznic/knot-resolver
```

Command line parameters:

* -t, --tty: Allocate a pseudo-TTY
* -i, --interactive: Keep STDIN open even if not attached
* --network=host: This allows you to access all ports served by the container from the local host and other machines on the network.

Knot Resolver is now listening on all network interfaces and you can see the configuration command prompt.

## Create a Policy

Execute the following command that blocks all DNS requests for the `google.com` domain and its subdomains. Moreover, it sets a custom error message to clarify the reason why the request was denied. 

```python
policy.add(policy.suffix(policy.DENY_MSG('Google is not allowed here.'), {todname('google.com.')}))
```

## Validate the Configuration

Knot Resolver is running and has been configured. Now, it is time to validate it does what was intended.

### Capture network packets between your computer and Knot Resolver

Tcpdump allows you to see network communication between your computer and Knot Resolver. Let's start with opening a new terminal window or tab where you should enter the following command:

```bash
sudo tcpdump -i lo -vv 'udp port 53'
```

Command line parameters:
* -i lo: Listen on the loopback interface.
* -vv: Increase the log level.
* 'udp port 53': Capture only DNS communication on port UDP/53 to keep the output short and comprehensive.

### Query Knot Resolver to Resolve an Unrestricted Domain

Open a new terminal window or tab and enter there the following command:

```bash
nslookup seznam.cz 127.0.0.1
```

Command line parameters:
* seznam.cz: This is the domain name we want to resolve.
* 127.0.0.1: The request will be sent to the resolver listening on 127.0.0.1.

The answer should be like this:

```
Server:		127.0.0.1
Address:	127.0.0.1#53

Non-authoritative answer:
Name:	seznam.cz
Address: 77.75.79.222
Name:	seznam.cz
Address: 77.75.77.222
Name:	seznam.cz
Address: 2a02:598:2::1222
Name:	seznam.cz
Address: 2a02:598:a::79:222
```

The output from tcpdump shows two packets. The output should look like this:

```
15:02:25.961793 IP (tos 0x0, ttl 64, id 64447, offset 0, flags [none], proto UDP (17), length 55)
    localhost.52468 > localhost.domain: [bad udp cksum 0x1846 -> 0x5d5b!] 22537+ A? seznam.cz. (27)
15:02:25.997240 IP (tos 0x0, ttl 64, id 1155, offset 0, flags [none], proto UDP (17), length 87)
    localhost.domain > localhost.52468: [udp sum ok] 22537 q: A? seznam.cz. 2/0/0 seznam.cz. A 77.75.79.222, seznam.cz. A 77.75.77.222 (59)
15:02:25.997987 IP (tos 0x0, ttl 64, id 28433, offset 0, flags [none], proto UDP (17), length 55)
    localhost.45458 > localhost.domain: [bad udp cksum 0x1846 -> 0x497c!] 65461+ AAAA? seznam.cz. (27)
15:02:26.060145 IP (tos 0x0, ttl 64, id 1156, offset 0, flags [none], proto UDP (17), length 111)
    localhost.domain > localhost.45458: [udp sum ok] 65461 q: AAAA? seznam.cz. 2/0/0 seznam.cz. AAAA 2a02:598:2::1222, seznam.cz. AAAA 2a02:598:a::79:222 (83)
```

As you can see, the DNS server returned the IP addresses that belong to seznam.cz.

### Query Knot Resolver to Test if the Policy is Applied

Open a new terminal window or tab and enter there the following command:

```bash
nslookup google.com 127.0.0.1
```

Command line parameters:
* google.com: This is the domain name we want to resolve.
* 127.0.0.1: The request will be sent to the resolver listening on 127.0.0.1.

The answer should be like this:

```
Server:		127.0.0.1
Address:	127.0.0.1#53

** server can't find google.com: NXDOMAIN
```

NXDOMAIN means that the DNS query failed because the domain name does not exist.

The output from tcpdump shows two packets. The output should look like this:

```
21:50:41.183346 IP (tos 0x0, ttl 64, id 57016, offset 0, flags [none], proto UDP (17), length 56)
    localhost.52468 > localhost.domain: [bad udp cksum 0xfe37 -> 0x70fa!] 44998+ A? google.com. (28)
21:50:41.183518 IP (tos 0x0, ttl 64, id 30768, offset 0, flags [none], proto UDP (17), length 165)
    localhost.domain > localhost.52468: [bad udp cksum 0xfea4 -> 0x8f4b!] 44998 NXDomain* q: A? google.com. 0/1/1 ns: google.com. SOA google.com. nobody.invalid. 1 3600 1200 604800 10800 ar: explanation.invalid. TXT "Google is not allowed here." (137)
```

The `Google is not allowed here.` message has been returned by the DNS server to prove the response came from Knot Resolver.

## Conclusion

Congratulations! You have successfully set up a DNS server that blocks DNS requests for malicious sites. The next step is to create an automation that checks blacklisted domains, e.g., https://fabriziosalmi.github.io/blacklists/, and automatically updates the Knot Resolver's policies.

## Further Reading

* [Knot Resolver website](https://www.knot-resolver.cz)
* [Knot Resolver documentation](https://www.knot-resolver.cz/documentation/v5.7.4/)
