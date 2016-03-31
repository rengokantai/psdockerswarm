#### psdockerswarm
#####Building Your Swarm Cluster
######start
install docker (current version is 1.10.3)
```
sudo su
apt-get install -y linux-image-extra-$(uname -r)
apt-get install apt-transport-https ca-certificates
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo deb https://apt.dockerproject.org/repo ubuntu-trusty main >> /etc/apt/sources.list.d/docker.list
apt-get update && apt-get purge lxc-docker && apt-cache policy docker-engine && sudo apt-get install -y docker-engine
```

#####we create 4 EC2 (multi-AZ) instances, one manager with one client. 2 managers 2clients

config docker on 2 EC2 instances:(config sg first)
MANAGER NODE1
```
	docker run --restart=unless-stopped -d -h consul1 --name consul1 -v /mnt:/data \
    -p 172.31.48.79:8300:8300 \
    -p 172.31.48.79:8301:8301 \
    -p 172.31.48.79:8301:8301/udp \
    -p 172.31.48.79:8302:8302 \
    -p 172.31.48.79:8302:8302/udp \
    -p 172.31.48.79:8400:8400 \
    -p 172.31.48.79:8500:8500 \
    -p 172.17.0.1:53:53/udp \
    progrium/consul -server -advertise 172.31.48.79 -bootstrap-expect 3
```

verify
```
docker exec -it consul1 bash
consul members
```
you can see this
```
bash-4.3# consul members
Node     Address            Status  Type    Build  Protocol  DC
consul2  172.31.50.19:8301  alive   server  0.5.2  2         dc1
consul1  172.31.48.79:8301  alive   server  0.5.2  2         dc1
```
MANAGER NODE2
```
	docker run --restart=unless-stopped -d -h consul2 --name consul2 -v /mnt:/data  \
    -p 172.31.50.19:8300:8300 \
    -p 172.31.50.19:8301:8301 \
    -p 172.31.50.19:8301:8301/udp \
    -p 172.31.50.19:8302:8302 \
    -p 172.31.50.19:8302:8302/udp \
    -p 172.31.50.19:8400:8400 \
    -p 172.31.50.19:8500:8500 \
    -p 172.17.0.1:53:53/udp \
    progrium/consul -server -advertise 172.31.50.19 -join 172.31.48.79
```
verify
```
docker exec -it consul2 bash
consul members
```
######building HA swarm mgr
MANAGER NODE1
```
docker run --restart=unless-stopped -h mgr1 --name mgr1 -d -p 3375:2375 swarm manage --replication --advertise 172.31.48.79:3375 consul://172.31.48.79:8500/
```
check log and ps
```
docker logs mgr1
docker ps
```
	
MANAGER NODE2
```
docker run --restart=unless-stopped -h mgr2 --name mgr2 -d -p 3375:2375 swarm manage --replication --advertise 172.31.50.19:3375 consul://172.31.50.19:8500/
```
######joining nodes to cluster
consul client build:
CLIENT NODE1
```
docker run --restart=unless-stopped -d -h consul-agt1 --name consul-agt1 -p 8300:8300 -p 8301:8301 -p 8301:8301/udp -p 8302:8302 -p 8302:8302/udp -p 8400:8400 -p 8500:8500 -p 8600:8600/udp progrium/consul -rejoin -advertise 172.31.55.46 -join 172.31.48.79	
```
```
docker run -d swarm join --advertise=172.31.55.46:2375 consul://172.31.55.46:8500/
```
		
CLIENT NODE2
```
docker run --restart=unless-stopped -d -h consul-agt2 --name consul-agt2 -p 8300:8300 -p 8301:8301 -p 8301:8301/udp -p 8302:8302 -p 8302:8302/udp -p 8400:8400 -p 8500:8500 -p 8600:8600/udp progrium/consul -rejoin -advertise 172.31.55.136 -join 172.31.48.79	
```
```
docker run -d swarm join --advertise=172.31.55.136:2375 consul://172.31.55.136:8500/
```
######check:
in manager node 1
```
docker exec -it consul2 bash
consul members
```
should see 2 servers and 2 clients.  
  
in manager node 2
```
curl http://172.31.50.19:8500/v1/catalog/nodes | python -m json.tool
```
if run
```
docker stop consul2	//docker start consul2
```
we can see `failed` signal in manager node1.

install refistrar in all 4 nodes:(ip=clientnode2)
```
docker run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://172.31.48.136:8500
```

install nginx in client node 1
```
docker run -d --name web1 -p 80:80 nginx
```
in manager node2,run
```
curl http://172.31.55.46:8500/v1/catalog/service/swarm | python -m json.tool
```
