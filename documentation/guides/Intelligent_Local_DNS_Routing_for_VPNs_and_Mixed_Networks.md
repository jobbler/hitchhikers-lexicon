# **Intelligent Local DNS Routing for VPNs & Mixed Networks** 
_A Guide to configuring BIND9 and systemd-resolved for seamless connectivity._

| **Author:** | John Herr         |
| :---------- | ----------------- |
| **Date:**   | November 20, 2025 |


> **⚠️ Editor's Note/Disclaimer:** This document was collaboratively reviewed and refined by an expert AI (Gemini) to enhance its clarity, structure, and readability for publication. The core technical content, expertise, and original configuration details were provided by the author.


This guide details a robust solution for managing complex, conditional DNS resolution needs on a Fedora workstation using a local BIND (`named`) instance as a caching and conditional nameserver, integrated with `systemd-resolved` and NetworkManager.

---

## Problem Statement

In complex networking environments, a developer’s machine must often resolve hostnames across multiple isolated networks simultaneously (e.g., the Public Internet, Corporate VPNs, and isolated Lab environments). Standard DHCP-provided DNS configurations are often insufficient because they enforce an "all-or-nothing" routing approach, where connecting to a VPN either overrides all DNS resolution or fails to resolve internal subdomains entirely.

### Environment

The configuration must support a local machine connecting to:

- **Public Internet:** General web access and fallback resolution.    
- **VPN-1 (Corporate):** Primary company domain (`company.example`).    
- **Lab-1 (Subdomain):** Specific subdomains (`lab1.company.example`) and OpenShift clusters (`example.org`).    
- **VPN-2 (External Lab):** Isolated lab domains (`lab2.example`).    


``` mermaid
graph LR;

laptop[Local Laptop or Desktop]
lab1dns[DNS: lab1.company.example]
lab2dns[DNS: lab2.example]
clusterdns[DNS: example.org clusters]
pubdns[DNS: Public Resolution]
companydns[DNS: Company Resolution]

laptop --- lab1dns;
laptop --- clusterdns;
laptop --- lab2dns;
laptop --- pubdns;

laptop -.- companydns;
laptop -.- pubdns;

subgraph VPN-2;
  subgraph Lab-2;
    lab2dns;
    end
end

subgraph VPN-1;
  companydns;
  subgraph Lab-1;
    lab1dns;
    clusterdns;
  end
end

subgraph Internet
    pubdns;
end
```

### Solution Goals

The objective is to configure a local DNS endpoint that intelligently routes queries based on the requested domain, satisfying three key requirements:

1. **Conditional Forwarding (Split-Horizon DNS):** Queries must be routed to specific upstream nameservers based on the domain (e.g., `lab1.company.example` → Lab DNS, while `company.example` → Corporate DNS).

2. **VPN Bootstrapping & Failover:** The resolver must gracefully handle scenarios where internal nameservers are unreachable (such as when the VPN is disconnected). It must automatically fall back to public nameservers to resolve the VPN endpoints themselves without user intervention.

3. **Persistence:** The configuration should rely on static definition files rather than ephemeral, dynamic state changes triggered by network interface events.    


### Architectural Decision: BIND9 vs. dnsmasq

Two common local resolvers were evaluated for this solution: **dnsmasq** and **BIND9**.

- **dnsmasq:** While lightweight, dnsmasq typically treats an unreachable upstream server as a hard failure for that specific domain. To support VPN bootstrapping (where the internal server is initially unreachable), dnsmasq requires complex workarounds, such as using `NetworkManager-dispatcher` scripts to dynamically inject and remove DNS entries via D-Bus upon VPN connection events.

- **BIND9:** BIND supports robust fallback logic within a static configuration. If a specific forwarder is unreachable, BIND can be configured to fall back to the default public forwarder. This allows the system to resolve the public VPN endpoints without requiring dynamic reconfiguration of the DNS service every time the VPN toggles.    

**Conclusion:** BIND9 is selected as the primary solution due to its stability and ability to function without reliance on external network state scripts.


### Solution Overview

The implemented solution utilizes a three-part configuration on the local machine:

1. **systemd-resolved:** Configured as a stub listener to direct _all_ DNS queries to the local loopback address.

2. **NetworkManager:** Configured to ignore DNS settings pushed by DHCP or VPN servers, ensuring they do not override the local configuration.

3. **BIND (named):** Configured to serve as the intelligent router for all DNS traffic.    

#### BIND Configuration Strategy

The BIND configuration defines the local listener port (**5353**) to avoid conflicts, specifies public fallback nameservers, and establishes **conditional forward zones** for all custom domains.

