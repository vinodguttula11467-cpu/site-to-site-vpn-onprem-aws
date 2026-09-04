---
title: "Building a Site-to-Site VPN Between an On-Prem Ubuntu Server and AWS (A Real Walkthrough, Bugs Included)"
description: "A hands-on, command-by-command guide to building an IPsec Site-to-Site VPN between an on-prem Ubuntu VM and an AWS VPC using strongSwan — including the two real bugs that broke connectivity and exactly how they were diagnosed and fixed."
date: 2026-09-04
tags: [aws, networking, vpn, ipsec, strongswan, cloud, sysadmin]
---

# Building a Site-to-Site VPN Between an On-Prem Ubuntu Server and AWS

*A real, command-by-command walkthrough — including the two bugs that broke connectivity, how they were diagnosed, and how they were fixed.*

Most VPN tutorials show you the happy path: run these ten commands, tunnel comes up, done. This one doesn't skip the part where the tunnel shows **UP** in the AWS console and you *still* can't ping anything. That's the part that actually teaches you how IPsec works — so it's included here in full, with the real diagnostic commands and real output.

By the end you'll have a working IPsec Site-to-Site VPN connecting a self-managed Ubuntu server to an AWS VPC, plus a mental model for debugging the next one when it inevitably breaks in a slightly different way.

---

## Architecture

![Architecture diagram showing the on-prem Ubuntu VM connected to an AWS VPC over two IPsec tunnels, with IP addresses and subnet details labeled](./assets/topology.png)

**The setup:**

- An Ubuntu server sitting behind a home router (i.e., behind NAT — this matters later)
- An AWS VPC with a Virtual Private Gateway, an EC2 test instance, and an Internet Gateway
- Two redundant IPsec tunnels (AWS always gives you two, for high availability)
- strongSwan, running in its modern **swanctl** (vici) configuration mode — not the legacy `ipsec.conf` format most tutorials still show

| Component | Value used in this walkthrough |
|---|---|
| On-prem public IP | `45.252.75.142` |
| On-prem LAN IP (behind NAT) | `192.168.1.39/24` |
| On-prem tunnel subnet | `172.16.1.0/24` |
| AWS VPC CIDR | `10.0.0.0/16` |
| AWS EC2 subnet | `10.0.1.0/24` |
| EC2 instance private / public IP | `10.0.1.244` / `44.206.236.3` |
| VPC ID | `vpc-06cfe29168305f509` |
| Internet Gateway ID | `igw-0334447a4eacd3aba` |
| Customer Gateway ID | `cgw-09f0bf77bf7057712` |
| VPN Gateway ID | `vgw-0e3dbecaf66b8ffcc` |
| VPN Connection ID | `vpn-08941301cb8386c08` |
| Route Table ID | `rtb-01a9c8b2881f60e2b` |

> Swap these for your own values throughout — they're kept concrete here (rather than replaced with placeholders) because seeing *real* AWS resource IDs next to real command output is what makes a walkthrough like this actually useful to follow along with.

---

## Part 1 — Prepare the On-Prem Ubuntu VM

Install strongSwan and turn on IP forwarding, since this box needs to route traffic between two networks, not just handle traffic addressed to itself.

```bash
sudo apt update && sudo apt install -y strongswan strongswan-pki libcharon-extra-plugins
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

Find your public IP — you'll register this with AWS as your Customer Gateway:

```bash
curl ifconfig.me
```

---

## Part 2 — Build the AWS Side

### Create the VPC, subnet, and Internet Gateway

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]'

aws ec2 create-subnet --vpc-id vpc-06cfe29168305f509 --cidr-block 10.0.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-subnet}]'

aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]'

aws ec2 attach-internet-gateway --vpc-id vpc-06cfe29168305f509 \
  --internet-gateway-id igw-0334447a4eacd3aba
```

Launch a small EC2 instance (t3.micro) in this subnet to act as your test target, and confirm it's running:

