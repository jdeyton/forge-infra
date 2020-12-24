# ntpd

This service uses the OpenNTPD implementation to provide an NTP server for a
local subnet.

## How to Build

It is assumed that you need to build with the host network so that alpine
packages can be resolved and installed.

```
docker build --network host -t forge-ntpd .
```

## How to Build

You can use regular builds:

```
docker build --network host -t forge-ntpd .
```

Or you can use the compose file:

```
docker-compose build forge-ntpd
```

## How to Use

Create your own **macvlan** network and update the `docker-compose.yml` to use
it or the **host** network driver. (I went with a macvlan to ensure the service
has a static IP address that is different from the host--in case the host is
replaced--and so that is accessible from the entire subnet outside the host.)

- Update the `ntpd.conf` with your service's *static* IP address.
- Update the `docker-compose.yml` with your service's *static* IP address.
- Update the `iptables.rules` with your subnet and ethernet device details
(typically `eth0` is the default device for alpine).

To start the service, you can run it with docker:

```
docker run --cap-add NET_ADMIN --cap-add SYS_NICE --cap-add SYS_TIME --network=my-macvlan --ip=192.168.3.42 --rm -it forge-ntpd
```

Or you can use the compose file:

```
docker-compose up forge-ntpd
```

## Notes

There are some tricks to get `ntpd` to work from a container. These are already
incorporated into the recipe here:

- The base image requires the following container capabilities:
  * `NET_ADMIN`
- OpenNTPD requires the following capabilities:
  * `SYS_NICE`
  * `SYS_TIME`
- OpenNTPD needs a home directory for the user that runs `ntpd` with specific ownership/permissions. This is created in the Dockerfile.
- OpenNTPD makes non-NTP queries upstream, including DNS queries and the constraint checks, so firewall rules must account for this.

### Example - Starting the Server

OpenNTPD's logged output can be noisy. Here is an example. You will note that:

- IPv6 is not being used, yet output about it is logged.
- NTP queries are made to pool.ntp.org.
- Queries are also made to the constraint (www.google.com).

```
user@host:/services/ntpd# docker-compose up forge-ntpd
WARNING: The Docker Engine you're using is running in swarm mode.

Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

To deploy your application across the swarm, use `docker stack deploy`.

Creating forge-ntpd ... done
Attaching to forge-ntpd
forge-ntpd    | creating new /var/db/ntpd.drift
forge-ntpd    | adjtimex adjusted frequency by 0.000000ppm
forge-ntpd    | listening on 192.168.3.42
forge-ntpd    | ntp engine ready
forge-ntpd    | constraint request to 74.125.198.99
forge-ntpd    | constraint request to 74.125.198.104
forge-ntpd    | constraint request to 74.125.198.103
forge-ntpd    | constraint request to 74.125.198.105
forge-ntpd    | constraint request to 74.125.198.147
forge-ntpd    | constraint request to 2607:f8b0:4003:c05::6a
forge-ntpd    | constraint request to 74.125.198.106
forge-ntpd    | constraint request to 2607:f8b0:4003:c05::69
forge-ntpd    | tls connect failed: 2607:f8b0:4003:c05::6a (www.google.com): connect: Address not available
forge-ntpd    | no constraint reply from 2607:f8b0:4003:c05::6a received in time, next query 900s
forge-ntpd    | tls connect failed: 2607:f8b0:4003:c05::69 (www.google.com): connect: Address not available
forge-ntpd    | no constraint reply from 2607:f8b0:4003:c05::69 received in time, next query 900s
forge-ntpd    | constraint reply from 74.125.198.106: offset -1.021405
forge-ntpd    | constraint reply from 74.125.198.104: offset -0.060555
forge-ntpd    | reply from 72.87.88.202: offset 0.126591 delay 0.087534, next query 6s
forge-ntpd    | set local clock to Thu Dec 24 07:25:06 UTC 2020 (offset 0.126591s)
forge-ntpd    | constraint reply from 74.125.198.103: offset -0.277867
forge-ntpd    | constraint reply from 74.125.198.105: offset -0.299484
forge-ntpd    | constraint reply from 74.125.198.147: offset -0.314066
forge-ntpd    | constraint reply from 74.125.198.99: offset -0.314229
```

### Example - Querying the Server

Unfortunately, `ntpq` does not play well with OpenNTPD, but you can check that
your new service is accessible on your subnet with the deprecated `ntpdate`
utility:

```
user@other:~# ntpdate -q 192.168.3.42
server 192.168.3.42, stratum 16, offset -3.020855, delay 0.02664
22 Dec 00:16:03 ntpdate[29939]: no server suitable for synchronization found
```

This shows a warning that the server hasn't been up long enough to be
trustworthy. The fact that you got this info is a good sign that your service is
running. If you try the same command later, you should get output like this:

```
user@other:~# ntpdate -q 192.168.3.42
server 192.168.3.42, stratum 3, offset -1.571508, delay 0.02657
22 Dec 00:24:34 ntpdate[29982]: step time server 192.168.3.42 offset -1.571508 sec
```