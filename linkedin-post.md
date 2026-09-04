I built a Site-to-Site VPN between my home Ubuntu server and AWS this week.

The tunnel came up in about 20 minutes. Getting an actual ping through it took a lot longer — and that gap is where the real learning happened.

Here's what the AWS console showed: both tunnels green, status "UP," provisioning "Available." Every box checked. And 100% packet loss on every single ping.

Two separate bugs were hiding behind that green checkmark:

𝗕𝘂𝗴 #𝟭 — IPsec doesn't care about your routing table.
My on-prem box's real IP was 192.168.1.39. My tunnel was configured to protect traffic from 172.16.1.0/24. IPsec matches packets against traffic selectors (its own policy database), not the kernel's routes — so every ping I sent just walked out my normal home gateway and vanished. No error. No warning. Just silence.
Fix: gave the box a second IP inside the right subnet, and the traffic finally matched the policy.

𝗕𝘂𝗴 #𝟮 — a route table doesn't help you if your subnet isn't using it.
SSH into the EC2 instance was also failing. Same time, seemingly unrelated. Turned out my subnet had no explicit route table association — so it was silently falling back to the VPC's *main* route table, which had my VPN route but zero path to the actual internet. One missing "0.0.0.0/0 → Internet Gateway" line was blocking SSH, EC2 Instance Connect, and the last leg of my VPN traffic, all at once.

The tell, in both cases, wasn't the AWS console. It was:

swanctl --list-sas → showed 0 bytes transferred even on an "ESTABLISHED" tunnel
ip xfrm state show → showed my outbound packets leaving with real timestamps, but the inbound SA had never received a single one

That one distinction — "the tunnel is negotiated" vs. "traffic is actually flowing through it" — is the whole lesson. A green status light in any cloud console tells you authentication worked. It tells you nothing about whether your packets are getting anywhere.

Wrote up the full command-by-command version (including every diagnostic command and its actual output) if anyone's building something similar or wants the troubleshooting flow for their own toolkit.

#AWS #Networking #VPN #IPsec #CloudComputing #Linux #SysAdmin #strongSwan #HandsOnLearning
