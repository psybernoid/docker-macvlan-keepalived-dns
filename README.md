# Running Multiple AdguardHome DNS Servers Across Different Docker Hosts, Keeping them in sync and presenting them all behind a single IP address.

## Why?
I wanted to have multiple DNS servers, for redundancy. Simply put, if one DNS server goes down - for maintenance, system crash or whatever, I needed to be sure other network clients could resolve DNS without issue.
> But doesn't DNS already have a failover because most DHCP servers allow you to define at least 2?

Yes. That's true. Most DHCP servers allow you put in at least a primary & secondary DNS server. If that's fine with you, great. It wasn't fine for me.
The problem with specifying multiple DNS servers in a DHCP scope, for me is that it's not an exact science. You might think that the PRIMARY & SECONDARY, or even the ordering of the DNS servers means something. It does not. Put simply, if you have 2 DNS servers defined in your scope, then it's a 50/50 chance which one you'll hit.

Which is fine, but it's less fine when you're running something like AdguardHome or Pi-Hole and your significant other's breathing down your neck because some website isn't working and you need to troubleshoot why it's being blocked. It's better for all involved if you know on which DNS server you're looking.

## How?
In order to achieve this, we need a minimum for 4 things:
1. [The AdguardHome Docker Container](https://github.com/AdguardTeam/AdGuardHome/wiki/Docker)
2. [The Keepalived Docker Container](https://github.com/osixia/docker-keepalived)
3. [The AdguardHome-Sync Docker Container](https://github.com/bakito/adguardhome-sync)
4. Unused IP addresses for all of the above


Let's get started.

It's a good idea to reference the AdguardHome documentation, but I'll presume you have and are familiar with using docker compose. Similarly, you should be familiar with macvlan docker networking too.

For this guide, we'll be using 4 IPs.
1 IP for each of the compose stacks running AdguardHome & KeepaliveD
1 (virtual) IP for the DNS server
1 IP for the AdguardHome Sync tool - this is optional, you could just use host networking for this one

So, we'll use the following

10.15.0.180  - virtual DNS IP
10.15.0.181  - AdguardHome/KeepaliveD host1
10.15.0.182  - AdguardHome/KeepaliveD host2
10.15.0.10   - AdguardHome-Sync

On your docker hosts, create a macvlan network with a name of your choosing. I'm using vlan15 here. Of course, the interface name with depend on your hardware, and the subnet will most likely be different.

`docker network create --driver macvlan -o parent=eth2.15 --subnet=10.15.0.0/24 --gateway=10.15.0.254 vlan15`

This will create a docker network that will persist across reboots. It's probably best you check the [documentation](https://docs.docker.com/engine/network/drivers/macvlan/) instead of blindly following what I type. The above command will most likely not work for you.

Now we have the macvlan created on both docker hosts, we need to deploy KeepaliveD and ensure everything's working. Here's the compose file:
```
services:
  keepalived_dns1:
    image: osixia/keepalived:2.0.20
    container_name: keepalived_dns1
    restart: unless-stopped
    networks:
      vlan15:
        ipv4_address: 10.15.0.181
    cap_add:
      - NET_ADMIN
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 80:80/tcp
      - 443:443/tcp
      - 3000:3000/tcp
      - 853:853/tcp
      - 853:853/udp
      - 5443:5443/tcp
      - 5443:5443/udp
      - 6060:6060/tcp
    environment:
      - KEEPALIVED_INTERFACE=eth0
      - KEEPALIVED_PRIORITY=100
      - KEEPALIVED_ROUTER_ID=53
      - KEEPALIVED_UNICAST_PEERS=10.15.0.182
      - KEEPALIVED_VIRTUAL_IPS=10.15.0.180
      - KEEPALIVED_STATE=MASTER
      - KEEPALIVED_PASSWORD=put_something_random_here
networks:
  vlan15:
    external: true
```

A quick rundown of what this is going to do:
1. It'll create a keepalived instance on the IP of 10.15.0.181
2. It'll expose all the ports required for AdguardHome (minus those required for ADH's DHCP server)
3. It'll set the priority of this KeepaliveD instance to 100 (higher number = higher priority)
4. It'll set a router ID (default is 51) to 53, because DNS. (the number is just a number. It means little else)
5. It'll look for 10.15.0.182 as a peer (you can add more IPs here as required)
6. It'll set the virtual IP to 10.15.0.180
7. It'll define itself as the master instance

Change put_something_random_here to whatever you want. This will also need to be set the same on any other node too.

Now, before we run docker compose up -d, let's do something. Get 2 terminals running and set a persistent ping on 10.15.0.180 & 10.15.0.181

Once they're running, execute `docker compose up -d`

Give it a moment and you'll notice both IPs are responding to ping.

Now, on your second host, run the very similar docker-compose.yaml

```
services:
  keepalived_dns2:
    image: osixia/keepalived:2.0.20
    container_name: keepalived_dns2
    restart: unless-stopped
    networks:
      vlan15:
        ipv4_address: 10.15.0.182
    cap_add:
      - NET_ADMIN
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 80:80/tcp
      - 443:443/tcp
      - 3000:3000/tcp
      - 853:853/tcp
      - 853:853/udp
      - 5443:5443/tcp
      - 5443:5443/udp
      - 6060:6060/tcp
    environment:
      - KEEPALIVED_INTERFACE=eth0
      - KEEPALIVED_PRIORITY=90
      - KEEPALIVED_ROUTER_ID=53
      - KEEPALIVED_UNICAST_PEERS=10.15.0.181
      - KEEPALIVED_VIRTUAL_IPS=10.15.0.180
      - KEEPALIVED_STATE=BACKUP
      - KEEPALIVED_PASSWORD=put_something_random_here
networks:
  vlan15:
    external: true
```

Before executing this, open yet another terminal and get a persistent ping up on 10.15.0.182, then run docker compose up -d

Note that .182 will shortly respond to ping. Keep all your pings going and then move to your fisrt host.

On the first host, execute `docker compose down` note that 10.15.0.181 will cease responding to ping, but 10.15.0.180 is still responding.

Congratulations! you're pretty much done.

All you need to do now is deploy the adguardhome containers inside the compose stacks. So bring both thse hosts down and edit your docker.compose.yaml files accordingly.

Host1:

```
services:
  keepalived_dns1:
    image: osixia/keepalived:2.0.20
    container_name: keepalived_dns1
    restart: unless-stopped
    networks:
      vlan15:
        ipv4_address: 10.15.0.181
    cap_add:
      - NET_ADMIN
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 80:80/tcp
      - 443:443/tcp
      - 3000:3000/tcp
      - 853:853/tcp
      - 853:853/udp
      - 5443:5443/tcp
      - 5443:5443/udp
      - 6060:6060/tcp
    environment:
      - KEEPALIVED_INTERFACE=eth0
      - KEEPALIVED_PRIORITY=100
      - KEEPALIVED_ROUTER_ID=53
      - KEEPALIVED_UNICAST_PEERS=10.15.0.182
      - KEEPALIVED_VIRTUAL_IPS=10.15.0.180
      - KEEPALIVED_STATE=MASTER
      - KEEPALIVED_PASSWORD=put_something_random_here
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    restart: unless-stopped
    network_mode: service:keepalived_dns1
    depends_on:
      - keepalived_dns1
    volumes:
      - /share/Docker/adguardhome/conf:/opt/adguardhome/conf
      - /share/Docker/adguardhome/conf:/opt/adguardhome/work
networks:
  vlan15:
    external: true
```

Host2:

```
services:
  keepalived_dns2:
    image: osixia/keepalived:2.0.20
    container_name: keepalived_dns2
    restart: unless-stopped
    networks:
      vlan15:
        ipv4_address: 10.15.0.182
    cap_add:
      - NET_ADMIN
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 80:80/tcp
      - 443:443/tcp
      - 3000:3000/tcp
      - 853:853/tcp
      - 853:853/udp
      - 5443:5443/tcp
      - 5443:5443/udp
      - 6060:6060/tcp
    environment:
      - KEEPALIVED_INTERFACE=eth0
      - KEEPALIVED_PRIORITY=90
      - KEEPALIVED_ROUTER_ID=53
      - KEEPALIVED_UNICAST_PEERS=10.15.0.181
      - KEEPALIVED_VIRTUAL_IPS=10.15.0.180
      - KEEPALIVED_STATE=BACKUP
      - KEEPALIVED_PASSWORD=put_something_random_here
  adguardhome:
    image: adguard/adguardhome
    container_name: adguardhome
    restart: unless-stopped
    network_mode: service:keepalived_dns2
    depends_on:
      - keepalived_dns2
    volumes:
      - /share/Docker/adguardhome/conf:/opt/adguardhome/conf
      - /share/Docker/adguardhome/work:/opt/adguardhome/work
networks:
  vlan15:
    external: true
```

And spin them both up.

Log onto both hosts with http://ip:3000 so, for example http://10.15.0.181:3000 and do the initial config of setting up the admin accounts, do the same for the second instance.
Note, which you could log onto http://10.15.0.180:3000 you'll only be configuring the master instance.

Once you've setup the admin account on both instances, log onto the primary instance only and configure it all to your liking - blocklists, upstreams, reverse lookups. All that good stuff.

Once you're done there, you can then deploy adguardhome-sync and ensure that the config from the master instance replicated to the backup instance.

```
services:
  adguardhome-sync:
    image: ghcr.io/bakito/adguardhome-sync
    container_name: adguardhome-sync
    restart: unless-stopped
    command: run
    networks:
      vlan15:
        ipv4_address: 10.15.0.10
    environment:
      LOG_LEVEL: info
      ORIGIN_URL: http://10.15.0.181
      ORIGIN_USERNAME: admin
      ORIGIN_PASSWORD: super_secret_password
      REPLICA1_URL: http://10.15.0.182
      REPLICA1_USERNAME: admin
      REPLICA1_PASSWORD: super_secret_password
      CRON: "*/10 * * * *"
      RUN_ON_START: "true"
      API_PORT: 8080
      API_METRICS_ENABLED: "true"
      API_USERNAME: api
      API_PASSWORD: api.password
      FEATURES_GENERAL_SETTINGS: "true"
      FEATURES_QUERY_LOG_CONFIG: "true"
      FEATURES_STATS_CONFIG: "true"
      FEATURES_CLIENT_SETTINGS: "true"
      FEATURES_SERVICES: "true"
      FEATURES_FILTERS: "true"
      FEATURES_DHCP_SERVER_CONFIG: "false"
      FEATURES_DHCP_STATIC_LEASES: "false"
      FEATURES_DNS_SERVER_CONFIG: "true"
      FEATURES_DNS_ACCESS_LISTS: "true"
      FEATURES_DNS_REWRITES: "true"
      FEATURES_THEME: "true"
networks:
  vlan15:
    external: true
```

Just spin that up with your frieldnly `docker compose up -d` command.

Now, all that remains is to put the virtual IP, in this case 10.15.0.180 into your DCHP server's DNS and job done. DNS requests will always go to 10.15.0.181, unless the server goes down. In which case, it'll fail over to 10.15.0.182, but always respond on 10.15.0.180


