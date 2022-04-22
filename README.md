# EdgeRouter-LoadBalance

#1. Enter Configuration mode.
configure

#2. Create a firewall network group specifying the private IP address ranges.
set firewall group network-group PRIVATE_NETS network 192.168.0.0/16
set firewall group network-group PRIVATE_NETS network 172.16.0.0/12
set firewall group network-group PRIVATE_NETS network 10.0.0.0/8

#3. Create a firewall modify policy with exclusion rules for the WAN interface addresses and the network group created earlier.
set firewall modify balance rule 10 action modify
set firewall modify balance rule 10 destination group network-group PRIVATE_NETS
set firewall modify balance rule 10 modify table main

set firewall modify balance rule 20 action modify
set firewall modify balance rule 20 destination group address-group ADDRv4_eth0
set firewall modify balance rule 20 modify table main

set firewall modify balance rule 30 action modify
set firewall modify balance rule 30 destination group address-group ADDRv4_eth4
set firewall modify balance rule 30 modify table main


#4. Add a firewall rule entry that sends all other traffic to a load balancing group.
set firewall modify balance rule 110 action modify
set firewall modify balance rule 110 modify lb-group G

#5. Apply the firewall to the LAN interface in the ingress/in direction.
set interfaces switch switch0 firewall in modify balance

#6. Create a Load-Balance group that includes the two WAN interfaces.
set load-balance group G interface eth0
set load-balance group G interface eth4

#Interface configured with the failover-only option will only become active when the other WAN interface(s) fail the route test.
#set load-balance group G interface eth0 failover-only

#Set Extra Steps for Testing the WAN Connection
set load-balance group G interface eth0 route-test count success 5
set load-balance group G interface eth0 route-test count failure 3
set load-balance group G interface eth0 route-test initial-delay 10
set load-balance group G interface eth0 route-test interval 5
set load-balance group G interface eth0 route-test type ping target 1.1.1.1

set load-balance group G interface eth4 route-test count success 5
set load-balance group G interface eth4 route-test count failure 3
set load-balance group G interface eth4 route-test initial-delay 10
set load-balance group G interface eth4 route-test interval 5
set load-balance group G interface eth4 route-test type ping target 9.9.9.9

#The flush-on-active feature clears all connections in the conntrack table whenever a WAN transition occurs.
set load-balance group G flush-on-active enable

#The lb-local-metric-change feature automatically changes the router's default route distance and is most useful when using a failover setup.
set load-balance group G lb-local-metric-change enable

#7. Commit the changes and save the configuration.
commit; save


Speedtest iperf from CLI command:
iperf3 -c bouygues.iperf.fr -R -P 10

Feel free to choose a server from this website:
https://iperf.fr/iperf-servers.php