![AWS console showing one running EC2 t3.micro instance named Multicloud-Ec2-instance with status checks passed](./assets/ec2-instance-created.png)

### Register your on-prem VM as a Customer Gateway

```bash
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 45.252.75.142 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=lab-cgw}]'
```

This creates an AWS object representing *your* end of the tunnel. `--bgp-asn` is required even though this walkthrough uses static routing, not BGP.

### Create and attach the Virtual Private Gateway

```bash
aws ec2 create-vpn-gateway --type ipsec.1 --amazon-side-asn 64512 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=lab-vgw}]'

aws ec2 attach-vpn-gateway --vpc-id vpc-06cfe29168305f509 \
  --vpn-gateway-id vgw-0e3dbecaf66b8ffcc
```

### Create the VPN connection and download its configuration

```bash
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-09f0bf77bf7057712 \
  --vpn-gateway-id vgw-0e3dbecaf66b8ffcc \
  --options StaticRoutesOnly=true

aws ec2 create-vpn-connection-route \
  --vpn-connection-id vpn-08941301cb8386c08 \
  --destination-cidr-block 172.16.1.0/24

aws ec2 describe-vpn-connections --vpn-connection-id vpn-08941301cb8386c08 \
  --query "VpnConnections[0].CustomerGatewayConfiguration" --output text > vpn-config.xml
```

`vpn-config.xml` now contains **two** tunnel definitions — AWS always provisions a redundant pair. Each one gives you an outside IP and a pre-shared key:

| Tunnel | AWS outside IP | Purpose |
|---|---|---|
| 1 | `13.216.107.39` | Primary |
| 2 | `100.63.37.136` | Standby / failover |

> **Security note:** this XML file contains your real pre-shared keys and your on-prem public IP in plaintext. Treat it like a credentials file — `chmod 600 vpn-config.xml`, and never commit it to a repository.

### Update the route table

```bash
aws ec2 create-route \
  --route-table-id rtb-01a9c8b2881f60e2b \
  --destination-cidr-block 172.16.1.0/24 \
  --gateway-id vgw-0e3dbecaf66b8ffcc

aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-01a9c8b2881f60e2b \
  --gateway-id vgw-0e3dbecaf66b8ffcc
```

Even with the tunnel established, AWS won't route traffic through it unless the VPC's route table explicitly points your on-prem CIDR at the VGW. **This step is the single most commonly forgotten one** — keep it in mind for the troubleshooting section later.

---

## Part 3 — Configure strongSwan (swanctl / vici)

Modern strongSwan on Ubuntu uses **swanctl**, not the classic `ipsec.conf`. Connections live as individual files under `/etc/swanctl/conf.d/`, auto-included by the main config.

### Tunnel 1 connection definition

```bash
sudo nano /etc/swanctl/conf.d/aws-tunnel1.conf
```

```
connections {
    aws-tunnel1 {
        version = 1
        remote_addrs = 13.216.107.39
        proposals = aes128-sha1-modp1024
        local {
            auth = psk
            id = 45.252.75.142
        }
        remote {
            auth = psk
            id = 13.216.107.39
        }
        children {
            aws-tunnel1 {
                local_ts = 172.16.1.0/24
                remote_ts = 10.0.1.0/24
                esp_proposals = aes128-sha1-modp1024
                start_action = trap
                dpd_action = restart
                life_time = 1h
                rekey_time = 55m
            }
        }
    }
}
```

The proposal (`aes128-sha1-modp1024`) matches exactly what AWS specified in `vpn-config.xml` — AES-128-CBC encryption, SHA-1 integrity, Diffie-Hellman Group 2. `local_ts`/`remote_ts` are the **traffic selectors** — this is the field that caused the first real bug below, so keep it in mind.

### Secrets file

