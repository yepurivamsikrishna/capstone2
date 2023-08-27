# Rancher based k8s setup

We need atleast five servers for installation of Rancher based k8s

- (Rancher Server) - One   Centos 7 Node - 1 vCPU, 4 GB RAM, 20 GB disk
- (K8s cluster) - Three Centos 7 Nodes - 2 vCPU, 8 GB RAM, 40 GB disk
- (K8s client) - One Centos 7 Node - 1 vCPU, 2 GB RAM, 20 GB disk

<br/><br/>

#### Step 1 : Install docker on rancher server and all k8s cluster machines (not required on k8s client).

Remove old versions

```sh
>> sudo su -
>> sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

Setup the repository

```sh
>> sudo yum install -y yum-utils
>> sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker Engine

```sh
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Start docker service

```sh
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

<br/><br/>

#### Install rancher server on rancher server node

Install rancher server as docker container

```sh
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  --privileged \
  rancher/rancher:latest
```

Access Rancher Server dashboard using the ip of the rancher server

```sh
http://<ip-of-rancher-server>
```

Get the rancher server password by running the following command

```sh
docker ps -a
docker logs  <replace-with-container-id>  2>&1 | grep "Bootstrap Password:"
```

[https://ranchermanager.docs.rancher.com/pages-for-subheaders/rancher-on-a-single-node-with-docker]()

<br/><br/>

#### Create k8s cluster using rancher server and run the commands in k8s nodes

Following are the roles we are going to assign to the newly created cluster.

```sh
master node roles : master,worker,etcd
slave1 node roles : worker
slave2 node roles : worker
```

- In the rancher server dashboard, click on "Create" to create a new cluster.

- Select "Custom" option while selecting the hosting options.

- Provide cluster name and select network provider as "Flannel" and click on "Next"

- You will be able to see the node run commands. 

- Select all roles and copy the command to run on master Node

- Select only worker role and copy the command to run on slave nodes.

- Run the command in master node and wait for it to synchronize with rancher server.

- Run the command in slave nodes and wait to join the cluster and rancher server.

- Wait for some time until all nodes are provisioned and register in rancher server.

<br/><br/>

#### Verify everything is working fine

- Using kubectl shell in rancher server, connect to the newly created cluster.
- Run the command to check the cluster is setup
  
  ```sh
  >> kubectl get nodes
  ```
- You should see some output like this
  
  ```sh
  >> kubectl get nodes
  NAME     STATUS   ROLES                      AGE     VERSION
  master   Ready    controlplane,etcd,worker   4m14s   v1.26.7
  slave1   Ready    worker                     76s     v1.26.7
  slave2   Ready    worker                     30s     v1.26.7
  ```

<br/><br/>

#### (In k8s clinet node) Download kubectl and configure kube config to connect to the created k8s cluster

In order to connect to the cluster, we need to get kubectl tool.

```sh
>> sudo su -
>> curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
>> sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
>> kubectl version --client
```

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux

We also need clusters kubeconfig file to connect using kubectl. Get the clusters kube config file from rancher server and copy it into clipboard. Create a file ~/.kube/config and paste the kubeconfig in that file.

```sh
>> mkdir ~/.kube
>> vi ~/.kube/config 
```

Verify kubectl is able to connect to the cluster.

```
>> kubectl get nodes
NAME     STATUS   ROLES                      AGE     VERSION
master   Ready    controlplane,etcd,worker   16m   v1.26.7
slave1   Ready    worker                     13m     v1.26.7
slave2   Ready    worker                     12m     v1.26.7
```

<br/><br/>

## Configure NFS storage class for the created cluster

In ***k8s-nfs-dynamic-provisioning-setup-yaml/3nfs-client-provisioner.yaml***, replace with the actual ip of the NFS server.

```sh
>> cd k8s-nfs-dynamic-provisioning-setup-yaml
>> kubectl apply -f 1nfs-rbac.yaml
>> kubectl apply -f 2nfs-storageclass.yaml
>> kubectl apply -f 3nfs-client-provisioner.yaml
```

Make this "nfs-storageclass" storage class as default storage class. If your PVC yaml file contains the filed "*storageClass: someSCname*", then skip this step.

```shell
>> kubectl patch storageclass nfs-storageclass -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
