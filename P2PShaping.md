**Peer-to-peer shaping**

*Intro:*
* Our network is 192.168.11.0/24, so if your is different, change it to yours
* This solution will detect local IP addresses by L7 filter and add them to address list "torrent_hosts"
* These hosts' connections will be marked as Torrent connection and mangled to create packet marks "Torrent"
* We create a simple queue rule to limit these connections to 128kbit/16kbit

*Solution:*
- Create L7 regular expression with name "torrents"
```
/ip firewall layer7-protocol
add name="torrents" regexp="^(\x13bittorrent protocol|azver\x01$|get /scrape\?info_hash=get /announce\?info_hash=|get /client/bitcomet/|GET /data\?fid=)|d1:ad2:id20:|\x08'7P\)[RP]"
```

- Create filter rule to add addresses to list "torrent_hosts" filtered by L7 with timeout of 2 minutes
```
/ip firewall filter
add chain=forward action=add-src-to-address-list layer7-protocol=torrents src-address=192.168.11.0/24 address-list=torrent_hosts address-list-timeout=2m
```
- Next, we mark connections identified by previously created address list and public ports 
```
/ip firewall mangle
add chain=prerouting action=mark-connection new-connection-mark=Torrent protocol=tcp src-address-list=torrent_hosts dst-port=!0-1024,8291,5900,5800,3389,14147,5222,59905
add chain=prerouting action=mark-connection new-connection-mark=Torrent protocol=udp src-address-list=torrent_hosts dst-port=!0-1024,8291,5900,5800,3389,14147,5222,59905
```
- With connections being marked we have to mark packets by them, so these connections can be processed by our simple queue
```
add chain=prerouting action=mark-packet new-packet-mark=Torrent passthrough=no connection-mark=Torrent
```

- Create simple queue
```
/queue simple
add name="P2P" packet-marks=Torrent max-limit=16k/128k
```