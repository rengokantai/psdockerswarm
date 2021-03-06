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

#####security
######create CA and key
CA(root key)
```
openssl genrsa -out ca-key.pem 2048
openssl req -config /usr/lib/ssl/openssl.cnf -new -key ca-key.pem -x509 -days 1825 -out ca-cert.pem
```
generate
```
openssl genrsa -out manager1-key.pem 2048
openssl req -subj "/CN=manager1" -new -key manager1-key.pem -out manager1.csr
echo subjectAltName = IP:172.31.48.79,IP:127.0.0.1 > extfile.cnf
openssl x509 -req -days 365 -in manager1.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out manager1-cert.pem -extfile extfile.cnf

openssl genrsa -out manager2-key.pem 2048
openssl req -subj "/CN=manager2" -new -key manager2-key.pem -out manager2.csr
echo subjectAltName = IP:172.31.50.19,IP:127.0.0.1 > extfile.cnf
openssl x509 -req -days 365 -in manager2.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out manager2-cert.pem -extfile extfile.cnf
```
client
```
openssl genrsa -out node1-key.pem 2048
openssl req -subj "/CN=node1" -new -key node1-key.pem -out node1.csr
echo subjectAltName = IP:172.31.55.46,IP:127.0.0.1 > extfile.cnf
openssl x509 -req -days 365 -in node1.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out node1-cert.pem -extfile extfile.cnf

openssl genrsa -out node2-key.pem 2048
openssl req -subj "/CN=node2" -new -key node2-key.pem -out node2.csr
echo subjectAltName = IP:172.31.55.136,IP:127.0.0.1 > extfile.cnf
openssl x509 -req -days 365 -in node2.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out node2-cert.pem -extfile extfile.cnf
```
CREATE CLIENT KEYS
```
openssl genrsa -out client-key.pem 2048
openssl req -subj "/CN=client" -new -key client-key.pem -out client.csr
echo extendedKeyUsage = clientAuth > extfile.cnf
openssl x509 -req -days 365 -in client.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out client-cert.pem -extfile extfile.cnf
```
copy:(set chomd to 002 first)
```
scp -i mykey.pem ./ca-cert.pem ubuntu@172.31.48.79:/home/ubuntu/.docker/ca.pem
scp -i mykey.pem ./manager1-cert.pem ubuntu@172.31.48.79:/home/ubuntu/.docker/cert.pem
scp -i mykey.pem ./manager1-key.pem ubuntu@172.31.48.79:/home/ubuntu/.docker/key.pem

scp -i mykey.pem ./ca-cert.pem ubuntu@172.31.50.19:/home/ubuntu/.docker/ca.pem
scp -i mykey.pem ./manager2-cert.pem ubuntu@172.31.50.19:/home/ubuntu/.docker/cert.pem
scp -i mykey.pem ./manager2-key.pem ubuntu@172.31.50.19:/home/ubuntu/.docker/key.pem
```
```
scp -i mykey.pem ./ca-cert.pem ubuntu@172.31.55.46:/home/ubuntu/.docker/ca.pem
scp -i mykey.pem ./node1-cert.pem ubuntu@172.31.55.46:/home/ubuntu/.docker/cert.pem
scp -i mykey.pem ./node1-key.pem ubuntu@172.31.55.46:/home/ubuntu/.docker/key.pem
scp -i mykey.pem ./ca-cert.pem ubuntu@172.31.55.136:/home/ubuntu/.docker/ca.pem
scp -i mykey.pem ./node2-cert.pem ubuntu@172.31.55.136:/home/ubuntu/.docker/cert.pem
scp -i mykey.pem ./node2-key.pem ubuntu@172.31.55.136:/home/ubuntu/.docker/key.pem
```

do this in all 4 machines:
```
mkdir .docker
chmod 777 .docker
```
public key->cert.pem private->key.pem  

######securing daemons
system command:
```
vim /etc/systemd/system/docker.service.d   (/etc/default/docker)
```
edit:
```
DOCKER_OPTS="-H tcp://0.0.0.0:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem"
```
restart docker
```
service docker restart
```

######client node
export manager1 node ip address
```
export DOCKER_HOST=172.31.48.79:3376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/home/ubuntu/.docker
```
start new consul and manager:
manager1
```
docker -H tcp://172.31.48.79:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem run --restart=unless-stopped -d -h consul1 --name consul1 -v /mnt:/data     -p 172.31.48.79:8300:8300     -p 172.31.48.79:8301:8301     -p 172.31.48.79:8301:8301/udp     -p 172.31.48.79:8302:8302     -p 172.31.48.79:8302:8302/udp     -p 172.31.48.79:8400:8400     -p 172.31.48.79:8500:8500     -p 172.17.0.1:53:53/udp     progrium/consul -server -advertise 172.31.48.79 -join 172.31.48.79
```
manager2
```
docker -H tcp://172.31.50.19:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem run --restart=unless-stopped -d -h consul2 --name consul2 -v /mnt:/data     -p 172.31.50.19:8300:8300     -p 172.31.50.19:8301:8301     -p 172.31.50.19:8301:8301/udp     -p 172.31.50.19:8302:8302     -p 172.31.50.19:8302:8302/udp     -p 172.31.50.19:8400:8400     -p 172.31.50.19:8500:8500     -p 172.17.0.1:53:53/udp     progrium/consul -server -advertise 172.31.50.19 -join 172.31.48.79
```
install manager
```
docker -H tcp://172.31.48.79:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem run --restart=unless-stopped -h mgr1 --name mgr1 -d -p 3376:2376 -v /home/ubuntu/.docker:/certs:ro swarm manage --tlsverify --tlscacert=/certs/ca.pem --tlscert=/certs/cert.pem --tlskey=/certs/key.pem --host=0.0.0.0:2376 --replication --advertise 172.31.48.79:2376 consul://172.31.48.79:8500/
```


start consul clients:
client1
```
docker -H tcp://172.31.55.46:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem run --restart=unless-stopped -d -h consul-agt1 --name consul-agt1 -p 8300:8300 -p 8301:8301 -p 8301:8301/udp -p 8302:8302 -p 8302:8302/udp -p 8400:8400 -p 8500:8500 -p 8600:8600/udp progrium/consul -rejoin -advertise 172.31.55.46 -join 172.31.48.79
```
client2
```
docker -H tcp://172.31.55.136:2376 --tlsverify --tlscacert=/home/ubuntu/.docker/ca.pem --tlscert=/home/ubuntu/.docker/cert.pem --tlskey=/home/ubuntu/.docker/key.pem run --restart=unless-stopped -d -h consul-agt2 --name consul-agt2 -p 8300:8300 -p 8301:8301 -p 8301:8301/udp -p 8302:8302 -p 8302:8302/udp -p 8400:8400 -p 8500:8500 -p 8600:8600/udp progrium/consul -rejoin -advertise 172.31.55.136 -join 172.31.48.79
```


######affinity filters
```
docker run -d --name c1 nginx
docker run -d --name c2 -e affinity:container==c1 nginx
docker run -d --name c3 -e affinity:container!=c1 nginx
```
######standard constranits
```
docker -H tcp://loalhost:2376 info
docker run -d --name newcontainer -e constraint:k1=v1 constraint:operatingsystem--windows-nano nginx
```
######custom constraints
```
vim /etc/default/docker
```

edit
```
DOCKER_OPTS: --label k1=y1 k2=y2
```

pull image:
```
docker run -d --name newcontainer -e constraint:k1=v1 constraint:k2=v2 nginx
```
