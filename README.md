# concourse-bootstrap
### Prerequisits
* Linux
* Docker-ce
#### For Yandex cloud installation, need also these:
* Yandex [yc](https://cloud.yandex.ru/docs/cli/quickstart), if you chose Yandex Managed Database(this is recommended, because create HA PostgreSQL and maintain take some time)
  * jq

## Abstract
* Bootstrap Concourse CI with bare minimum
 * Web
   * Database
     * On docker
     * On Yandex Managed Database
 * Worker


## Description
With local PostgreSQL in docker
```bash
docker run --name concourse-db \
 -v /opt/atc-db:/var/lib/postgresql/data/ \
 --restart=always \
 -h concourse-postgres \
 -p 5432:5432 \
 -e POSTGRES_USER=concourse \
 -e POSTGRES_PASSWORD=concourse \
 -e POSTGRES_DB=concourse \
 -d postgres:12.0 
```
With Managed Database PostgreSQL
On ATC host:
```bash
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
On your laptop:
https://cloud.yandex.ru/docs/managed-postgresql/operations/cluster-create#create-cluster
```

```bash
yc managed-postgresql cluster create \
     --name concourse \
     --environment production \
     --network-name default \
     --resource-preset s1.nano \
     --host zone-id=ru-central1-c,subnet-id=$(yc vpc network get --name=default --format=json  | jq -r .id) \
     --disk-type network-ssd \
     --disk-size 60 \
     --user name=concourse,password=concourse \
     --database name=concourse,owner=concourse
```
Create keys:
```bash
mkdir -p /opt/concourse/keys 
cd /opt/concourse/keys
ssh-keygen -t rsa -q -N '' -f ./tsa_host_key
ssh-keygen -t rsa -q -N '' -f ./worker_key
ssh-keygen -t rsa -q -N '' -f ./session_signing_key
```
Create script to launch Concourse CI with Yandex Managed DB:
```bash
#!/usr/bin/env bash
cw=5.8.0-ubuntu
docker run --name concourse-$cw -h concourse -p 2222:2222 -p 80:80 --detach --privileged --restart=always \
-v /opt/concourse/keys:/data/keys:ro \
-v /root/.postgresql/root.crt:/root/.postgresql/root.crt:ro \
-e CONCOURSE_CONTAINER_PLACEMENT_STRATEGY=volume-locality \
-e CONCOURSE_ADD_LOCAL_USER=concourserocks \
-e CONCOURSE_MAIN_TEAM_LOCAL_USER=concourserocks \
-e CONCOURSE_EXTERNAL_URL=http://your_ip_addresss8:80 \
-e CONCOURSE_BIND_PORT=80 \
-e CONCOURSE_POSTGRES_HOST=YOUR_MDB.mdb.yandexcloud.net \
-e CONCOURSE_POSTGRES_PORT=6432 \
-e CONCOURSE_POSTGRES_USER=concourse \
-e CONCOURSE_POSTGRES_DATABASE=concourse \
-e CONCOURSE_POSTGRES_PASSWORD=concourse \
-e CONCOURSE_POSTGRES_SSLMODE=verify \
-e CONCOURSE_POSTGRES_CA_CERT=/root/.postgresql/root.crt \
-e CONCOURSE_AUTH_DURATION=120h \
-e CONCOURSE_TSA_HEARTBEAT_INTERVAL=15s \
-e CONCOURSE_TSA_SESSION_SIGNING_KEY=/data/keys/session_signing_key \
-e CONCOURSE_TSA_HOST_KEY=/data/keys/tsa_host_key \
-e CONCOURSE_TSA_AUTHORIZED_KEYS=/data/keys/worker_key.pub \
-e CONCOURSE_ENCRYPTION_KEY=generate-some-key \
-e CONCOURSE_OLD_ENCRYPTION_KEY= \
concourse/concourse:$cw web
```
or

Create script to launch Concourse CI with PostgreSQL in docker:
```bash
#!/usr/bin/env bash
cw=5.8.0-ubuntu
docker run --name concourse-$cw -h concourse -p 2222:2222 -p 80:80 --detach --privileged --restart=always \
-v /opt/concourse/keys:/data/keys:ro \
-e CONCOURSE_CONTAINER_PLACEMENT_STRATEGY=volume-locality \
-e CONCOURSE_ADD_LOCAL_USER=concourserocks \
-e CONCOURSE_MAIN_TEAM_LOCAL_USER=concourserocks \
-e CONCOURSE_EXTERNAL_URL=http://your_ip_addresss8:80 \
-e CONCOURSE_BIND_PORT=80 \
-e CONCOURSE_POSTGRES_HOST=YOUR_POSTGRESQL_IP \
-e CONCOURSE_POSTGRES_PORT=5432 \
-e CONCOURSE_POSTGRES_USER=concourse \
-e CONCOURSE_POSTGRES_DATABASE=concourse \
-e CONCOURSE_POSTGRES_PASSWORD=concourse \
-e CONCOURSE_AUTH_DURATION=120h \
-e CONCOURSE_TSA_HEARTBEAT_INTERVAL=15s \
-e CONCOURSE_TSA_SESSION_SIGNING_KEY=/data/keys/session_signing_key \
-e CONCOURSE_TSA_HOST_KEY=/data/keys/tsa_host_key \
-e CONCOURSE_TSA_AUTHORIZED_KEYS=/data/keys/worker_key.pub \
-e CONCOURSE_ENCRYPTION_KEY=generate-some-key \
-e CONCOURSE_OLD_ENCRYPTION_KEY= \
concourse/concourse:$cw web
```

On your worker:
* [Worker bootstrap script](worker-bsp)


## TODO
* Simplify Worker part
* Use latest-greatest detection for Github assets on realesing new Concourse CI, with some tradeofs, like flags changedmay lead to non-working Concourse CI components anymore
* Worker as packer image, and terraform code to launch whole worker construction
  * Extend to take web too? 
