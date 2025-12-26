# Container Parameters

In [open5gs.env](open5gs.env) the following parameters can be set:

SUBSCRIBER_DB=/db.csv

```
# Kept in the following format: "Name,IMSI,Key,OP_Type,OP/OPc,AMF,QCI,IP_alloc"
#
# Name:     Human readable name to help distinguish UE's. Ignored by the HSS
# IMSI:     UE's IMSI value
# Key:      UE's key, where other keys are derived from. Stored in hexadecimal
# OP_Type:  Operator's code type, either OP or OPc
# OP/OPc:   Operator Code/Cyphered Operator Code, stored in hexadecimal
# AMF:      Authentication management field, stored in hexadecimal
# QCI:      QoS Class Identifier for the UE's default bearer.
# IP_alloc: IP allocation stratagy for the SPGW.
#           With 'dynamic' the SPGW will automatically allocate IPs
#           With a valid IPv4 (e.g. '10.45.0.2') the UE will have a statically assigned IP.
#
# Note: Lines starting by '#' are ignored and will be overwritten
# List of UEs with IMSI, and key increasing by one for each new UE. Useful for testing with AmariUE simulator and ue_count option
ue01,001010000000001,fec86ba6eb707ed08905757b1bb44b8f,opc,C42449363BBAD02B66D16BC975D77CC1,9001,9,10.45.1.2
ue02,001010000000011,fec86ba6eb707ed08905757b1bb44b8f,opc,C42449363BBAD02B66D16BC975D77CC1,9001,9,10.45.2.2
ue03,001010000000008,fec86ba6eb707ed08905757b1bb44b8f,opc,C42449363BBAD02B66D16BC975D77CC1,9001,9,10.45.3.2
```

# Open5GS Parameters

Open5GS can be setup using [open5gs-5gc.yml](open5gs-5gc.yml).

```
  dns:
    - 35.9.37.103
    - 35.9.37.241
    - 2001:4860:4860::8888
    - 2001:4860:4860::8844
```

# Usage

Create a Docker network to assign a specified IP to the Open5GS conainer (here: 10.53.1.2):

`docker network create --subnet=10.53.0.0/16 open5gsnet`

Build the Docker container using:

`docker build --target open5gs -t open5gs-docker .`

You can overwrite open5gs version by adding `--build-arg OPEN5GS_VERSION=v2.6.6`

```
cd srsRAN_Project/docker/open5gs
sudo docker exec -it open5gs /bin/bash
iptables -t nat -L -n -v
sysctl -w net.ipv4.ip_forward=1
iptables-legacy -t nat -A POSTROUTING -s 10.45.0.0/16 -o eth0 -j MASQUERADE
```

Then run the docker container with:

```
cd srsRAN_Project/docker/open5gs
sudo docker run -d \
  --name open5gs \
  --net open5gsnet \
  --ip 10.53.1.2 \
  --env-file open5gs.env \
  --privileged \
  --publish 9999:9999 \
  -v $(pwd)/db.csv:/db.csv \
  open5gs-docker \
  ./build/tests/app/5gc -c open5gs-5gc.yml
```



To use this container with srsgnb, the `addr` option under `amf` section in gnb configuration must be set OPEN5GS_IP (here: 10.53.1.2).
It could also be required to modify `bind_addr` option under `amf` section in gnb configuration to the local ethernet/wifi IP address for the host or container where gnb is running, not a localhost IP.

```
amf: 10.53.1.2
blind: 10.53.0.1
```

To ping a connected UE setup the necessary route to the UE_IP_BASE + ".0/24" (here: 10.45.0) via the OPEN5GS_IP (here: 10.53.1.2) using:

`sudo ip ro add 10.45.0.0/16 via 10.53.1.2`

## Note

The Open5GS WebUI to manually add/change UEs to the mongodb can be accessed at [localhost:9999](localhost:9999).
