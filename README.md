# Nextcloud and Collabora Online on Kubernetes

The objective of this project is to provive a highly scalable and easy-to-deploy LibreOffice Online for collaborative works using the combination of Nextcloud and multiple instances of Collabora Online loadbalanced by HAProxy.

YAML files are provided to configure the Kubernetes cluster and to set-up Nextcloud using MariaDB, Collabora Online, HAProxy and Nginx as a SSL/TLS proxy.

## Pre-requisites
* Installed Debian GNU/Linux Buster (10.3)
* A simple user with Sudo privileges
* Basic Docker and Kubernetes knowledges

Type the following commands to install required packages:
```shell=
$ sudo apt update && sudo apt upgrade -y
$ sudo apt install -y docker.io curl gnupg
$ sudo systemctl enable docker
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update
$ sudo apt install -y kubeadm
$ sudo swapoff -a
$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl taint nodes --all node-role.kubernetes.io/master-
$ git clone https://github.com/nirinarisantatra/nextcloud-kubernetes/tree/master
```

Note down the kubeadm join-message printed in the console to be able to connect further Kubernetes nodes in the future.

Use the following in an extra-terminal to be able to see what the Kubernetes-cluster is doing:

```shell=
$ watch -n 5 kubectl get deployment,svc,pods,pvc,pv,ing
```

## MariaDB
Change into `kubernetes-yaml` directory.

Edit the db-deployment.yaml file to:
1. Change MYSQL_PASSWORD
2. Change MYSQL_ROOT_PASSWORD
3. Change database's HostPath, which should be the absolute location of db-pv (for example, /home/sant/nextcloud-k8s/db-pv)

Then deploy using the following commands:
```shell=
$ kubectl create -f db-deployment.yaml
$ kubectl create -f db-svc.yaml
```

## Nextcloud
Next, adjust the following by editing nc-deployment.yaml:
1. Change NEXTCLOUD_URL
2. Change NEXTCLOUD_ADMIN_PASSWORD
3. Change MYSQL_PASSWORD (the value entered before)
4. Change HTML's hostPath (for example, /home/sant/nextcloud-k8s/nc-pv)

Deploy using the following commands:
```shell=
$ kubectl create -f nc-deployment.yaml
$ kubectl create -f nc-svc.yaml
```
## Collabora Online
Edit the file c{1-3}-deployment.yaml and update the following field:
1. Change the hostAliases IP
2. Change the hostnames
3. Change the volumes hostPath (for example, /home/sant/nextcloud-k8s/collab-pv)

Inside loolwsd.xml:
1. Check that the IP private block and the FQDN of the reverse proxy are allowed on "Backend storage" section
2. Add the IPv4 private block on "Network settings" if not present

Deploy by executing these commands:
```shell=
$ kubectl create -f c1-deployment.yaml -f c2-deployment.yaml -f c3-deployment.yaml
$ kubectl create -f c1-svc.yaml -f c2-svc.yaml -f c3-svc.yaml
```

## HAProxy
The loadbalancer's configuration can be customized by editing the file haproxy.cfg:
1. Change the frontend SSL certificate file path
2. Change the backend section to indicate the number of Collabora Online instances used

Execute the following command to create the SSL certificate of HAProxy:
```shell=
$ sudo cat <cert_name>.key <cert_name>.crt >> <cert_name>.pem
```

Then update hp-deployment.yaml:
1. Change configuration's hostPath to the location provided before (for example, /home/sant/nextcloud-k8s/haproxy.cfg)
2. Change cert's hostPath (for example, /home/sant/nextcloud-k8s/certs-pv)

## Self-signed certificates
The [OMGWTFSSL-Docker](https://hub.docker.com/r/paulczar/omgwtfssl/) image offers an easy-to-use certificate creation. Here we are using only a Pod, not a Deployment. Once the certificates are generated, the Pod will stop.

Update omgwtfssl-pod.yaml according to the following:
1. Change SSL_SUBJECT to your server's name
2. Change CA_SUBJECT to your mail-adress
3. Change SSL_KEY to a proper filename
4. Change SSL_CSR to a proper filename
5. Change SSL_CERT to a proper filename
6. Change cert's hostPath (for example, /home/sant/nextcloud-k8s/certs-pv)

## Nginx as a reverse proxy
Use of a Nginx in front of Nextcloud and Collabora Online makes SSL encryption's configuration easy. The proxy is not a Deployment but a Pod, to provide the standard HTTP/HTTPS ports 80 and 443.

Change the following inside nginx.conf:
1. Change server_name (two locations in the file) to the server name provided earlier for SSL_SUBJECT
2. Change ssl_certificate to the filename provided earlier for SSL_CERT
3. Change ssl_certificate_key to the filename provided earlier for SSL_KEY

Update proxy-pod.yaml and:
1. Change cert's hostPath to the location provided before ---> change nginx-config's hostpath to the location where nginx.conf was stored before (for example, /home/sant/nextcloud-k8s/nginx.conf)
2. Change nginx-logs' hostpath to a proper location

Deploy using the below command:
```shell=
$ kubectl create -f proxy-pod.yaml
```

## Disable swap
To permanently disable the swap's use on the system, comment the corresponding line in /etc/fstab like this:
```shell=
#UUID=83046e3c-0d98-407f-afcf-8500643f0494 none            swap    sw              0       0
```

###### tags: `iCraft` `Documentation`