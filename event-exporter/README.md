## See README-ORIGINAL.md for more details.

This is a modified version of `event exporter`, which can export "events" to a project of our choice.

## Steps

## Build image

```
make container
```
If the above command fails , add your user to docker group so that you can [run docker commands without sudo](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)

The newly built image will be tagged as `staging-k8s.gcr.io/event-exporter:v0.3.1` (or perhaps a different version)

tag the image to your own repository, from which your k8s clusters can pull.
```
docker tag staging-k8s.gcr.io/event-exporter:v0.3.1 <repository>/event-exporter:<tag>
```

## Upload service account json as a secret

Upload a service account key to a path accessible by container, and set env valriable to its path.
Please refer to the documentation [here](https://cloud.google.com/logging/docs/agent/authorization#copy-private-key) for more details.

creating the secret: 
```shell
kubectl create -n kube-system secret generic sa-key --from-file=application_default_credentials.json  --kubeconfig /path/to/kubeconfig-file
```

## Deploy event-notifier with approriate env-variable settings.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: event-exporter-sa
  namespace: default
  labels:
    app: event-exporter
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: event-exporter-rb
  labels:
    app: event-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: event-exporter-sa
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event-exporter
  namespace: default
  labels:
    app: event-exporter
spec:
  selector:
    matchLabels:
      app: event-exporter
  replicas: 1
  template:
    metadata:
      labels:
        app: event-exporter
    spec:
      serviceAccountName: event-exporter-sa
      containers:
      - name: event-exporter
        image: <repository>/event-exporter:<tag>
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /etc/google/auth/application_default_credentials.json
        volumeMounts:
        - name: sa-key
          mountPath: /etc/google/auth
        command:
        - '/event-exporter'
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sa-key
        secret: 
          defaultMode: 420
          secretName: sa-key
```
