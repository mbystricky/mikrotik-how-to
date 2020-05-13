How to check internet connection when you have multiple gateways, but gateways are another routers (you're behind NAT)
---

Let's pretend we have setup like this:

          Mikrotik router
            --> ether1    -->   router1 (192.168.1.1)  --> internet
            --> ether2    -->   router2 (192.168.2.1)  --> internet
       LAN  <-- ether3

So we have two different gateways (WAN1, WAN2):

```
WAN1 (ether1) has gateway 192.168.1.1
WAN2 (ether2) has gateway 192.168.2.1
```

Our routing table is:

```
DST-ADDRESS     0.0.0.0     GATEWAY     192.168.1.1     DISTANCE    1
DST-ADDRESS     0.0.0.0     GATEWAY     192.168.2.1     DISTANCE    2
```
But when we have these gateways in our routing table and internet goes down on first gateway (192.168.1.1), we are not able to detect if connection goes down because of WAN IP (e.g. 192.168.1.1) is still alive.

To check if internet is online on interface WAN1 we have to ping internet on specified interface:

```
/ping 1.1.1.1 interval=1 count=5 interface=ether1
```

If it fails, we have to disable route targeted to WAN1, then WAN2 goes active.

```
:foreach i in=[/ip route find where gateway=192.168.1.1 disabled=no] do={/ip route set $i disabled=yes}
```

This script will search for gateway 192.168.1.1 and disables it.


But what if internet goes up on WAN1 ? If a route was disabled before, we are not able to check because of disabled route :)
So we have to do another routes specific for internet checking:

```
/ip route add gateway=192.168.1.1 routing-mark=WAN1_Check distance=1
/ip route add gateway=192.168.2.1 routing-mark=WAN2_Check distance=2
```

These routes will not be used be default from LAN because of routing mark, but we will be still able to ping internet from these routes when we use routing table withing ping commands:

```
/ping 1.1.1.1 interval=1 count=5 interface=ether1 routing-table=WAN1_Check
/ping 1.1.1.1 interval=1 count=5 interface=ether2 routing-table=WAN2_Check
```

So if internet goes up on WAN1 with specified routing-table we can freely enable route.

Scheduler script (for ether1):

```
:local PING_HOST "1.1.1.1"
:local PING_COUNT "20"
:local INTERFACE_NAME "ether1"
:local GATEWAY "192.168.1.1"
:local NAME "WAN1"
:local ROUTING_TABLE "WAN1_Check"

:if ([/ping $PING_HOST interval=1 count=$PING_COUNT interface=$INTERFACE_NAME routing-table=$ROUTING_TABLE] = 0) do={
   /log info message="$NAME Internet Check: DOWN"
   :foreach i in=[/ip route find where gateway=$GATEWAY routing-mark!="$ROUTING_TABLE" disabled=no] do={/ip route set $i disabled=yes}
} else {
   /log info message="$NAME Internet Check: UP"
   :foreach i in=[/ip route find where gateway=$GATEWAY routing-mark!="$ROUTING_TABLE" disabled=yes] do={/ip route set $i disabled=no}
}
```
