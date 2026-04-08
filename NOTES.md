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

sudo -i

export REGISTRY_AUTH_FILE=/home/cloud-user/rhem-cloud-init-config/pull-secret.json

podman login -u='XXX' -p='XXX' quay.io
```

## Build base bootc image

```
IMAGE_NAME=quay.io/rh-ee-kamirsar/base-bootc
IMAGE_TAG=v1

podman build -t "${IMAGE_NAME}:${IMAGE_TAG}" -f Containerfile

podman push "${IMAGE_NAME}:${IMAGE_TAG}"
```

## Create QCOW image

```
IMAGE_NAME=quay.io/rh-ee-kamirsar/base-bootc
IMAGE_TAG=v1

mkdir ./output
podman run --rm -it \
    --privileged \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    -v ./output:/output \
    registry.redhat.io/rhel9/bootc-image-builder:latest \
    --type qcow2 \
    ${IMAGE_NAME}:${IMAGE_TAG}

```

```
cp disk.qcow2 /var/www/html/downloads/base-bootc-v1.qcow2

```

```
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: base-bootc
  namespace: edge-virt
  annotations:
    "cdi.kubevirt.io/storage.bind.immediate.requested": "true"
spec:
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi
    volumeMode: Block
  source:
    http:
      url: http://rhel9-http-route-kasra-virt.apps.ocp-spoke-01.hotwheels.cow/downloads/base-bootc-v1.qcow2

apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: base-bootc
  namespace: openshift-virtualization-os-images
  annotations:
    "cdi.kubevirt.io/storage.bind.immediate.requested": "true"
spec:
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 30Gi
    volumeMode: Block
  source:
    http:
      url: http://rhel9-http-route-kasra-virt.apps.ocp-spoke-01.hotwheels.cow/downloads/base-bootc-v1.qcow2
```

```
kind: Template
apiVersion: template.openshift.io/v1
metadata:
  name: base-bootc-machine
  namespace: edge-virt
  labels:
    app.kubernetes.io/instance: base-bootc-vm-template
    app.kubernetes.io/name: common-templates
    app.kubernetes.io/version: 4.20.0
    template.kubevirt.io/type: base
    template.kubevirt.io/version: v0.34.1
  annotations:
    template.kubevirt.io/provider: RHEM demo
    template.kubevirt.io/architecture: amd64
    template.kubevirt.io/provider-url: 'https://www.redhat.com/en/products/edge'
    template.kubevirt.io/containerdisks: |
      registry.redhat.io/rhel9/rhel-guest-image
    template.kubevirt.io/version: v1alpha1
    openshift.io/display-name: Base Bootc Machine
    openshift.io/documentation-url: 'https://github.com/kubevirt/common-templates'
    defaults.template.kubevirt.io/disk: rootdisk
    name.os.template.kubevirt.io/rhel9.0: Red Hat Enterprise Linux 9.0 or higher
    name.os.template.kubevirt.io/rhel9.1: Red Hat Enterprise Linux 9.0 or higher
    template.kubevirt.io/editable: |
      /objects[0].spec.template.spec.domain.cpu.sockets
      /objects[0].spec.template.spec.domain.cpu.cores
      /objects[0].spec.template.spec.domain.cpu.threads
      /objects[0].spec.template.spec.domain.memory.guest
      /objects[0].spec.template.spec.domain.devices.disks
      /objects[0].spec.template.spec.volumes
      /objects[0].spec.template.spec.networks
    name.os.template.kubevirt.io/rhel9.2: Red Hat Enterprise Linux 9.0 or higher
    template.openshift.io/bindable: 'false'
    os.template.kubevirt.io/rhel9.0: 'true'
    name.os.template.kubevirt.io/rhel9.3: Red Hat Enterprise Linux 9.0 or higher
    os.template.kubevirt.io/rhel9.1: 'true'
    name.os.template.kubevirt.io/rhel9.4: Red Hat Enterprise Linux 9.0 or higher
    tags: 'hidden,kubevirt,virtualmachine,linux,rhde'
    os.template.kubevirt.io/rhel9.2: 'true'
    name.os.template.kubevirt.io/rhel9.5: Red Hat Enterprise Linux 9.0 or higher
    os.template.kubevirt.io/rhel9.3: 'true'
    template.kubevirt.io/provider-support-level: None
    os.template.kubevirt.io/rhel9.4: 'true'
    os.template.kubevirt.io/rhel9.5: 'true'
    description: Template for RHEM demo. Won't provision until a data source is available.
    os.template.kubevirt.io/rhel9.6: 'true'
    workload.template.kubevirt.io/server: 'true'
    os.template.kubevirt.io/rhel9.7: 'true'
    openshift.io/support-url: 'https://github.com/kubevirt/common-templates/issues'
    iconClass: icon-rhel
    openshift.io/provider-display-name: RHEM demo
objects:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      annotations:
        vm.kubevirt.io/validations: |
          [
            {
              "name": "minimal-required-memory",
              "path": "jsonpath::.spec.domain.memory.guest",
              "rule": "integer",
              "message": "This VM requires more memory.",
              "min": 1610612736
            }
          ]
      labels:
        app: '${NAME}'
        kubevirt.io/dynamic-credentials-support: 'true'
        vm.kubevirt.io/template: rhel9-server-small
        vm.kubevirt.io/template.revision: '1'
        vm.kubevirt.io/template.version: v0.31.1
      name: '${NAME}'
    spec:
      dataVolumeTemplates:
        - apiVersion: cdi.kubevirt.io/v1beta1
          kind: DataVolume
          metadata:
            name: '${NAME}-disk'
          spec:
            sourceRef:
              kind: DataSource
              name: base-bootc
              namespace: edge-virt
            storage:
              resources:
                requests:
                  storage: 30Gi
      running: false
      template:
        metadata:
          annotations:
            vm.kubevirt.io/flavor: small
            vm.kubevirt.io/os: rhel9
            vm.kubevirt.io/workload: server
          labels:
            kubevirt.io/domain: '${NAME}'
            kubevirt.io/size: small
        spec:
          architecture: amd64
          domain:
            cpu:
              cores: 1
              sockets: 1
              threads: 1
            devices:
              blockMultiQueue: true
              disks:
                - cache: writethrough
                  disk:
                    bus: virtio
                  name: rootdisk
                - disk:
                    bus: virtio
                  name: cloudinitdisk
              interfaces:
                - masquerade: {}
                  model: virtio
                  name: default
              rng: {}
            features:
              smm:
                enabled: true
            firmware:
              bootloader:
                efi: {}
            memory:
              guest: 2Gi
          networks:
            - name: default
              pod: {}
          terminationGracePeriodSeconds: 180
          volumes:
            - dataVolume:
                name: '${NAME}-disk'
              name: rootdisk
            - cloudInitNoCloud:
                userData: |-
                  #cloud-config
                  user: cloud-user
                  password: ${CLOUD_USER_PASSWORD}
                  chpasswd: { expire: False }
              name: cloudinitdisk
parameters:
  - name: NAME
    description: VM name
    value: base-bootc-machine


```