```
secrets {
    ike-aws-tunnel1 {
        id-1 = 45.252.75.142
        id-2 = 13.216.107.39
        secret = "<PSK_FROM_VPN_CONFIG_XML>"
    }
}
```

### Load and bring the tunnel up

```bash
sudo swanctl --load-all
sudo swanctl --initiate --child aws-tunnel1
sudo swanctl --list-sas
```

### Add the second tunnel (for redundancy)

Same structure, pointed at AWS's second outside IP:

```bash
sudo nano /etc/swanctl/conf.d/aws-tunnel2.conf
```

```
connections {
    aws-tunnel2 {
        version = 1
        remote_addrs = 100.63.37.136
        proposals = aes128-sha1-modp1024
        local {
            auth = psk
            id = 45.252.75.142
        }
        remote {
            auth = psk
            id = 100.63.37.136
        }
        children {
            aws-tunnel2 {
                local_ts = 172.16.1.0/24
                remote_ts = 10.0.1.0/24
                esp_proposals = aes128-sha1-modp1024
                start_action = trap
                dpd_action = restart
                life_time = 1h
                rekey_time = 55m
            }
        }
    }
}
```

Add the matching secret block (in the same `secrets { }` container as tunnel 1), then reload:

```bash
sudo chmod 600 /etc/swanctl/conf.d/aws-tunnel2.conf
sudo swanctl --load-all
sudo swanctl --initiate --child aws-tunnel2
```

**Result — both tunnels UP in the AWS console:**

![AWS console showing both VPN Tunnel 1 and Tunnel 2 with status Up and provisioning status Available](./assets/vpn-tunnels-up.png)

At this point it looks finished. It wasn't.

---

## Part 4 — "The Tunnel Is Up, But I Can't Ping Anything"

This is the part most guides skip, and the part that actually matters. The tunnel showing `UP` only confirms that **IKE negotiation and authentication** succeeded — it says nothing about whether real traffic can actually flow. Two separate, unrelated bugs were hiding behind that green checkmark.

![Troubleshooting decision flow for diagnosing a VPN tunnel that shows Up but passes no traffic](./assets/troubleshooting-flow.png)

### Symptom

```bash
root@ubuntu-server:~# ping 10.0.1.244
PING 10.0.1.244 (10.0.1.244) 56(84) bytes of data.
^C
--- 10.0.1.244 ping statistics ---
68 packets transmitted, 0 received, 100% packet loss, time 68609ms
```

The AWS console showed the security group already allowed all ICMP from `0.0.0.0/0`, so that was ruled out immediately. Time to look deeper than the console.

### Step 1 — Check whether traffic is actually entering the tunnel

```bash
sudo swanctl --list-sas
```

```
aws-tunnel1: #1, ESTABLISHED, IKEv1, ...
  aws-tunnel1: #5, reqid 1, INSTALLED, TUNNEL-in-UDP, ESP:AES_CBC-128/HMAC_SHA1_96/MODP_1024
    installed 427s ago, rekeying in 2776s, expires in 3174s
    in  c985c83f,      0 bytes,     0 packets
    out c43b0083,      0 bytes,     0 packets
    local  172.16.1.0/24
    remote 10.0.1.0/24
```

**`0 bytes, 0 packets` in both directions** — the tunnel was healthy at the IKE/ESP level, but *nothing had ever actually used it*. That's a completely different problem from "the tunnel is down," and it's a distinction most people miss.

### Step 2 — Find out why nothing enters the tunnel

```bash
ip addr show
```

```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.39/24 metric 100 brd 192.168.1.255 scope global dynamic enp0s3
```

```bash
ip route get 10.0.1.244
```

```
10.0.1.244 via 192.168.1.1 dev enp0s3 src 192.168.1.39 uid 0
```

