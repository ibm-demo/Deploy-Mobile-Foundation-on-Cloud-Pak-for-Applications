# Deploy Mobile Foundation on Cloud Pak for Applications

## Provision Db2 lite

Get a Db2 lite plan from IBM Cloud. It is limited to 5 simultaneous connections, so it can be used for small demo environment.

## Configure environment variables

```bash
export ENTITLED_REGISTRY=cp.icr.io
export ENTITLED_REGISTRY_USER=cp
export ENTITLED_REGISTRY_KEY=<REPLACE_ME>
export MFPF_DB_USERNAME=<REPLACE_ME>
export MFPF_DB_PASSWORD=<REPLACE_ME>
export OPENSHIFT_URL=$(oc whoami --show-server)
export OPENSHIFT_TOKEN=$(oc whoami --show-token)

export MFPF_DB_TYPE=db2
export MFPF_SERVER_DB_HOST=<REPLACE_ME>
export MFPF_SERVER_DB_PORT=50000
export MFPF_SERVER_DB_NAME=BLUDB
export MFPF_SERVER_DB_SCHEMA=<REPLACE_ME>
```

Customise the configuration

```bash
docker login "$ENTITLED_REGISTRY" -u "$ENTITLED_REGISTRY_USER" -p "$ENTITLED_REGISTRY_KEY"

mkdir ~/install-cp4app/data
cd ~/install-cp4app
mkdir data
docker run -v $PWD/data:/data:z -u 0 \
           -e LICENSE=accept \
           "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:4.1.1" cp -r "data/*" /data
cp data/mobilefoundation.yaml data/mobilefoundation.yaml.bak
sed -i "s/MFPF_SERVER_DB_HOST\|MFPF_LIVEUPDATE_DB_HOST\|MFPF_APPCNTR_DB_HOST/${MFPF_SERVER_DB_HOST}/g" data/mobilefoundation.yaml
sed -i "s/DB_TYPE/${MFPF_DB_TYPE}/g" data/mobilefoundation.yaml
sed -i "s/MFPF_SERVER_DB_PORT\|MFPF_LIVEUPDATE_DB_PORT\|MFPF_APPCNTR_DB_PORT/${MFPF_SERVER_DB_PORT}/g" data/mobilefoundation.yaml
sed -i "s/MFPF_SERVER_DB_NAME\|MFPF_LIVEUPDATE_DB_NAME\|MFPF_APPCNTR_DB_NAME/${MFPF_SERVER_DB_NAME}/g" data/mobilefoundation.yaml
sed -i "s/MFPF_SERVER_DB_SCHEMA\|MFPF_LIVEUPDATE_DB_SCHEMA\|MFPF_APPCNTR_DB_SCHEMA/${MFPF_SERVER_DB_SCHEMA}/g" data/mobilefoundation.yaml
sed -i "s/replicas: 2/replicas: 1/g" data/mobilefoundation.yaml
sed -i "s/storageClassName: \"\"/storageClassName: \"ibmc-file-gold\"/g" data/mobilefoundation.yaml
sed -i "/mfppush:/,/repository:/ s/enabled: true/enabled: false/g" data/mobilefoundation.yaml
sed -i "/mfpliveupdate:/,/repository:/ s/enabled: false/enabled: false/g" data/mobilefoundation.yaml
sed -i "/mfpanalytics:/,/repository:/ s/enabled: false/enabled: true/g" data/mobilefoundation.yaml
sed -i "/mfpanalytics_recvr:/,/repository:/ s/enabled: false/enabled: true/g" data/mobilefoundation.yaml
diff data/mobilefoundation.yaml data/mobilefoundation.yaml.bak
```

