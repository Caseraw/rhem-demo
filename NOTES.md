# NOTES.md

## Login with flightctl

```
oc login --token=XXX --server=https://api.ocp-hub.hotwheels.cow:6443

export EDGE_NS=redhat-rhem
export RHEM_API_SERVER_URL=$(oc get route -n ${EDGE_NS} flightctl-api-route -o json | jq -r .spec.host)

flightctl login https://${RHEM_API_SERVER_URL} -k --token=$(oc whoami -t)

flightctl certificate request --signer=enrollment --expiration=365d --output=embedded > config.yaml

```

## Login on container registry

```
export REGISTRY_AUTH_FILE=/home/cloud-user/rhem-cloud-init-config/pull-secret.json

podman login -u='XXX' -p='XXX' quay.io
```

## Build base bootc image

```
IMAGE_NAME=quay.io/rh-ee-kamirsar/base-bootc
IMAGE_TAG=v1

sudo podman build -t "${IMAGE_NAME}:${IMAGE_TAG}" -f Containerfile
```