**There it is.** The on-prem VM's real address is `192.168.1.39` — but the tunnel's `local_ts` is scoped to `172.16.1.0/24`. IPsec doesn't consult the kernel routing table to decide what gets encrypted; it uses its own **Security Policy Database (SPD)**, and a packet only gets encrypted if its *source* address falls inside `local_ts`. Since `192.168.1.39` is outside `172.16.1.0/24`, every ping was quietly leaving over the normal home-router route instead of going anywhere near the tunnel — matching the symptom exactly: no error, just silence.

### Root Cause #1

> The VM's real network identity didn't match the IPsec traffic selector it was configured to protect.

### Fix #1 — give the VM a routable identity inside the tunnel's subnet

```bash
sudo ip addr add 172.16.1.10/24 dev enp0s3
```

```bash
ping -I 172.16.1.10 10.0.1.244
```

This got traffic flowing *outbound* — confirmed by watching the SA counters change:

```bash
sudo swanctl --list-sas
```
```
in  c985c83f, ... anti-replay context: seq 0x0, oseq 0x29     <- outbound sequence now incrementing
```

But the ping still didn't come back. On to bug #2.

### Step 3 — Confirm what's actually happening at the packet level

```bash
sudo tcpdump -i enp0s3 udp port 4500 -c 10 -n
```

(Port 4500, not raw ESP — this tunnel negotiated **NAT-T**, since strongSwan detected the VM sits behind a home router. All ESP traffic gets wrapped inside UDP/4500.)

```bash
sudo ip xfrm state show
```

```
src 192.168.1.39 dst 13.216.107.39
    proto esp spi 0xc43b0083 reqid 1 mode tunnel
    ...
    lastused 2026-09-04 10:55:38
    anti-replay context: seq 0x0, oseq 0x29
    dir out

src 13.216.107.39 dst 192.168.1.39
    proto esp spi 0xc985c83f reqid 1 mode tunnel
    ...
    anti-replay context: seq 0x0, oseq 0x0
    dir in
```

This is the key evidence: the **outbound** SA had a `lastused` timestamp and a nonzero sequence number — the ping genuinely left on-prem, correctly encrypted, and reached AWS. The **inbound** SA had never received a single packet. So the problem wasn't the tunnel at all anymore — it had moved to the AWS/EC2 side.

### Step 4 — Isolate the AWS side

Since regular SSH to the instance's public IP was *also* failing, both symptoms were checked together. EC2 Instance Connect (browser-based SSH, no key needed) was tried first to get a shell without depending on external SSH — and it failed with the same generic error, which was itself a clue: both external SSH and Instance Connect ultimately arrive over the same path from outside the VPC.

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-01c6c33fa4fff3fb7" \
  --query "RouteTables[].{RTId:RouteTableId,Routes:Routes}" --output json
```

```json
[]
```

Empty — meaning this subnet had **no explicit route table association** and was silently falling back to the VPC's main route table:

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-06cfe29168305f509" "Name=association.main,Values=true" \
  --query "RouteTables[].{RTId:RouteTableId,Routes:Routes}" --output json
```

```json
[
  {
    "RTId": "rtb-01a9c8b2881f60e2b",
    "Routes": [
      { "DestinationCidrBlock": "172.16.1.0/24", "GatewayId": "vgw-0e3dbecaf66b8ffcc", "State": "active" },
      { "DestinationCidrBlock": "10.0.0.0/16",   "GatewayId": "local",                 "State": "active" }
    ]
  }
]
```

### Root Cause #2

> The subnet's route table had a route back to on-prem, but **no `0.0.0.0/0 → Internet Gateway` route at all** — so the instance had no path to/from the public internet, which blocked SSH, Instance Connect, and (as it turned out) the final leg of return VPN traffic.

### Fix #2 — add the missing internet route

```bash
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=vpc-06cfe29168305f509" \
  --query "InternetGateways[].InternetGatewayId" --output text
# igw-0334447a4eacd3aba

aws ec2 create-route \
  --route-table-id rtb-01a9c8b2881f60e2b \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0334447a4eacd3aba
```

