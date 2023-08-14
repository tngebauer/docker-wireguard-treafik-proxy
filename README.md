# Tunneling Multiple Local Webservices over a secure Wireguard Connection and Exposing them using Traefik
# All in a single Docker Container


## Get your client config
Start the Container to generate your wireguard configs. 

Make sure to map the containers config directory to your servers filesystem or a docker volume, so that the configs are persistant and easily accessible.

Get the client config from ```config/peer1/peer1.conf```, change the remote ip and deploy them to the local machine running the web services. Check if your local machine can ping the remote server. Everything else is required to be modified at the server-side.

## Configuring Wireguard (Server)

Add 
```
PostUp = iptables -P FORWARD DROP
```
in the ```config/wg0.conf``` right after the ```Private Key: XXXX```-Line.
Then add 
```
PostUp = iptables -A FORWARD -i eth0 -o wg0 -p tcp --syn --dport $PORT -m conntrack --ctstate NEW -j ACCEPT
PostUp = iptables -A FORWARD -i eth0 -o wg0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostUp = iptables -A FORWARD -i wg0 -o eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostUp = iptables -t nat -A PREROUTING -i eth0 -p tcp --dport $PORT -j DNAT --to-destination 10.0.1.2
PostUp = iptables -t nat -A POSTROUTING -o wg0 -p tcp --dport $PORT -d 10.0.1.2 -j SNAT --to-source 10.0.1.1
```
for every service that you want to expose from your local machine (has to be TCP) and replace $PORT with the corresponding port.
This setup forwards the TCP port $PORT from the container to the TCP port $PORT of your local machine. 
If you want to map your local machines port to a different container port, you have to nat the destination port using iptables.
In this case all $PORT occurances in the above code represent the containers TCP $PORT. 
.

## Configuring Traefink (in the docker-compose.yml)
1. Expose the Port of every service you want to forward. 
2. Add the following fields to the ```labels:``` for every service and fill all data corresponding to your services:
```
- traefik.http.services.$SERVICE.loadbalancer.server.port=$PORT
- traefik.http.routers.$SERVICE.service=$SERVICE
- traefik.http.routers.$SERVICE.rule=Host(`$DOMAIN`)
- traefik.http.routers.$SERVICE.tls.certresolver=myresolver
- traefik.http.routers.$SERVICE.entrypoints=websecure
```
$PORT is the exposed port of the container 

$SERVICE is a unique service identifier you can choose

$DOMAIN is your subdomain traefik should route 