> **Technical Note on Zone Priority**: 
> For standard type forward zones, BIND utilizes a "longest-match" algorithm, meaning it automatically selects the most specific zone (e.g., `lab1.company.example`) over a parent zone (`company.example`) regardleThis document was collaboratively reviewed and refined by an expert AI (Gemini) to enhance its clarity, structure, and readability for publication. The core technical content, expertise, and original configuration details were provided by the author.ss of their order in the configuration file. However, if your configuration utilizes BIND views or complex **ACLs**, the order of definition may become significant, as those features often process rules sequentially.


-----

## Configure systemd-resolved and NetworkManager

### Reroute DNS Queries in resolved.conf

Modify the `/etc/systemd/resolved.conf` file to instruct `systemd-resolved` to use the local BIND instance listening on port **5353** for all non-local domains (`~.`).

```conf
# /etc/systemd/resolved.conf

# Use the local DNS server
DNS=127.0.0.1:5353

# Route all unknown domains to the above DNS server
Domains=~.

# Disable LLMNR support
LLMNR=false

# Do not cache failed lookups for better VPN fallback
Cache=no-negative
```

Restart the service for the changes to take effect:

```bash
sudo systemctl restart systemd-resolved
```

### Configure Interfaces to Ignore DHCP DNS

By default, NetworkManager connections will push their own DNS servers, overriding the settings in `systemd-resolved`. You must disable automatic DNS configuration on your primary interfaces (wired, wireless, and VPNs).

This can be seen using the `resolvectl status` command.

```bash title:"resolvectl status (Before changes)"
resolvectl status

Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub
Current DNS Server: 127.0.0.1:5353
       DNS Servers: 127.0.0.1:5353
        DNS Domain: ~.

Link 2 (enp0s31f6)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 192.168.1.2
        DNS Domain: home.example

Link 4 (wlp0s20f3)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported

Link 15 (tun0)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 10.0.0.2 10.0.0.3
        DNS Domain: company.example
```

To prevent DHCP/VPN DNS servers from overriding the local setup, modify each relevant connection using **NetworkManager CLI (nmcli)** to ignore automatically configured DNS settings:

```bash
sudo nmcli connection modify wired ipv4.ignore-auto-dns true
sudo nmcli connection modify vpn-1 ipv4.ignore-auto-dns true
sudo nmcli connection modify vpn-2 ipv4.ignore-auto-dns true
# Repeat for any other primary interfaces (e.g., "wifi")
```

For the changes to be applied, bring the affected interfaces down and then up again (or simply reboot the machine).

Verify the change using `resolvectl status`. The `Link` sections should no longer show any `DNS Servers` from DHCP:

```bash title:"resolvectl status (After changes)"
resolvectl status

Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub
Current DNS Server: 127.0.0.1:5353
       DNS Servers: 127.0.0.1:5353
        DNS Domain: ~.

Link 2 (enp0s31f6)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported

Link 4 (wlp0s20f3)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported

Link 15 (tun0)
    Current Scopes: none
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```



## Configuring BIND (named)

### Installation

Install BIND and the utility package on your Fedora system:

```bash
sudo dnf install bind bind-utils
```

### Configure named.conf

The BIND configuration will define the listener port (**5353**), specify public fallback nameservers, and set up **conditional forward zones** for all custom domains.

The key directives are:

  * `listen-on port 5353`: The port `systemd-resolved` is now configured to use.
  * `forward only`: Ensures BIND only uses its configured forwarders.
  * `forwarders`: Sets the global **default public DNS** (e.g., 8.8.8.8, 4.4.4.4) for any domain not covered by a specific zone.
  * `zone "..." { type forward; ... }`: Sets up conditional forwarding for specific domains.

Place the following configuration in `/etc/named.conf`:

```conf
// /etc/named.conf

options {
	// Listen on port 5353, allowing queries from any interface (including localhost)
	listen-on port 5353 { any; }; 

	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { any; };

  dnssec-validation no;
  recursion yes;

  // Use forwarders only, do not attempt root-server lookups
  forward only; 

  // Global Public Fallback Nameservers
  forwarders {
      8.8.8.8;
      4.4.4.4;
  };

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	include "/etc/crypto-policies/back-ends/bind.config";
};

// --- Conditional Forward Zones ---

// Company Internal Nameservers (Forward and Reverse)
// If the internal servers (10.0.x.x) are unreachable (e.g., VPN down), BIND falls back to the global forwarders (8.8.8.8/4.4.4.4).
zone "company.example" {
    type forward;
    forwarders {
        10.0.100.39;
    };
};
zone "100.0.10.in-addr.arpa" { // Example reverse zone for 10.0.100.x
    type forward;
    forwarders {
        10.0.100.39;
    };
};


// Lab 1 (Subdomain)
zone "lab1.company.example" {
    type forward;
    forwarders {
        10.0.80.2;
        10.0.80.3;
    };
};
zone "80.0.10.in-addr.arpa" {
    type forward;
    forwarders {
        10.0.80.2;
        10.0.80.3;
    };
};

// OpenShift Cluster in Lab 1 (example.org)
zone "example.org" {
    type forward;
    forwarders {
        10.0.100.39;
    };
};
zone "100.0.10.in-addr.arpa" {
    type forward;
    forwarders {
        10.0.100.39;
    };
};

// Lab 2
zone "lab2.company.example" {
    type forward;
    forwarders {
        192.168.13.11;
    };
};
zone "13.168.192.in-addr.arpa" {
    type forward;
    forwarders {
        192.168.13.11;
    };
};
```

