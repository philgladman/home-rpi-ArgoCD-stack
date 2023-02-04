# home-rpi-ArgoCD-stack
Deploy Grafana, Loki, Promtail, NAS, NFS Server, pihole, and wireguard via GitOps(ArgoCD) on a Raspberry Pi K3s Cluster

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
- create directory to mount drives to `sudo mkdir /nas-vol && sudo mkdir /nfs-vol`
- create smbusers group for NAS volume `sudo groupadd smbusers -g 1010`
- change group ownership for NAS volume `sudo chown root:smbusers -R /nas-vol`
- change directory permissions NAS volume `sudo chmod 770 -R /nas-vol`
- To make mount perisistent, edit /etc/fstab file with the following command `sudo vim /etc/fstab/` and add the following to the bottom of the existing mounts.
```bash
/dev/sda1 /nas-vol ext4 defaults 0 2
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

## Step 6.) - Install tools
- Install sops `brew install sops`
- Install gpg `brew install gpg`
- Install gpg `brew install ksops`

## Step 7.) - Configure SOPS
- Generate your GPG key by running the following script,
```
export KEY_NAME="k3s.argocd.cluster"
export KEY_COMMENT="sops key for argocd"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: ${KEY_COMMENT}
Name-Real: ${KEY_NAME}
EOF
```
- Now get the ID of the newly create gpg key `gpg --list-secret-keys "${KEY_NAME}"`
expected output below;
```
sec   rsa4096 2023-02-01 [SCEA]
      LEP97F720CDHEKDU38744F7995AE746DHJR4GL8D <<<< COPY THIS ID
uid           [ultimate] k3s.argocd.cluster (sops key for argocd)
ssb   rsa4096 2023-02-01 [SEA]
```
- Copy the key ID, and save as a variable `export KEY_ID=LEP97F720CDHEKDU38744F7995AE746DHJR4GL8D`
- Create the argocd namesapce `kubectl apply -f kustomize/argocd/argocd-ns.yaml`
- Import the key as a secret into our cluster 
```
gpg --export-secret-keys --armor "${KEY_ID}" |
kubectl create secret generic sops-gpg \
--namespace=argocd \
--from-file=sops.asc=/dev/stdin
```
- update the `.sops.yaml` file to have the new key id, `sed -i "" "s|REPLACE_ME|$KEY_ID|g" .sops.yaml`
- Now our sops key has been imported into our cluster, ArgoCD can use this key to encrypt/decrpyt our secrets.

## Step 8.) - Deploy ArgoCD
- clone your newly forked git repo locally `git clone https://github.com/<your-github-username>/home-rpi-ArgoCD-stack.git`
- cd into repo `cd home-rpi-ArgoCD-stack`
- deploy argocd with `kubectl apply -k kustomize/argocd/.`
- FYI - argocd release.yaml was created with this helm templating command `helm template argocd charts/argo-cd -f kustomize/argocd/values.yaml --include-crds --debug > kustomize/argocd/release.yaml`

## Step 9.) - Install NFS Driver & NFS Server
- Install the NFS Driver with `curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.1.0/deploy/install-driver.sh | bash -s v4.1.0 --`
- if you created a different name for your NFS Volume, us this command to change the name from `/nfs-vol` to your custom name, `sed -i "s|/nfs-vol|<your-custom-name>|g" nfs-driver/kube-nfs-server.yaml`

## Step 10.) - Configure samba
- add a username and password for the smbuser, this will be the user/pass you will use to access the NAS.
- replace `testuser` and `testpassword` below with a custom user/password of your choice
- create username for smbuser `sed -i "" 's|USERNAME_REPLACE_ME|testuser|g' kustomize/samba/smb-credentials.enc.yaml`
- create password for smbuser `sed -i "" 's|PASSWORD_REPLACE_ME|testpassword|g' kustomize/samba/smb-credentials.enc.yaml`
- Now use your sops key to encrypt these credentials `sops -e -i kustomize/samba/smb-credentials.enc.yaml`

## Step 11.) - Configure pihole
- Create password for pihole (replace `testpassword` with a custom password of your choice) `sed -i "" 's|PASSWORD_REPLACE_ME|testpassword|g' kustomize/pihole/pihole-credentials.enc.yaml`
- Now use your sops key to encrypt the password `sops -e -i kustomize/pihole/pihole-credentials.enc.yaml`
- Lets create a local DNS name for pihole, argocd, and grafana so we can acces their UI's by a DNS name vs ip and port number. In my example, I use `pihole.phils-home.com`. You can use anything as this will only be accessible locally.
- change the `kustomize/pihole/custom.list` file below to have your custom DNS name for argocd, grafana, and pihole. Also the ip address of your nginx ingress controller
```bash
192.168.1.x argocd.phils-home.com
192.168.1.x grafana.phils-home.com
192.168.1.x pihole.phils-home.com
```
- change metallb ipaddress that will be used for by pihole `sed -i "" "s|192.168.1.x|<your-available-metallb-ip>|g" kustomize/pihole/pihole-svc.yaml`
- change timezone in pihole to your local timezone `sed -i "" "s|America/New_York|<your-local-timezone>|g" kustomize/pihole/pihole-cm.yaml`
- copy the `custom.list` file with your pihole custom dns entry over to the `/nfs-vol/pihole/` dir with this `sudo mkdir -p /nfs-vol/pihole/ && sudo cp kustomize/pihole/custom.list /nfs-vol/pihole/`

## Step 12.) - Configure Grafana
- add a password for the admin user for grafana
- replace `testpassword` below with a custom password of your choice
- create password for smbuser `sed -i "" 's|PASSWORD_REPLACE_ME|testpassword|g' kustomize/monitoring/grafana-credentials.enc.yaml`
- Now use your sops key to encrypt these credentials `sops -e -i kustomize/monitoring/grafana-credentials.enc.yaml`

