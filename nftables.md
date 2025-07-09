# Nftables - basic examples

These examples assume you know the basics, such as:
* what is a filter
* what is a NAT
* whare are IPs and PORTs, destination and source
* basic firewall definitions, tcp vs icmp
* that ICMP is not just PING, but actually the very foundation of communication and should NOT be simple blocked regardless of ICMP type (_seriously!_)

First have a look at [nftables-filter.md](nftables-filter.md). This example shows how to use `nftables`, how to define and apply IP/PORT sets, and inbound/outbound firewall rules to accept, drop or reject packets.

For creating NAT rules, also have a look at [nftables-nat.md](nftables-nat.md). This example shows a compressed version of how NAT would be setup in `nftables`. This assumes you have read the [nftables-filter.md](nftables-filter.md) first and doesn't repeat certain lessons.
