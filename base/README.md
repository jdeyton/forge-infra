# base

This is the base for services that will expose ports to a subnet. A functioning
firewall is a minimal expectation of such services as we do not anticipate
providing all services to all networked devices.

## How to Build

It is assumed that you need to build with the host network so that alpine
packages can be resolved and installed.

```
docker build --network host -t forge-base .
```

## How to Use

Create a file `iptables.rules` with content like this:

```
*filter

# Policies:
:INPUT DROP
:FORWARD DROP
:OUTPUT ACCEPT

# Accept local traffic:
-A INPUT -i lo -j ACCEPT

# Create a chain for your network interface:
-N eth0
-A INPUT -i eth0 -j eth0

# Allow incoming connections tied to outgoing connections:
-A eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Apply other firewall rules...

COMMIT
```

Create your service's `Dockerfile` with your startup command like this:

```
FROM forge-base:latest
COPY iptables.rules .
CMD ./init && <your service command>

# Do your other Dockerfile stuff.
```

**Important:** Run your container with the `NET_ADMIN` capability. Confirm that
it applies the desired firewall rules and behaves as expected.