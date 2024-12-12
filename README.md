## Introduction to Charmed OpenSearch Rock (OCI Image)
[![Publish](https://github.com/canonical/charmed-opensearch-rock/actions/workflows/release.yaml/badge.svg)](https://github.com/canonical/charmed-opensearch-rock/actions/workflows/release.yaml)
[![Build and Test](https://github.com/canonical/charmed-opensearch-rock/actions/workflows/ci.yaml/badge.svg)](https://github.com/canonical/charmed-opensearch-rock/actions/workflows/ci.yaml)

[OpenSearch](https://opensearch.org/) is an open-source search and analytics suite. 
Developers build solutions for search, data observability, data ingestion and more using OpenSearch. 
OpenSearch is offered under the Apache Software Licence, version 2.0.

[Charmed OpenSearch rock](https://github.com/canonical/charmed-opensearch-rock/pkgs/container/charmed-opensearch) 
is an Open Container Initiative (OCI) image derived from the [Charmed OpenSearch Snap](https://snapcraft.io/opensearch). 
The tool used to create this rock is called [Rockcraft](https://canonical-rockcraft.readthedocs-hosted.com/en/latest/index.html).

This repository contains the packaging metadata for creating a Charmed OpenSearch rock. This rock image is based on the [OpenSearch Snap](https://github.com/canonical/opensearch-snap)

For more information on rocks, visit the [rockcraft Github](https://github.com/canonical/rockcraft).

## Version
The Charmed OpenSearch rock release aligns with the [OpenSearch upstream major version](https://opensearch.org/docs/latest/version-history/) naming. OpenSearch releases major versions such as 1.0, 2.0, and so on.

## Release
Charmed OpenSearch [Rock Release Notes](https://discourse.charmhub.io/t/release-notes-charmed-opensearch-2-rock/10278).


## Rock Usage
### Building the Rock
The steps outlined below are based on the assumption that you are building the rock with the latest LTS of Ubuntu.  
If you are using another version of Ubuntu or another operating system, the process may be different.
To avoid any issue with other operating systems you can simply build the image with [multipass](https://multipass.run/):
```bash
sudo snap install multipass
multipass launch 22.04 -n rock-dev
multipass shell rock-dev
``` 

#### Clone Repository
```bash
git clone https://github.com/canonical/charmed-opensearch-rock.git
cd charmed-opensearch-rock
```
#### Installing Prerequisites
```bash
sudo snap install rockcraft --edge --classic
sudo snap install docker
sudo snap install lxd
sudo snap install skopeo --edge --devmode
```
#### Configuring Prerequisites
```bash
sudo usermod -aG docker $USER 
sudo lxd init --auto
```
*_NOTE:_* You will need to open a new shell for the group change to take effect (i.e. `su - $USER`)
#### Packing and Running the Rock
```bash
rockcraft pack

version="$(cat rockcraft.yaml | yq .version)"

sudo skopeo --insecure-policy \
  copy \
  oci-archive:charmed-opensearch_"${version}"_amd64.rock \
  docker-daemon:charmed-opensearch:"${version}"

docker run \
  -d --rm -it \
  -e NODE_NAME=cm0 \
  -e INITIAL_CM_NODES=cm0 \
  -p 9200:9200 \
  --name cm0 \
  charmed-opensearch:"${version}"
```

### Testing a multi nodes deployment:
```
# create first cm_node container
container_0_id=$(docker run \
  -d --rm -it \
  -e NODE_NAME=cm0 \
  -e INITIAL_CM_NODES=cm0 \
  -p 9200:9200 \
  --name cm0 \
  charmed-opensearch:"${version}")
container_0_ip=$(docker inspect -f '{{ .NetworkSettings.IPAddress }}' "${container_0_id}")

# wait a bit for it to fully initialize
sleep 15s

# create data/voting_only node container
container_1_id=$(docker run \
    -d --rm -it \
    -e NODE_NAME=data1 \
    -e SEED_HOSTS="${container_0_ip}" \
    -e NODE_ROLES=data,voting_only \
    -p 9201:9200 \
    --name data1 \
    charmed-opensearch:"${version}")
container_1_ip=$(docker inspect -f '{{ .NetworkSettings.IPAddress }}' "${container_1_id}")

# wait a bit for it to fully initialize
sleep 15s

# create 2nd cm_node container
container_2_id=$(docker run \
    -d --rm -it \
    -e NODE_NAME=cm1 \
    -e SEED_HOSTS="${container_0_ip},${container_1_ip}" \
    -e INITIAL_CM_NODES="cm0,cm1" \
    -p 9202:9200 \
    --name cm1 \
    charmed-opensearch:"${version}")

# wait a bit for it to fully initialize
sleep 15s
```

You now can query the nodes:
```
curl -X GET http://127.0.1.1:9200/_nodes/
```
And expect to see 3 nodes.

**NOTE:** This deployment IS NOT suitable for production AS IS. As this deployment does not automate the security setup of OpenSearch. Please, set it manually or use the image as part of the Juju OpenSearch K8s charm once ready.

## License
The Charmed OpenSearch rock is free software, distributed under the Apache
Software License, version 2.0. See
[LICENSE](https://github.com/canonical/opensearch-rock/blob/main/licenses)
for more information.


## Security, Bugs and feature request
If you find a bug in this rock or want to request a specific feature, here are the useful links:
- Raise the issue or feature request in the [Canonical GitHub repository](https://github.com/canonical/charmed-opensearch-rock/issues).
- Meet the community and chat with us if there are issues and feature requests in our [Mattermost Channel](https://chat.charmhub.io/charmhub/channels/data-platform).

## Contributing
Please see the [Juju SDK docs](https://juju.is/docs/sdk) for guidelines on enhancements to this charm following best practice guidelines, and [CONTRIBUTING.md](https://github.com/canonical/mongodb-operator/blob/main/CONTRIBUTING.md) for developer guidance.

## Trademark notice
OpenSearch is a registered trademark of Amazon Web Services. Other trademarks are property of their respective owners. Charmed OpenSearch is not sponsored, endorsed, or affiliated with Amazon Web Services.

## License
The Charmed OpenSearch rock, Charmed OpenSearch Snap, and Charmed OpenSearch Operator are free software, distributed under the [Apache Software License, version 2.0](https://github.com/canonical/charmed-opensearch-rock/blob/main/licenses/LICENSE-rock). They install and operate OpenSearch, which is also licensed under the [Apache Software License, version 2.0](https://github.com/canonical/charmed-opensearch-rock/blob/main/licenses/LICENSE-opensearch).