### Start and Enable BIND

Enable the `named` service to start on boot and start it immediately:

```bash
sudo systemctl enable named --now
```

### SELinux Configuration

If you are using a non-standard port like 5353, you may need to adjust SELinux rules to allow BIND to listen on that port:

```bash
sudo setsebool -P nis_enabled 1
```

-----

## Verification
To ensure all DNS queries are correctly routing through your new local BIND setup and then conditionally forwarded, use the following monitoring tools.

### Verify `systemd-resolved` Activity

Use `resolvectl monitor` to confirm that `systemd-resolved` receives and forwards queries:

``` bash
resolvectl monitor
```

### Monitor BIND (`named`) Activity

Use `journalctl` to watch BIND's logs, which will show which forwarder it attempts to use for each domain:

``` bash
sudo journalctl -xefu named
```

### Direct Query Testing (Client to BIND)

Test both a public domain (using the global forwarders) and a custom domain (using a conditional forwarder) by querying the local BIND server directly:

``` bash
# Public Domain (Uses 8.8.8.8/4.4.4.4)
dig @localhost -p 5353 google.com

# Lab 1 Domain (Uses 10.0.80.2/3)
dig @localhost -p 5353 host.lab1.company.example
```

### Packet-Level Verification using `tcpdump`

For definitive proof that the DNS lookups are going to the intended upstream servers, use `tcpdump` to watch the DNS traffic on your physical interfaces. Replace `eth0` with your active network interface (e.g., `enp0s31f6` or `wlp0s20f3`).

#### Scenario A: Testing Global Forwarding (Public Domains)

Run this command, then perform a DNS lookup for a public domain (e.g., `yahoo.com`):

``` bash
# Listen for DNS packets on port 53 (not 5353) to/from the public internet servers
sudo tcpdump -i eth0 port 53 and host 8.8.8.8 -nn
```

#### Scenario B: Testing Conditional Forwarding (Internal Domains)

Run this command, then perform a DNS lookup for one of your internal domains (e.g., `host.lab1.company.example`). You should see packets targeting the specific internal DNS server (e.g., `10.0.80.2`):

``` bash
# Listen for DNS packets on port 53 to/from the Lab 1 DNS server
sudo tcpdump -i eth0 port 53 and host 10.0.80.2 -nn
```

This ensures that the entire chain—client $\rightarrow$ `systemd-resolved` $\rightarrow$ BIND (5353) $\rightarrow$ Upstream DNS (53)—is functioning as intended.


-----


## Alternative: Using dnsmasq (The Caveat)

While **BIND** is preferred for its robust fallback behavior (using global forwarders when conditional forwarders are unreachable), **dnsmasq** is a viable alternative if you use NetworkManager's dispatcher scripts to dynamically manage the conditional servers.

> **Note on dnsmasq issues:** A common frustration with `dnsmasq` in this conditional forwarding setup is its default behavior: if a conditional nameserver is unreachable (e.g., the VPN is down), `dnsmasq` often returns no results instead of falling back to the global public forwarders. This breaks the ability to resolve the VPN gateway's public FQDN, preventing the VPN connection entirely. The standard workaround requires complex D-Bus scripting via `NetworkManager-dispatcher` to dynamically add and remove internal DNS entries, which BIND handles inherently with its forward-zone logic.

### dnsmasq Workaround (Dynamic Configuration)

To use `dnsmasq`, you must **omit** the VPN-dependent domain (`company.example`) from the static `/etc/dnsmasq.d/10-conditional.conf` and instead use **NetworkManager-dispatcher** scripts to add or remove it via D-Bus when the VPN interface (`tun0`) comes up or goes down.

#### Static Configuration (`10-conditional.conf`)

Note the `company.example` lines are commented out:

