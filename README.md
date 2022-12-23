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


## Step 1.) - Setup external drive for NAS and NFS Server
- now that the Raspberry pi is up and running with K3s, lets prepare the external drives
- Plug in the external drive to rpi for the NAS.
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
- Repeat this process for the NFS Server external drive.

## Step 2.) - Mount volume for NAS and for NFS Server
- Run the following on the master node
- create directory to mount drives to `sudo mkdir /NAS-volume && sudo mkdir /nfs-vol`
- create smbusers group for NAS volume `sudo groupadd smbusers -g 1010`
- change group ownership for NAS volume `sudo chown root:smbusers -R /NAS-volume`
- change directory permissions NAS volume `sudo chmod 770 -R /NAS-volume`
- To make mount perisistent, edit /etc/fstab file with the following command `sudo vim /etc/fstab/` and add the following to the bottom of the existing mounts.
```bash
/dev/sda1 /NAS-volume ext4 defaults 0 2
/dev/sdb1 /nfs-vol ext4 defaults 0 2
```
- mount the drives `sudo mount -a`

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
- label master node so samba container for the NAS will only run on master node since it has the NAS external drive connected `kubectl label nodes master disk=disk1`

## Step 5.) - Fork this git repo
- Since we will be using ArgoCD to deploy everything, you will need your own repository for Argo to pull its config from.
- Copy(fork) this repo by clicking [here](https://github.com/philgladman/home-rpi-ArgoCD-stack/fork)
- Once you have successfully forked this repo, move on to step 6.)

## Step 6.) - Deploy ArgoCD
- clone your newly forked git repo locally `git clone https://github.com/<your-github-username>/home-rpi-ArgoCD-stack.git`
- cd into repo `cd home-rpi-ArgoCD-stack`
- deploy argocd with `kubectl apply -k kustomize/argocd/.`

## Step 7.) - Install NFS Driver & NFS Server
- Install the NFS Driver with `curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.1.0/deploy/install-driver.sh | bash -s v4.1.0 --`
- if you created a different name for your NFS Volume, us this command to change the name from `/NFS-Vol` to your custom name, `sed -i "s|/nfs-vol|<your-custom-name>|g" nfs-driver/kube-nfs-server.yaml`
- Create the nfs namespace and the nfs-server `kubectl apply -k nfs-driver/.`

## Step 8.) - Configure samba
- add a username for the smbuser, this will be the user/pass you will use to access the NAS `echo -n "username" > kustomize/samba/smbcredentials/smbuser`
- add a password for smbuser `echo -n "testpassword" > kustomize/samba/smbcredentials/smbpass`

## Step 9.) - Configure pihole
- Create the directories below on the master node in order to persist our pihole data.
- `sudo mkdir -p /mnt/pihole/pihole`
- `sudo mkdir -p /mnt/pihole/dnsmasq.d`
- label master node so pihole container will only run on master node since that is where we want the persistent data to be stored `kubectl label nodes $(hostname) disk=disk1`
- Create password for pihole (replace `testpassword` with a custom password of your choice) `echo -n "testpassword" > kustomize/pihole/credentials/pihole-pass`
- Lets create a local DNS name for pihole, argocd, and grafana so we can acces their UI's by a DNS name vs ip and port number. In my example, I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `custom.list` file as well as the ingress to have your custom DNS name for __pihole__ `sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" kustomize/pihole/custom.list && sed -i "s|pihole.phils-home.com|<your-local-dns-name-for-pihole>|g" kustomize/pihole/pihole-ingress.yaml`
- change the `custom.list` file to have your custom DNS name for __argocd__ `sed -i "s|argocd.phils-home.com|<your-local-dns-name-for-argocd>|g" kustomize/pihole/custom.list && sed -i "s|argocd.phils-home.com|<your-local-dns-name-for-argocd>|g" kustomize/argocd/ingress.yaml`
- change the `custom.list` file to have your custom DNS name for __grafana__ `sed -i "s|grafana.phils-home.com|<your-local-dns-name-for-grafana>|g" kustomize/pihole/custom.list && sed -i "s|grafana.phils-home.com|<your-local-dns-name-for-grafana>|g" kustomize/monitoring/grafana-ingress.yaml`
- change the `custom.list` file to have the ip address of your nginx ingress controller `sed -i "s|192.168.1.x|$(kubectl get svc -n nginx-ingress nginx-ingress-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[].ip}')|g" kustomize/pihole/custom.list`
- change metallb ipaddress that will be used for by pihole `sed -i "s|192.168.1.x|<your-available-metallb-ip>|g" kustomize/pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "s|America/New_York|<your-local-timezone>|g" kustomize/pihole/pihole-cm.yaml`
- copy the `custom.list` file with your pihole custom dns entry over to the `/mnt/pihole/pihole/` dir with this `sudo cp kustomize/pihole/custom.list /mnt/pihole/pihole/`

## Step 10.) - Commit changes back to repo
- first we need to update our argocd files to point to your new repo `export NEW_REPO_URL=<your-new-git-repo-url>` (example = https://github.com/philgladman/home-rpi-ArgoCD-stack.git)
- `sed -i "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/parent-app/master-app.yaml && sed -i "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/ingress-app.yaml && sed -i "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/pihole-app.yaml && sed -i "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/ingress-app.yaml  && sed -i "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/samba-app.yaml`
- Now that we have made changes to the repo, we need commit and push them back up to github. `git add --all && git commit -m "replacing values in files" && git push`.
- FYI the passwords and usernames will NOT be be pushed up, as those are being ignored in the .gitignore file.

## Step 11.) - Deploy Apps
- `kubectl apply -k kustomize/.` 
- Wait for all pods to be up and running. This may take up to 5 minutes
- Once all pods are up, now you just need to configure your router or your host to use the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
- Once you have updated your DNS on your host to point to pihole, you should be able to access your pihole by this custom DNS name, without having to port forward
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Dont forget to configure your router or host to use pihole as its DNS Server.

## MISC
- Grafana username=`admin` & password=`kubectl get secret --namespace monitoring loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
- ArgoCD username=`admin` & password=`kubectl get secret --namespace argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode ; echo`