---

## Part 5 — Verification

With both root causes fixed, every direction was retested from scratch.

**SSH now succeeds**, and from inside the instance, pinging back to the tunnel address works cleanly:

![Terminal output showing a successful ping from the AWS EC2 instance to the on-prem VM with 0 percent packet loss](./assets/ping-aws-to-onprem.png)

**And the original direction — on-prem to AWS — now succeeds too:**

![Terminal output showing a successful ping from the on-prem Ubuntu VM to the AWS EC2 instance with 0 percent packet loss](./assets/ping-onprem-to-aws.png)

```
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 255.356/267.708/274.073/8.735 ms
```

Full bidirectional connectivity, 0% packet loss both ways. The Site-to-Site VPN is genuinely working end-to-end.

---

## Lessons Learned

1. **"Tunnel UP" only proves IKE/ESP negotiation succeeded — it proves nothing about whether traffic can actually flow.** Always check the SA byte/packet counters (`swanctl --list-sas`) before assuming a tunnel works.
2. **IPsec traffic selectors are matched against the packet's actual source/destination IP, not the kernel's routing table.** If your local machine's real IP isn't inside `local_ts`, nothing you send will ever enter the tunnel — silently.
3. **A subnet inherits the VPC's main route table if it isn't explicitly associated with one.** Don't assume "I set up a route table" means the subnet you care about is using it — always verify with `Name=association.subnet-id`, not just `Name=vpc-id`.
4. **When two symptoms look unrelated (SSH failing, ping failing) but appear at the same time, check for a shared root cause first** — here, the missing IGW route explained both.
5. **`ip xfrm state show`'s `lastused` timestamp and sequence counters are the ground truth** for "did this packet actually get encrypted and sent," far more reliable than inferring from ping output alone.

---

## Cleanup

```bash
aws ec2 delete-vpn-connection --vpn-connection-id vpn-08941301cb8386c08
aws ec2 detach-vpn-gateway --vpn-gateway-id vgw-0e3dbecaf66b8ffcc --vpc-id vpc-06cfe29168305f509
aws ec2 delete-vpn-gateway --vpn-gateway-id vgw-0e3dbecaf66b8ffcc
aws ec2 delete-customer-gateway --customer-gateway-id cgw-09f0bf77bf7057712
aws ec2 terminate-instances --instance-ids i-0d99159c65b4df910
aws ec2 delete-vpc --vpc-id vpc-06cfe29168305f509
```

AWS VPN connections don't carry an hourly charge the way a VPN Gateway on Azure does, but there's no reason to leave test infrastructure running — tear it down once you're done experimenting.

---

## Reference: Full Diagnostic Command Cheat Sheet

| Question | Command |
|---|---|
| Is the IKE/ESP session up? | `sudo swanctl --list-sas` |
| Is traffic actually flowing (bytes/packets)? | `sudo swanctl --list-sas` (check `in`/`out` counters) |
| What's my real interface IP? | `ip addr show` |
| Will this packet use the tunnel or the default route? | `ip route get <dest-ip>` |
| What does the kernel's IPsec policy table look like? | `sudo ip xfrm policy show` |
| What does the kernel's active SA state look like? | `sudo ip xfrm state show` |
| Is real (NAT-T) traffic leaving the NIC? | `sudo tcpdump -i <iface> udp port 4500 -n` |
| Does the VPC route table send return traffic through the VGW? | `aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"` |
| Does *this specific subnet* use that route table? | `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"` |
| Does the subnet have internet access at all? | look for `0.0.0.0/0 → igw-...` in the route list above |

---

*Questions or found a different failure mode? This walkthrough reflects one real debugging session end-to-end — your specific environment (router, AMI, region) may surface a different third bug. The diagnostic order above (IKE → SA counters → traffic selectors → kernel XFRM state → AWS routing) generalizes well regardless of what breaks.*
