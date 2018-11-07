### Install IBM ACE from PPA Archive

	bx pr load-ppa-archive --archive /home/user/DOWNLOADS/ACE_V11.0_Container.tar.gz --clustername mycluster.icp

### Verify ACE production chart in the ICP catalog

> ICP -> Catalog

### Install and run

- install using Catalog
- show UI (NodePort)
- explain the default configuration
- show upload of BAR though GUI
- explain the server configuration settings - statistics, logging, etc.

### View internal structure - pod, deployment, service

	kubectl get pods
	kubectl get deploymnet
	kubectl get service

Store the pod and deployment name in a variable for re-reuse

	ACE_POD=$(kubectl get pods | grep ibm-ace | awk '{print $1}')
	echo $ACE_POD
	
	ACE_DEPLOYMENT=$(kubectl get deploy | grep ibm-ace | awk '{print $1}')
	echo $ACE_DEPLOYMENT

### Show logs
	
	kubectl logs $ACE_POD

### ACE configuration using `server.conf.yaml` via configmap

Locate the default ACE server config file

	kubectl exec $ACE_POD ls ace-server

Copy the default `server.conf.yaml` to local directory

	kubectl cp $ACE_POD:ace-server/server.conf.yaml .

Edit `server.conf.yaml` and modify the following items:

	resourceStatsReportingOn: true
	StatsSnapPublicationOn: active
	StatsSnapOutputFormat: "json,usertrace"

Create configmap

    kubectl create configmap ibm-ace-server-conf --from-file=server.conf.yaml

*Optional step:* Label the configmap

Mount the configmap as volume in the deployment:

```yaml
kubectl patch deployment $ACE_DEPLOYMENT --patch="$(cat <<EOF
spec:
  template:
    spec:
      containers:
      - name: $ACE_DEPLOYMENT
        volumeMounts:
        - name: server-config-volume
          mountPath: /home/aceuser/ace-server/server.conf.yaml
          subPath: server.conf.yaml
      volumes:
      - name: server-config-volume
        configMap:
          name: ibm-ace-server-conf
EOF
)"
```

### BAR upload via shared volume

Mount the NFS shared volume with BAR file

```yaml
kubectl get pvc

kubectl patch deployment $ACE_DEPLOYMENT --patch="$(cat <<EOF
spec:
  template:
    spec:
      containers:
      - name: $ACE_DEPLOYMENT
        args:
        - /bin/bash
        - -c
        - /usr/local/bin/ace_license_check.sh && mqsibar -w /home/aceuser/ace-server -a \$(ls -t /tmp/BARs/*.bar|head -1) -c && IntegrationServer -w /home/aceuser/ace-server --console-log
        volumeMounts:
        - name: bars
          mountPath: /tmp/BARs
      volumes:
      - name: bars
        persistentVolumeClaim:
          claimName: iib-pvc
EOF
)"
```

### Test new flow

	kubectl get service | grep 7800
	curl 192.168.24.33:31711/hello
  
