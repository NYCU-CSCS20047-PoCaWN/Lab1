# free5GC compose

This repository is a docker compose version of [free5GC](https://github.com/free5gc/free5gc) for stage 3. It's inspired by [free5gc-docker-compose](https://github.com/calee0219/free5gc-docker-compose) and also reference to [docker-free5gc](https://github.com/abousselmi/docker-free5gc).

You can setup your own config in [config](./config) folder and [docker-compose.yaml](docker-compose.yaml)

## Prerequisites

- [GTP5G kernel module](https://github.com/free5gc/gtp5g): needed to run the UPF (Currently, UPF only supports GTP5G versions 0.9.9 (use git clone --branch v0.9.9 --depth 1 https://github.com/free5gc/gtp5g.git).)
- [Docker Engine](https://docs.docker.com/engine/install): needed to run the Free5GC containers
- [Docker Compose v2](https://docs.docker.com/compose/install): needed to bootstrap the free5GC stack

## Start free5gc

Because we need to create tunnel interface, we need to use privileged container with root permission.


### [Optional] Build docker images from local sources

```bash
cd Lab1

# clone free5gc sources
cd base
git clone --recursive -j `nproc` https://github.com/free5gc/free5gc.git 
cd ..

# Build the images
make all
docker compose -f docker-compose-build.yaml build

```

Note:

Dangling images may be created during the build process. It is advised to remove them from time to time to free up disk space.

```bash
docker rmi $(docker images -f "dangling=true" -q)
```

### Run free5GC

You can create free5GC containers based on local images or docker hub images:

```bash
# use local images
docker compose -f docker-compose-build.yaml up

```

Destroy the established container resource after testing:

```bash
# Remove established containers (local images)
docker compose -f docker-compose-build.yaml rm
# Remove established containers (remote images)
docker compose rm
```

## Troubleshooting

Please refer to the [Troubleshooting](./TROUBLESHOOTING.md) for more troubleshooting information.

## Integration with (external) gNB/UE

### UERANSIM Notes

The integration with the [UERANSIM](https://github.com/aligungr/UERANSIM) eNB/UE simulator is documented [here](https://free5gc.org/guide/5-install-ueransim/).

This [issue](https://github.com/free5gc/free5gc-compose/issues/28) provides detailed steps that might be useful.

#### Option 1: Run UE inside gNB container

You can launch a UE using:

```console
docker exec -it ueransim bash
root@host:/ueransim# ./nr-ue -c config/uecfg.yaml
```

#### Option 2: Run UE on a separate container

By default, the provided UERANSIM service on this `docker-compose.yaml` will only act as a gNB. If you want to create a UE you'll need to:

1. Create a subscriber through the WebUI. Follow the steps [here](https://free5gc.org/guide/Webconsole/Create-Subscriber-via-webconsole/#4-open-webconsole)
1. Copy the `UE ID` field
1. Change the value of `supi` in `config/uecfg.yaml` to the UE ID that you just copied
1. Change the `linkIp` in `config/gnbcfg.yaml` to `gnb.free5gc.org` (which is also present in the `gnbSearchList` in `config/uecfg.yaml`) to enable communication between the UE and gNB services
1. Add an UE service on `docker-compose.yaml` as it follows:

```yaml
ue:
  container_name: ue
  image: free5gc/ueransim:latest
  command: ./nr-ue -c ./config/uecfg.yaml
  volumes:
    - ./config/uecfg.yaml:/ueransim/config/uecfg.yaml
  cap_add:
    - NET_ADMIN
  devices:
    - "/dev/net/tun"
  networks:
    privnet:
      aliases:
        - ue.free5gc.org
  depends_on:
    - ueransim
```

5. Run `docker-compose.yaml`