## Step 13.) - Configure wireguard
### Configure DYNU DNS (wireguard)
- You will need a url to point to your public ip address. Here we are going to use DYNU to get a free domain.
- Go to https://www.dynu.com/ and create an account
- Once logged in with your new accout go to the [Control Plane](https://www.dynu.com/en-US/ControlPanel), and click on `DDNS Services`.
- Click `Add`, and use Option 1 to create a custom domain to point to your public IP. For this we will use `www.test.com`.
- Click add, and then on the next page click save.
- Now we have a url that points to our Public IP. However, since you more than likely have a dynamic public IP, as soon as your Public Ip changes, this URL will no longer point to your public IP.

### Confiugre script to update IP Address (wireguard)
- Run the following on the master node
- create a directory for DYNU `mkdir ~/DYNU`
- create a MD5 hash of your DYNU password with this `echo -n "your-password" | md5sum`
- add the following to create our `~/DYNU/updateIP.sh` script, Replacing the values with your DYNU username and password hash.
```
cat << EOF > ~/DYNU/updateIP.sh
echo url="https://api.dynu.com/nic/update?username=<your-DYNU-username>&password=<your-DYNU-password-MD5-hash>" | curl -k -o /home/ubuntu/DYNU/dynu.log -K -
EOF
```
- update ownership and perms of our script `sudo chown root:root ~/DYNU/updateIP.sh && sudo chmod 700 ~/DYNU/updateIP.sh`
- Now that we have our script configured, lets create a cronjob that runs the script every 5 minutes `echo "*/5 * * * * /home/pi/DYNU/updateIP.sh" >> /var/spool/cron/crontabs/root`. You will need to run this as root, `sudo su`.
- Now that our Public Ip is getting updated every 5 minutes, we can move forward with installing Wireguard VPN.

### Configure and install Wireguard VPN (wireguard)
- edit the `kustomize/wireguard-cm.yaml` to have your values
- set the `TZ` variable to have your Timezone.
- for this demo we will create 2 peers `philiphone` and `philmackbook`. Feel free to change the names, or add more/less. The peer name must only contain letters and numbers.
- `PEERDNS` is set to `auto`. If you have a pihole running, you can set this `PEERDNS` to the clusterIP of your pihole svc.
- `INTERNAL_SUBNET` is the subnet on the vpn.
- Upddate wireguard config to have your newly craeted DYNU url
- replace `www.test.com` below with your url `sed -i "" 's|URL_REPLACE_ME|www.test.com|g' kustomize/wireguard/host-url.enc.yaml`
- encrypt this url with `sops -e -i kustomize/wireguard/host-url.enc.yaml`
- On your router, you will need to port foward port 51820 UDP to the ip of the wireguard svc, which we will determine after all our apps are deployed.


## Step 14.) - Commit changes back to repo
- first we need to update our argocd files to point to your new repo `export NEW_REPO_URL=<your-new-git-repo-url>` (example = https://github.com/philgladman/home-rpi-ArgoCD-stack.git)
```bash
sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/parent-app/master-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/ingress-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/monitoring-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/nfs-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/pihole-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/samba-app.yaml && sed -i "" "s|https://github.com/philgladman/home-rpi-ArgoCD-stack.git|$NEW_REPO_URL|g" kustomize/apps/child-apps/wireguard-app.yaml
```
- confirm that our 3 secrets have been encrypted with our sops key.
  - `kustomize/samba/smb-credentials.enc.yaml`
  - `kustomize/pihole/pihole-credentials.enc.yaml`
  - `kustomize/wireguard/host-url.enc.yaml`
- Now that we have made changes to the repo, we need commit and push them back up to github. `git add --all && git commit -m "replacing values in files" && git push`.

## Step 15.) - Deploy Apps
- `kubectl apply -k kustomize/.` 
- Wait for all pods to be up and running. This may take up to 5 minutes
- Once all pods are up, now you just need to configure your router or your host to use the ip address of the udp/tcp pihole service as its DNS Server. You can get this ip address by running this command `kubectl get svc -n pihole pihole-dns-udp -o yaml -o jsonpath='{.status.loadBalancer.ingress[].ip}'`
- Once you have updated your DNS on your host to point to pihole, you should be able to access your pihole by this custom DNS name, without having to port forward
- In your browser, connect to your pihole custome DNS name (ex = `pihole.phils-home.com/admin`) and login.
- Pihole is up and running! Dont forget to configure your router or host to use pihole as its DNS Server.

## Step 16.) - Finish wireguard confi
- Now that wireguard has been deployed we need to port forward to our cluster, as well as get the wireguard QR codes
- On your router, you will need to port foward port 51820 UDP to the ip of the wireguard svc(wireguard svc ip = `kubectl get svc -n wireguard wireguard -o jsonpath='{.status.loadBalancer.ingress[].ip}'`)
- Run the following command to output the logs of the wireguard container, which will have a QR code for each peer. `kubectl logs -n wireguard $(kubectl get pods -n wireguard -o jsonpath='{.items[].metadata.name}')`
- This will output the logs of the wireguard container, which will have a qr code for each peer. 
- Download the wireguard app on your phone/computer. 
- Once downloaded, click the plus sign to add a new wireguard tunnel
- click `Create from QR code`
- scan the QR code from earlier.
- Now you can turn your new VPN on and test.

## MISC
- Grafana username=`admin` & password=`kubectl get secret --namespace monitoring loki-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
- ArgoCD username=`admin` & password=`kubectl get secret --namespace argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode ; echo`
