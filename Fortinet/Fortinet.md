# Firewall Issue: Inability to Block Incoming (WAN to LAN) Connection Despite Deny Policy

# Description #

Encountering a scenario where the firewall fails to block incoming WAN to LAN connections for a specific IP, even with a deny policy in place. The setup involves an inbound NAT for accessing an internal web server from an external network, and the goal is to prevent one specific external IP from accessing it. Despite configuring a deny policy with the source set to the IP of the external client, the firewall policy is not triggering as expected


# Solution VIP
To address this issue, follow the steps below:

 
Step 1:
Edit the denied firewall policy from the CLI.

Step 2:
Run the following command to enable the match-vip option:
```
set match-vip enable
```

Note: This option is applicable only for DENY policies and is available since version 6.4.3. It is no longer supported for ACCEPT policies.

Step 3:
After enabling the match-vip option, DNATed packets that do not match a VIP policy will be matched with the general policy. This allows for explicit dropping and logging of the undesired traffic.

Alternative Solution:
As an alternative approach:

Always configure the deny policy with the destination address set to the VIP for which traffic is denied, instead of using 'All'. This ensures that the deny policy explicitly targets the specified VIP, providing a more precise control over the blocked traffic.

Implementing either of these solutions should resolve the issue of the firewall not blocking incoming connections from the specified external IP despite having a deny policy in place.
