### cmd file

```shell

node1:
docker run -d --name=node1 --restart=always -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' -p 8300:8300 -p 8301:8301 -p 8301:8301/udp -p 8302:8302/udp -p 8302:8302 -p 8400:8400 -p 8500:8500 -p 8600:8600 -h node1 consul agent -server -bind=172.17.0.2 -bootstrap-expect=3 -node=node1 -data-dir=/tmp/data-dir -client 0.0.0.0 -ui
node2:
docker run -d --name=node2 --restart=always -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' -p 9300:8300 -p 9301:8301 -p 9301:8301/udp -p 9302:8302/udp -p 9302:8302 -p 9400:8400 -p 9500:8500 -p 9600:8600 -h node2 consul agent -server -bind=172.17.0.3 -join=10.0.8.7 -node-id=$(uuidgen | awk '{print tolower($0)}') -node=node2 -data-dir=/tmp/data-dir -client 0.0.0.0 -ui
node3:
docker run -d --name=node3 --restart=always -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' -p 10300:8300 -p 10301:8301 -p 10301:8301/udp -p 10302:8302/udp -p 10302:8302 -p 10400:8400 -p 10500:8500 -p 10600:8600 -h node3 consul agent -server -bind=172.17.0.4 -join=10.0.8.7 -node-id=$(uuidgen | awk '{print tolower($0)}') -node=node3 -data-dir=/tmp/data-dir -client 0.0.0.0 -ui


client:
docker run -d --name=node4  --restart=always -e 'CONSUL_LOCAL_CONFIG={"leave_on_terminate": true}' -p 11300:8300 -p 11301:8301 -p 11301:8301/udp -p 11302:8302/udp -p 11302:8302 -p 11400:8400 -p 11500:8500 -p 11600:8600 -h node4 consul agent -bind=172.17.0.5 -retry-join=10.0.8.7 -node-id=$(uuidgen | awk '{print tolower($0)}') -node=node4 -client 0.0.0.0 -ui


registrator:
docker run -d --name=registrator -v /var/run/docker.sock:/tmp/docker.sock --net=host gliderlabs/registrator -ip="10.0.8.7" consul://10.0.8.7:8500

```