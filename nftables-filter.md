# How nftables filter rules work - basic example

The following example explains how a ruleset can be created with a named table, without affecting other rulesets. Prerequisite is to have `nftables` package installed. This example has been tested to work on all supported Ubuntu and Rockylinux versions at the time of writing this document.

We will use a named table that we can delete and recreate. We will also use multiple definition files, split for readability. This is not a requirement and definitions and config can be split however we want (or even all included in a single nft config file).

Note you can call youer tables and chains however you want, there are no predefined names. The only thing that matters is the hooks that tell nftables when to execute the rulesets.

## Prep

We are going to store our rules in `/etc/gofw`

```bash
rm -rf /etc/gofw
mkdir -p /etc/gofw
```

## Create main rule

This creates the main ruleset. The design does the following:
1. delete the table called `gofw`
2. include 3 rules files
   * `sets.nft` where we will define our sets of IPs and ports if we want to
   * `in.nft` where we will define our inbound rules
   * `out.nft` where we will define our outbound rules
  
The table is defined just before being deleted for a simple reason: if this is the first time the rule is used, it will error saying that `gofw` doesn't exist when it tries to delete it. So, we make sure it exists before the rest of the config runs. This way it works first time and on subsequent runs, overriding and reloading the config each time.

```bash
cat <<'EOF' > /etc/gofw/main.nft
table inet gofw {
}
delete table inet gofw
include "/etc/gofw/sets.nft"
include "/etc/gofw/in.nft"
include "/etc/gofw/out.nft"
EOF
```

## Create IP and Port sets

Two example sets are created, one with IPs and one with ports, so they can be referenced in rules.

```bash
cat <<'EOF' > /etc/gofw/sets.nft
table inet gofw {
    set xdrdc2 {
        type ipv4_addr;
        elements = { 192.168.1.10, 192.168.1.42, 10.0.0.66 }
    }
    set asd_ports {
        type inet_service;
        elements = { 21, 23, 25, 445, 666 }
    }
}
EOF
```

## Create input rules

This example contains example of input rules to choose from. Many more are possible, these are just the common scenarios.

```bash
cat <<'EOF' > /etc/gofw/in.nft
table inet gofw {
    chain input {
        # hook this rule to run on input packets, at priority 0, default policy is accept
        type filter hook input priority 0; policy accept;

        # packets with state established, related will be accepted - only useful if default policy is drop
        ct state established,related accept

        # allow ports 80, 443
        tcp dport { 80, 443 } accept

        # drop packets from given IP with destination port 8080
        ip saddr 192.168.4.31 tcp dport 8080 drop

        # reject packets from given IP with destination port 8081, with a specific reject ICMP type
        ip saddr 192.168.4.31 tcp dport 8081 reject with icmp type port-unreachable

        # rate limit packets from specific IP with destination port 8082, up to 10 packets per second, then drop; allow a 5-packet burst
        ip saddr 192.168.4.31 tcp dport 8082 limit rate over 10/second burst 5 packets drop

        # rate limiting using numgen - from specific IP with destination port 8083; drop packets which are over 20 out of 100 - drop 20% randomly
        ip saddr 192.168.4.31 tcp dport 8083 numgen random mod 100 < 20 drop

        # rate limit src IP and dst PORT; rate limit up to a specific usage - 10MB/second, then drop
        ip saddr 192.168.4.31 tcp dport 8084 limit rate over 10 mbytes/second drop

        # if icmp type echo-request is received (ping), drop the packets; the counter is there to tell nft to count the number of packets dropped so we can see that in list ruleset command
        ip protocol icmp icmp type echo-request counter drop

        # drop all packets from IPs mentioned in the xdrdc2 set with one of the destination ports listed in the asd_ports set
        ip saddr @xdrdc2 tcp dport @asd_ports drop

        # drop connections to port 22 from all IPs
        tcp dport 22 drop
    }
}
EOF
```

## Create output rules

Output rules behave pretty much the same way as input rules. In these examples we are matching `sport` for source port instead of `dport` for destination port. In the same way `daddr` will match destination IP, while `saddr` matches source IP.

```bash
cat <<'EOF' > /etc/gofw/out.nft
table inet gofw {
    chain output {
        type filter hook output priority 0; policy accept;
        ct state established,related accept
        tcp sport { 80, 443 } accept
        tcp sport 22 drop
    }
}
EOF
```

## Apply the rules

```bash
# apply the rules from main.nft
nft -f /etc/gofw/main.nft

# list the rules in json format
nft --json list ruleset

# list the rules in human readable format
nft list ruleset
```

## Note on rule priorities

The rules have priorities. We are defining a default priority 0 in our example. One could create more tables and chains, with their own rulesets and priorities. The way this works is that the lower the priority the earlier the rule will be validated.

This means we can create a table, chain with hook priority -100 and it will be evaluated before the priority 0, in the same way hook with prio 100 will be executed after 0.
