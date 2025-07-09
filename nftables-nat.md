# Basic NAT with nftables

This example assumes you have read and understood [nftables-filter.md](nftables-filter.md) first.

The below example shows a basic NAT ruleset. This can be split into multiple config files and applied just like the filter ruleset.

## Basic example

```nftables
# define if doesn't exist - so we can always run this with the delete
table ip gonat {
}

# delete existing table if it exists
delete table ip gonat

# define the new table
table ip gonat {
    # this chain will be used for prerouting
    chain prerouting {
        # hook this chain in nft to prerouting
        type nat hook prerouting priority dstnat;

        # packets incoming for 182.0.2.1:8080, DNAT to 10.0.0.42:8080
        ip daddr 192.0.2.1 tcp dport 8080 dnat to 10.0.0.42:8080

        # similar to the above rule, but ONLY if the packets come in via eth0
        iifname "eth0" ip daddr 192.0.2.1 tcp dport 8081 dnat to 10.0.0.42:8081
    }

    # this chain will be used for postrouting
    chain postrouting {
        # hook this chain to postrouting
        type nat hook postrouting priority srcnat;

        # required rule, packets coming out of eth0, which have a source IP of 10.0.0.0/24 (our NAT), rename to source IP 192.0.2.1 - we need to masquerade on our way out of the NAT
        ip saddr 10.0.0.0/24 oifname "eth0" snat to 192.0.2.1
    }
}
```

### Here's the above, without comments for readability:

```nftables
table ip gonat {
}
delete table ip gonat

table ip gonat {
    chain prerouting {
        type nat hook prerouting priority dstnat;
        ip daddr 192.0.2.1 tcp dport 8080 dnat to 10.0.0.42:8080
        iifname "eth0" ip daddr 192.0.2.1 tcp dport 8081 dnat to 10.0.0.42:8081
    }

    chain postrouting {
        type nat hook postrouting priority srcnat;
        ip saddr 10.0.0.0/24 oifname "eth0" snat to 192.0.2.1
    }
}
```

## Masq

Instead of specifying SNAT, you can use MASQUERADE. This will automatically slap the interface IP on packets on their way out instead of manually specifying the IP to apply. Useful for dynamic IPs that change.

```
ip saddr 10.0.0.0/24 oifname "eth0" masquerade
```

## Filter what to masq/snat

If you want some ports to not be masqueraded, or SNAT, you can filter like so:

```
ip saddr 10.0.0.0/24 oifname "eth0" tcp dport != { 22, 443 } masquerade
```
