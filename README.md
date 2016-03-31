#### psdockerswarm
#####Building Your Swarm Cluster
######start
install docker (current version is 1.10.3)
```
sudo su && apt-get update
apt-get install linux-image-extra-$(uname -r)
apt-get install apt-transport-https ca-certificates
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo deb https://apt.dockerproject.org/repo ubuntu-trusty main >> /etc/apt/sources.list.d/docker.list
apt-get update && apt-get purge lxc-docker && apt-cache policy docker-engine && sudo apt-get install -y docker-engine
```

config docker on 2 EC2 instances:(config sg first)
NODE1
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
NODE2
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
NODE1
```
docker run --restart=unless-stopped -h mgr1 --name mgr1 -d -p 3375:2375 swarm manage --replication --advertise 172.31.48.79:3375 consul://172.31.48.79:8500/
```
check log and ps
```
docker logs mgr1
docker ps
```
	
NODE2
```
docker run --restart=unless-stopped -h mgr2 --name mgr2 -d -p 3375:2375 swarm manage --replication --advertise 172.31.50.19:3375 consul://172.31.50.19:8500/
```
