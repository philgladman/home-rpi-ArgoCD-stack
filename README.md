# home-rpi-ArgoCD-stack
Deploy Grafana, Loki, Promtail, NAS, pihole, and wireguard on a Raspberry Pi K3s Cluster

## Prereq Steps
### Raspberry Pi Setup
- Please follow this link for instructions on [How to install and setup K3s Cluster on raspberry pi](https://github.com/philgladman/home-rpi-k3s-cluster.git)
- This link will show you how to;
  - Prepare your raspberry pi for kubernetes
  - Spin up K3s
  - Install MetalLb
  - Install Nginx Ingress
- If you already have a Raspberry pi configured with K3s, MetalLb, and Nginx ingress, please move on to [Step 1.)](README.md#step-1---configure-monitoring-stack)


# NAS
## Step 2.) - Setup external drive for NAS
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

## Step 3.) - Mount volume for NAS
- create directory to mount drive to `sudo mkdir /NAS-volume`
- create smbusers group `sudo groupadd smbusers -g 1010`
- change group ownership `sudo chown root:smbusers -R /NAS-volume`
- change directory permissions `sudo chmod 770 -R /NAS-volume`
- To make mount perisistent, edit /etc/fstab file with the following command `sudo vim /etc/fstab/` and add the following to the bottom of the existing mounts. `/dev/sda1 /NAS-volume ext4 defaults 0 2`
- mount the drive `sudo mount -a`

## Step 1.) - Turn off systemd-resolver for pihole (Ubuntu)
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
- may have to update your /etc/hosts with `127.0.0.0.1 <your-hostname>`

## Step 4.) - Label Master node
- label master node so samba container will only run on master node since it has the external drive connected `kubectl label nodes $(hostname) disk=disk1`

## Step 5.) - Deploy ArgoCD
- clone git repo `git clone https://github.com/philgladman/home-rpi-NAS.git`
- cd into repo `cd home-rpi-NAS`
- deploy argocd with `kubectl apply -k kustomize/argocd/.`
- wait for all argocd pods to be up and running `kubectl get pods -n argocd -w`
- when all pods are up, get ArgoCD admin password `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo`
- port forward ArgoCD service `kubectl port-forward svc/argocd-server 8080:8080 -n argocd`
- login to argocd `localhost:8080`, sign in with user=admin and password that you just retrieved
- keep this tab open as you deploy the Apps in the next step

## Configure samba
- add a username for the smbuser, this will be the user/pass you will use to access the NAS `echo -n "username" > kustomize/samba/smbcredentials/smbuser`
- add a password for smbuser `echo -n "testpassword" > kustomize/samba/smbcredentials/smbpass`
- deploy apps `kubectl apply -k kustomize/.` This will deploy a Master App, a Samba App, and a Nginx Ingress App.

## Configure pihole
- Create the directories below on the master node in order to persist our pihole data.
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- label master node so pihole container will only run on master node since that is where we want the persistent data to be stored `kubectl label nodes $(hostname) disk=disk1`
- clone this repo `git clone https://github.com/philgladman/home-rpi-pihole.git`
- cd into repo `cd home-rpi-pihole`
- Create password for pihole (replace `testpassword` with a custom password of your choice) `echo -n "testpassword" > pihole/pihole-pass`
- Lets create a local DNS name for our pihole server, so we can acces the pihole UI by a DNS name vs ip and port number. In my example, I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `custom.list` file to have your custom DNS name for pihole `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/custom.list`
- change the `custom.list` file to have the ip address of your nginx ingress controller `sed -i "s|192.168.1.x|$(kubectl get svc -n nginx-ingress nginx-ingress-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[].ip}')|g" pihole/custom.list`
- change pihole ingress name to your pihole custom DNS name `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" pihole/pihole-ingress.yaml`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.x|<your-available-metallb-ip>|g" pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" pihole/pihole-cm.yaml`
- copy the `custom.list` file with your pihole custom dns entry over to the `/mnt/pihole/pihole/` dir with this `sudo cp pihole/custom.list /mnt/pihole/pihole/`
- Deploy pihole `kubectl apply -k .` 
- Wait for pod to spin up
- Now you just need to configure your router or your host to use the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
- Once you have updated your DNS on your host to point to pihole, you should be able to access your pihole by this custom DNS name, without having to port forward
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Dont forget to configure your router or host to use pihole as its DNS Server.

# Grafana, Loki, Promtail
- Get Grafana password `kubectl get secret --namespace monitoring loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
- Port forward the grafana svc to 3000
- Go to `localhost:3000` and login with user=`admin` and the password from above