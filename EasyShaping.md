*Easy shaping*

Connections with rate over 2Mbit will be automatically shaped to 2M/2M maximum rate.

- Add addresses of hosts where connection rate is over 2Mbit to address list "NetworkLoad >2Mbit"

```
/ip firewall filter
add chain=forward action=add-dst-to-address-list address-list="NetworkLoad >2Mbit" address-list-timeout=30m connection-rate=2M-100M

```
- Mangle all connections from the list with packet mark "shaping"
```
/ip firewall mangle
add chain=prerouting action=mark-packet new-packet-mark=shaping passthrough=no src-address-list="NetworkLoad >2Mbit" 
```
- Create simple queue
```
/queue simple
add name="shaping" packet-marks=shaping max-limit=2M/2M
```