```conf
# /etc/dnsmasq.d/10-conditional.conf (Partial)

# Public dns servers - These are the default dns servers used for anything not specified here.
server=8.8.8.8
server=8.8.4.4

# ... (Lab 1, Lab 2, Cluster configurations remain the same) ...

# These are commented out and then dynamically added through d-bus
#server=/company.example/10.0.0.2
#server=/company.example/10.0.0.3
#rev-server=10.0.0.0/8,10.0.0.2
#rev-server=10.0.0.0/8,10.0.0.3
```

#### NetworkManager-dispatcher Script

The script must convert the IP addresses to their **uint32 integer form** for the D-Bus commands.

##### IP Address to `uint32` Integer Conversion

The D-Bus command used by `NetworkManager-dispatcher` to communicate with `dnsmasq` requires the IP address to be expressed as a 32-bit unsigned integer (`uint32`). This integer representation is known as the **decimal value** of the IP address.

To convert a standard dotted-decimal IPv4 address (A.B.C.D) to its decimal integer equivalent, you multiply each octet by a decreasing power of 256, starting from $256^3$.

The formula is:
$$\text{uint32 Value} = (A \times 256^3) + (B \times 256^2) + (C \times 256^1) + (D \times 256^0)$$

##### Example: Converting `10.0.0.2`

Let's convert the internal DNS server **10.0.0.2** using the formula:
- **A** = 10
- **B** = 0    
- **C** = 0    
- **D** = 2

|**Octet**|**Value**|**Power of 256**|**Calculation**|**Result**|
|---|---|---|---|---|
|A|10|$256^3$ (16,777,216)|$10 \times 16,777,216$|167,772,160|
|B|0|$256^2$ (65,536)|$0 \times 65,536$|0|
|C|0|$256^1$ (256)|$0 \times 256$|0|
|D|2|$256^0$ (1)|$2 \times 1$|2|
|**Sum**||||**167,772,162**|

The `uint32` equivalent for `10.0.0.2` is **167772162**.

##### Example: Converting `10.0.0.3`

|**Octet**|**Value**|**Power of 256**|**Calculation**|**Result**|
|---|---|---|---|---|
|A|10|$256^3$|$10 \times 16,777,216$|167,772,160|
|B|0|$256^2$|$0 \times 65,536$|0|
|C|0|$256^1$|$0 \times 256$|0|
|D|3|$256^0$|$3 \times 1$|3|
|**Sum**||||**167,772,163**|

The `uint32` equivalent for `10.0.0.3` is **167772163**.

##### D-Bus Script Usage

These calculated values are then used directly in the `dbus-send` commands:

Bash

```
# ... (Part of the dispatcher script)
dbus-send --system --print-reply \
    --dest=uk.org.thekelleys.dnsmasq \
      /uk/org/thekelleys/dnsmasq \
      uk.org.thekelleys.dnsmasq.SetServers \
      uint32:167772162 string:"company.example" \
      uint32:167772163 string:"company.example"
# ...
Intelligent Local DNS Routing for VPNs & Mixed Networks A Guide to configuring BIND9 and systemd-resolved for seamless connectivity.```


The script `/etc/NetworkManager/dispatcher.d/10-global-vpn` manages the dynamic entries:

```bash
#! /bin/bash

interface=$1
event=$2

## Global VPN - Only run when the VPN interface (tun0) is active
[[ $1 == "tun0" ]] && [[ $2 == "up" ]] && {
  # Add company DNS servers for company.example
  dbus-send --system --print-reply \
    --dest=uk.org.thekelleys.dnsmasq \
      /uk/org/thekelleys/dnsmasq \
      uk.org.thekelleys.dnsmasq.SetServers \
      uint32:167772162 string:"company.example" \
      uint32:167772163 string:"company.example"

  # Clear dnsmasq's cache
  dbus-send --print-reply --system \
    --dest=uk.org.thekelleys.dnsmasq \
      /uk/org/thekelleys/dnsmasq \
      uk.org.thekelleys.dnsmasq.ClearCache
}

[[ $1 == "tun0" ]] && [[ $2 == "down" ]] && {
  # Remove dynamic entries by sending a blank list
  dbus-send --system --print-reply \
    --dest=uk.org.thekelleys.dnsmasq \
      /uk/org/thekelleys/dnsmasq \
      uk.org.thekelleys.dnsmasq.SetServers \
      uint32:

  # Clear dnsmasq's cache
  dbus-send --print-reply --system \
    --dest=uk.org.thekelleys.dnsmasq \
      /uk/org/thekelleys/dnsmasq \
      uk.org.thekelleys.dnsmasq.ClearCache
}
```

