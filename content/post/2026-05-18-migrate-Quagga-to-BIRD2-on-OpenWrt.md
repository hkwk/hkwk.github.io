---
author: HKL
categories:
- Default
date: "2026-05-18T22:15:00+08:00"
slug: migrate-Quagga-to-BIRD2-on-OpenWrt
draft: false
tags:
- OpenWrt
- Routing
- Networking
title: "Migrating from Quagga (ripd) to BIRD2 on OpenWrt: A Practical Guide"
---

Routing daemons on embedded devices like OpenWrt have come a long way. For years, Quagga (and later FRR) was the go-to suite for running traditional dynamic routing protocols like RIP, OSPF, and BGP. However, as networks evolve, resource constraints and configuration elegance become paramount. 

If you are still running Quagga's `ripd` on OpenWrt, it is time to upgrade to **BIRD2 (Internet Routing Daemon)**. This blog post covers why you should migrate, a real-world transformation example involving Tinc VPN tunnels, and a step-by-step transition guide.

---

## Why Move to BIRD2?

While Quagga uses a Cisco-like CLI syntax (`vtysh`) that many network engineers find familiar, it introduces unnecessary complexity for modern Linux environments. Here is why BIRD2 shines on OpenWrt:

*   **Ultra-Lightweight Memory Footprint:** BIRD2 is notoriously efficient. It uses significantly less CPU and RAM than the Quagga/FRR ecosystem, making it perfect for resource-constrained OpenWrt routers.
*   **Single Daemon Architecture:** Quagga splits functionality across multiple daemons (`zebra` for core routing, `ripd` for RIP, `ospfd` for OSPF). BIRD2 handles everything inside a single process, eliminating inter-daemon communication overhead.
*   **Programmable Configuration:** BIRD2 drops the legacy CLI style in favor of a powerful, modern, curly-brace configuration language complete with logical statements (`if/then/else`) and robust route filtering.

---

## The Migration Example

Let’s look at a concrete scenario. Suppose you have an OpenWrt router acting as a node in a mesh network connected via **Tinc VPN (`tincn0`)** and serving a local LAN (`br-lan`).

### The Old Quagga Configuration (`/etc/quagga/ripd.conf`)
In Quagga, the configuration relied on VTY lines for local access security and network network-fuzzing statements:

```bird
password zebra
!
router rip
 network 10.193.111.0/24
 route 10.193.99.0/24
!
access-list vty permit 127.0.0.0/8
access-list vty deny any
!
line vty
 access-class vty

```

### The New Equivalent BIRD2 Configuration (`/etc/bird.conf`)

In BIRD2, there is no need for local VTY passwords because administration is safely handled via a local Unix Domain Socket (`/var/run/bird.ctl`).

Instead of network network-fuzzing statements, BIRD2 maps explicitly to kernel interfaces and uses an **Export Filter** to control exactly what routes get broadcasted:

```bird
# 1. Standard Production Log Levels
log syslog { info, warning, error, fatal };

# 2. Unique Router Identifier
router id 10.193.111.99;

# Core Protocol: Synchronizes BIRD routing table with the Linux Kernel
protocol kernel {
    ipv4 {
        import all;
        export all; # Push routes learned via RIP straight to OpenWrt kernel
    };
}

# Core Protocol: Monitors interface link states (Up/Down)
protocol device {
}

# Core Protocol: Imports local directly-connected interfaces into BIRD's memory
protocol direct {
    ipv4;
    interface "br-lan", "tincn0";
}

# RIP Dynamic Routing Protocol Instance
protocol rip my_rip {
    ipv4 {
        import all;    # Accept all RIP routes sent by neighbors
        export filter {
            # Equivalent to Quagga's 'network' and 'route' statements.
            # Only announce these specific local prefixes to neighbors.
            if net ~ [ 10.193.111.0/24, 10.193.99.0/24 ] then accept;
            reject;
        };
    };

    # Run RIPv2 Multicast over the Tinc VPN Interface
    interface "tincn0" {
        version 2;
        mode multicast;
        update time 30;
    };

    # Run RIPv2 Multicast over the Local LAN Interface
    interface "br-lan" {
        version 2;
        mode multicast;
        update time 30;
    };
}

```

---

## Key Configuration Differences Decoded

1. **Interface-Driven vs Network-Driven:** Quagga uses `network 10.193.111.0/24` to *implicitly* find which interface owns that subnet and start RIP on it. BIRD2 requires you to *explicitly* define the interface name block (`interface "tincn0"`). This prevents accidental route leaks on interfaces you didn't mean to touch.
2. **The Role of `protocol direct`:** To announce your local LAN segment (`10.193.99.0/24`) to your neighbors in BIRD2, you must use `protocol direct` to ingest that local link state into BIRD's internal tables, allowing the RIP export filter to pick it up and pass it along.

---

## Step-by-Step Deployment on OpenWrt

Ready to make the switch? Run these commands directly on your OpenWrt terminal:

### 1. Clean Up Quagga and Install BIRD2

Remove the old packages entirely to free up space and avoid socket conflicts, then install BIRD2 and its CLI controller:

```bash
opkg update
opkg remove quagga* frr*
opkg install bird2 birdc

```

### 2. Apply the Configuration

Paste the new configuration block into `/etc/bird.conf` using `vi` or `nano`. Ensure you update the `router id` and the `interface` names to match your device (`ip addr`).

### 3. Launch the Daemon

Stop any remaining legacy routing processes, enable BIRD2 to run on boot, and start the service:

```bash
killall -9 ripd zebra 2>/dev/null
/etc/init.d/bird enable
/etc/init.d/bird restart

```

---

## Verifying the Migration

BIRD2 comes with a remarkably snappy management tool called `birdc`. Use it to verify that everything went smoothly.

### Check Protocol Status

Verify that your protocols are up and routes are syncing:

```text
root@OpenWrt:~# birdc show protocols
BIRD 2.18 ready.
Name       Proto      Table      State  Since        Info
kernel1    Kernel     master4    up     22:15:23.102
device1    Device     ---        up     22:15:23.102
direct1    Direct     ---        up     22:15:23.102
my_rip     RIP        master4    up     22:15:23.102

```

### Inspect RIP Details

Check how many routes are being imported from your neighbors and exported out:

```text
root@OpenWrt:~# birdc show protocols all my_rip
...
    Routes:         4 imported, 2 exported, 4 preferred
...

```

Finally, running `ip route` in your system shell should show your dynamically learned routes seamlessly marked with `proto bird` instead of the old `proto zebra`.

## Conclusion

Migrating from Quagga to BIRD2 on OpenWrt modernizes your routing setup. You trade a clunky, multi-process legacy tool for a sleek, secure, programmatic, single-process engine—saving valuable system memory while gaining predictable routing control. Happy routing!

