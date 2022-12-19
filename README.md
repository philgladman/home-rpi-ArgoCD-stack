# home-rpi-ArgoCD-stack
Deploy Grafana, Loki, Promtail, NAS, pihole, and wireguard via GitOps(ArgoCD) on a Raspberry Pi K3s Cluster

## Prereq Steps
### Raspberry Pi Setup
- Please follow this link for instructions on [How to install and setup K3s Cluster on raspberry pi](https://github.com/philgladman/home-rpi-k3s-cluster.git)
- This link will show you how to;
  - Prepare your raspberry pi for kubernetes
  - Spin up K3s
  - Install MetalLb
  - Install Nginx Ingress
- If you already have a Raspberry pi configured with K3s, MetalLb, and Nginx ingress, please move on to [Step 1.)](README.md#step-1---setup-external-drive-for-nas)


## Step 1.) - Setup external drive for NAS
- now that the Raspberry pi is up and running with K3s, lets prepare the external drive
- Plug in external drive to rpi.
- run `sudo fdisk -l` to find the drive, should be at the bottom, labeled something like`/dev/sda/`
- run `sudo fdisk /dev/sda` to partition the drive
- type in `d` and hit enter to delete, and then hit eneter to delete the default partition. Do this for each partition on the drive.
- Now that all partions have been deleted, lets create a new partition.
- hit `n` and enter to create a new partition
- hit enter for default for `primary` partition
- hit enter for default for `1 partition number`.
- hit enter for default for `First sector` size.
- hit enter for default for `Last sector` size.
- Partition 1 has been created of type `linux`, now hit `w` to write, this will save/create the partion and exit out of fdisk.
- Now make a filesystem on the newly created partition by running the following command `sudo mkfs -t ext4 /dev/sda1`

## Step 2.) - Mount volume for NAS
- create directory to mount drive to `sudo mkdir /NAS-volume`
- create smbusers group `sudo groupadd smbusers -g 1010`
- change group ownership `sudo chown root:smbusers -R /NAS-volume`
- change directory permissions `sudo chmod 770 -R /NAS-volume`
- To make mount perisistent, edit /etc/fstab file with the following command `sudo vim /etc/fstab/` and add the following to the bottom of the existing mounts. `/dev/sda1 /NAS-volume ext4 defaults 0 2`
- mount the drive `sudo mount -a`

## Step 3.) - Turn off systemd-resolver for pihole (Ubuntu)
- If using ubuntu, check if systemd-resolver is running and using port 53 `sudo lsof -i :53`.
- If output of command shows systemd-resolver is using 53, example below;
```bash
COMMAND   PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r 674 systemd-resolve   12u  IPv4  21062      0t0  UDP localhost:domain 
systemd-r 674 systemd-resolve   13u  IPv4  21063      0t0  TCP localhost:domain (LISTEN)
```
then run the following command to turn this off to free up port 53 for pihole `sudo sed -i "s/#DNS=/DNS=192.168.1.x/g; s/#DNSStubListener=yes/DNSStubListener=no/g" /etc/systemd/resolved.conf`. Change the IP Address with the IP of your Router.
- create sysmbolic link `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`
- reboot to apply changes `sudo systemctl reboot`
- confirm systemd-resolver is no longer using port 53 `sudo lsof -i :53`
- may have to update your /etc/hosts with `127.0.0.1 <your-hostname>`

## Step 4.) - Label Master node
- label master node so samba container will only run on master node since it has the external drive connected `kubectl label nodes $(hostname) disk=disk1`

## Step 5.) - Fork this git repo
- Since we will be using ArgoCD to deploy everything, you will need your own repository for Argo to pull its config from.
- Copy(fork) this repo by by clicking [here](https://github.com/philgladman/home-rpi-ArgoCD-stack/fork)
- Once you have successfully forked this repo, move on to step 6.)

## Step 6.) - Deploy ArgoCD
- clone your newly forked git repo locally `git clone https://github.com/<your-github-username>/home-rpi-ArgoCD-stack.git`
- cd into repo `cd home-rpi-ArgoCD-stack`
- Lets create a local DNS name for our ArgoCD server, so we can acces the ArgoCD UI by a DNS name vs ip and port number. In my example, I use `argocd.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `custom.list` file to have your custom DNS name for argocd `sed -i "s|argocd.phils-home.com|<your-local-dns-name-for-argocd>|g" pihole/custom.list`
- deploy argocd with `kubectl apply -k kustomize/argocd/.`
- wait for all argocd pods to be up and running `kubectl get pods -n argocd -w`
- when all pods are up, get ArgoCD admin password `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo`

## Step 7.) - Configure samba
- add a username for the smbuser, this will be the user/pass you will use to access the NAS `echo -n "username" > kustomize/samba/smbcredentials/smbuser`
- add a password for smbuser `echo -n "testpassword" > kustomize/samba/smbcredentials/smbpass`

## Step 8.) - Configure pihole
- Create the directories below on the master node in order to persist our pihole data.
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- label master node so pihole container will only run on master node since that is where we want the persistent data to be stored `kubectl label nodes $(hostname) disk=disk1`
- Create password for pihole (replace `testpassword` with a custom password of your choice) `echo -n "testpassword" > kustomize/pihole/credentials/pihole-pass`
- Lets create a local DNS name for pihole, argocd, and grafana so we can acces their UI's by a DNS name vs ip and port number. In my example, I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `custom.list` file as well as the ingress to have your custom DNS name for __pihole__ `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/custom.list && sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/pihole-ingress.yaml`
- change the `custom.list` file to have your custom DNS name for __argocd__ `sed -i "s|argocd.phils-home.com|<your-local-dns-name-for-argocd>|g" pihole/custom.list && sed -i "s|argocd.phils-home.com|<your-local-dns-name-for-argocd>|g" argocd/ingress.yaml`
- change the `custom.list` file to have your custom DNS name for __grafana__ `sed -i "s|grafana.phils-home.com|<your-local-dns-name-for-grafana>|g" pihole/custom.list && sed -i "s|grafana.phils-home.com|<your-local-dns-name-for-grafana>|g" monitoring/grafana-ingress.yaml`
- change the `custom.list` file to have the ip address of your nginx ingress controller `sed -i "s|192.168.1.x|$(kubectl get svc -n nginx-ingress nginx-ingress-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[].ip}')|g" pihole/custom.list`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.x|<your-available-metallb-ip>|g" pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" pihole/pihole-cm.yaml`
- copy the `custom.list` file with your pihole custom dns entry over to the `/mnt/pihole/pihole/` dir with this `sudo cp kustomize/pihole/custom.list /mnt/pihole/pihole/`

## Step 9.) - Deploy Apps
- `kubectl apply -k kustomize/.` 
- Wait for pod to spin up
- Now you just need to configure your router or your host to use the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
- Once you have updated your DNS on your host to point to pihole, you should be able to access your pihole by this custom DNS name, without having to port forward
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Dont forget to configure your router or host to use pihole as its DNS Server.

## MISC
- Get Grafana password `kubectl get secret --namespace monitoring loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
- Get ArgoCD password `kubectl get secret --namespace argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode ; echo`
