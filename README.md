# Kubernetes Cluster Backup Guide - Velero - MinIO on Docker
Full guide on how to backup kubernetes cluster using Velero CLI and MinIO running on Docker container. The documentation is prepared for **Linux**, specificially **Ubuntu  24.04**.

## Contents
1. Requirements
2. Deploy MinIO with Docker
3. Velero Configuration and Install
4. Creating Backup and Restore

## Step 1. Requirements
* sudo privilege
* git
* Docker. Follow [this](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) guide.
* kubectl with kubeconfig, should access to the cluster
* Velero CLI. Follow [this](https://velero.io/docs/v1.3.0/basic-install/#option-2-github-release) guide.
* ifconfig

## Step 2. Deploy MinIO with Docker
Quoted from [this](https://min.io/docs/minio/container/index.html) guide.
```c
mkdir -p ${HOME}/minio/data

docker run \
   -p 9000:9000 \
   -p 9001:9001 \
   --user $(id -u):$(id -g) \
   --name minio1 \
   -e "MINIO_ROOT_USER=minio" \
   -e "MINIO_ROOT_PASSWORD=minio123" \
   -v ${HOME}/minio/data:/data \
   quay.io/minio/minio server /data --console-address ":9001"
```
The example above works this way:
* `mkdir` creates a new local directory at `~/minio/data` in your home directory.
* `docker run` starts the MinIO container.
* `-p` binds a local port to a container port.
* `-user` sets the username for the container to the policies for the current user and user group.
* `-name` creates a name for the container.
* `-v` sets a file path as a persistent volume location for the container to use. When MinIO writes data to `/data`, that data actually writes to the local path `~/minio/data` where it can persist between container restarts. You can replace `${HOME}/minio/data` with another location in the user’s home directory to which the user has read, write, and delete access.
* `-e` sets the environment variables `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`, respectively. These set the root user credentials. Change the example values to use for your container.

## Step 3. Velero Configuration and Install
### For Minio Credentials.
You need to create a `minio.credentials` file for Velero to read credentials and access MinIO.

```c
cat <<EOF>> minio.credentials
[default]
aws_access_key_id=<YOUR-ACCESS-KEY-ID>
aws_secret_access_key=<YOUR-ACCESS-KEY-SECRET>
EOF
```

You can create access key and secret on MinIO UI.
1. Head to `http://localhost:9001/` and login with using credentials.
2. Locate to `Access Keys` from left menu.
3. Click `Create Access Key` on top right of the screen.
4. You can see `Access Key` and `Secret Key`. Copy and paste them to save.
5. Click `Create` and create an Access Key using these credentials.

### Get the IP address of your device
For Minikube to access Docker container you need to get the IP address of your device. You can get it with `ifconfig` command on Linux bash.
```bash
ifconfig
```
If the code above doesn't work well, you might need to install `net-tools`.
```bash
sudo apt-get install net-tools
```
After that run `ifconfig` again and see your IP address like:
```bash
$ ifconfig

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        # Your IP adress is right below
        inet <YOUR-IP-ADDRESS-HERE>  netmask 255.xxx.xxx.x  broadcast xxx.xxx.xxx
        inet6 xxxx:xxxx:xxxx:xxxx:xxxx  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:b9:a3:2b  txqueuelen 1000  (Ethernet)
        RX packets 5706  bytes 2017093 (2.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 909  bytes 133528 (133.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
### Deploy Velero on Kubernetes Cluster with using MinIO Credentials
```bash
velero install \
   --provider aws \
   --bucket <BUCKET-NAME-HERE> \
   --secret-file ./minio.credentials \
   --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://<YOUR-IP-ADDRESS-HERE>:9000 \
   --plugins velero/velero-plugin-for-aws
```

* Enter your bucket name that you create on Velero UI into `bucket`.
* Enter the IP Address that **you use to connect MinIO** into `s3Url`. MinIO API uses 9000 port so keep the port untouched.

After that command you should have successfully deployed Velero into your Kubernetes under the namespace `velero` and ready to work with Velero.

## Step 4. Creating Backup and Restore
### Creating Backup
For creating backup, you need to apply:
```bash
velero backup create [name for backup] --include-namespaces [namespaces will gonna backuped]
```
After applying the command, you will see a backup request submit done successfully. After few minutes you can check the list of backups with:
```bash
velero backup get
```
If you see `Created` near the name of your backup, your backup has been created successfully. If it has `Failed` you need to inspect the backup log or Velero pod log. For the help you can use the commands below:
```bash
# Backup logs
velero backup describe [backup name]
velero backup logs [backup name]

# Pod logs
## First you need to learn the name of your pod
kubectl get pods -n velero
## Then you can check the logs
kubectl describe pod [name of your pod] -n velero
kubectl logs [name of your pod] -n velero
```

### Restore
#### Creating manuel disaster and delete your namespace
For restore to work, you might need to create a manuel disaster scenerio and get rid of your namespace. For it you can do:
```bash
kubectl delete ns [name of namespace]
```
After doing that, you can restore with the restore command.
#### Restoring from a backup
To restore from a backup using Velero you can use this command below:
```bash
velero restore create [name-for-restoration] --from-backup [backup-name-will-be-restored]
```
Check the status of restore:
```bash
velero restore get
```