```bash
$ cat data/mobilefoundation.yaml
apiVersion: mf.ibm.com/v1
kind: MFOperator
metadata:
  name: mf
  labels:
    app.kubernetes.io/name: mf-operator
    app.kubernetes.io/instance: mf-instance
    app.kubernetes.io/managed-by: helm
spec:

  ###############################################################################
  ## Global configuration
  ###############################################################################
  global:
    arch:
      amd64: "3 - Most preferred"
    image:
      pullPolicy: IfNotPresent
      pullSecret: "{{ icpa.pullSecret }}"
    ingress:
      hostname: "mobilefoundation.{{ clusterSubdomain }}"
      secret: ""
      sslPassThrough: false
    https: false
    dbinit:
      enabled: true
      repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-dbinit"
      tag: 2.1.6
      resources:
        requests:
          cpu: 750m
          memory: 1024Mi
        limits:
          cpu: 1000m
          memory: 1024Mi
  ###############################################################################
  ## MFP Server configuration
  ###############################################################################
  mfpserver:
    enabled: true
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-server"
    tag: 2.1.6
    consoleSecret: ""   # created by default if left empty
    db:
      type: "db2"
      host: "<REPLACE_ME>"
      port: "50000"
      name: "BLUDB"
      secret: mobilefoundation-db-secret    # this secret is generated if it doesn't exists
      schema: "<REPLACE_ME>"
      ssl: false
      driverPvc: ""
      adminCredentialsSecret: ""
    adminClientSecret: ""
    pushClientSecret: ""
    internalClientSecretDetails:
      adminClientSecretId: mfpadmin
      adminClientSecretPassword: <REPLACE_ME>
      pushClientSecretId: push
      pushClientSecretPassword: hsup
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1536Mi
      limits:
        cpu: 2000m
        memory: 2048Mi
  ###############################################################################
  ## MFP Push configuration
  ###############################################################################
  mfppush:
    enabled: false
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-push"
    tag: 2.1.6
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1024Mi
      limits:
        cpu: 1500m
        memory: 2048Mi
  ###############################################################################
  ## MFP Liveupdate configuration
  ###############################################################################
  mfpliveupdate:
    enabled: false
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-liveupdate"
    tag: 2.1.6
    db:
      type: "db2"
      host: "<REPLACE_ME>"
      port: "50000"
      name: "BLUDB"
      secret: liveupdate-db-secret    # this secret is generated if it doesn't exists
      schema: "<REPLACE_ME>"
      ssl: false
      driverPvc: ""
      adminCredentialsSecret: ""
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1024Mi
      limits:
        cpu: 1500m
        memory: 2048Mi
  ###############################################################################
  ## MFP Analytics configuration
  ###############################################################################
  mfpanalytics:
    enabled: true
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-analytics"
    tag: 2.1.6
    consoleSecret: ""   # created by default if left empty
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    shards: "3"
    replicasPerShard: "1"
    persistence:
      enabled: true
      useDynamicProvisioning: true
      volumeName: "data-stor"
      claimName: ""
      storageClassName: "ibmc-file-gold"
      size: 20Gi
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1024Mi
      limits:
        cpu: 1500m
        memory: 2048Mi
 ###############################################################################
  ## MFP Analytics Receiver configuration
 ###############################################################################
  mfpanalytics_recvr:
    enabled: true
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-analytics-recvr"
    tag: 2.1.6
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    analyticsRecvrSecret: ""
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1024Mi
      limits:
        cpu: 1500m
        memory: 2048Mi
  ###############################################################################
  ## MFP Application center configuration
  ###############################################################################
  mfpappcenter:
    enabled: false
    repository: "{{ icpa.registry }}{{ icpa.mf.registryNamespace }}/mfpf-appcenter"
    tag: 2.1.6
    consoleSecret: ""   # created by default if left empty
    db:
      type: "db2"
      host: "<REPLACE_ME>"
      port: "50000"
      name: "BLUDB"
      secret: appcenter-db-secret    # this secret is generated if it doesn't exists
      schema: "<REPLACE_ME>"
      ssl: false
      driverPvc: ""
      adminCredentialsSecret: ""
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1024Mi
      limits:
        cpu: 1500m
        memory: 2048Mi
```

Run the installer

```bash
docker run -v ~/.kube:/root/.kube:z -u 0 -t \
    -v $PWD/data:/installer/data:z \
    -e LICENSE=accept \
    -e ENTITLED_REGISTRY -e ENTITLED_REGISTRY_USER -e ENTITLED_REGISTRY_KEY \
    -e OPENSHIFT_URL \
    -e OPENSHIFT_TOKEN \
    -e MFPF_DB_USERNAME -e MFPF_DB_PASSWORD \
    "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:4.1.1" mf-install
```

## Uninstall

```bash
docker run -v ~/.kube:/root/.kube:z -u 0 -t \
    -v $PWD/data:/installer/data:z \
    -e LICENSE=accept \
    -e ENTITLED_REGISTRY -e ENTITLED_REGISTRY_USER -e ENTITLED_REGISTRY_KEY \
    -e OPENSHIFT_URL \
    -e OPENSHIFT_TOKEN \
    -e MFPF_DB_USERNAME -e MFPF_DB_PASSWORD \
    "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:4.1.1" mf-uninstall
```

## Troubleshooting

### Workaround when secrets not created

```bash
oc create secret generic mf-d2c8bucfhjtey07v3hl3r6etx-mfppushclientsecret --from-literal=MFPF_ADMIN_AUTH_CLIENTID=mfpadmin --from-literal=MFPF_ADMIN_AUTH_SECRET=<REPLACE_ME> --from-literal=MFPF_PUSH_AUTH_CLIENTID=push --from-literal=MFPF_PUSH_AUTH_SECRET=<REPLACE_ME>
```
