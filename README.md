# wireguard_google_cloud
Instructions to have your own Wireguard VPN with Google Cloud Free Tier

### Create Google Cloud Compute instance

[Documentation for free Tier](https://cloud.google.com/free/docs/gcp-free-tier/#compute)

Now go create the instance at https://console.cloud.google.com/compute/instancesAdd
* region: us-east1 or one of the region indicated in the documentation.
* Select General Purpose -> N1 -> F1-micro. You will see "Your first 744 hours of f1-micro instance usage are free this month"
* Select Boot disk -> public image -> ubuntu -> 20.04LS -> boot disk type: Standard persistent disk (HDD) -> size 30gb (or as per documentation)
* Allow http and https traffic (or don't check the boxes, if you don't intend to use port 80 and 443)
* Make sure to allow IP forwarding BEFORE creating the instance, this setting cannot be changed later on
* Click on Create


You can check "view billing report" to make sure you did it right.

You will also need a static address (not free but cheap): https://console.cloud.google.com/networking/addresses , script:

```
gcloud compute addresses create mywebsite --project=$PROJECT_ID --network-tier=STANDARD --region=us-east1
gcloud compute instances add-access-config instance-1 --project=$PROJECT_ID --zone=us-east1-b --address=IP_OF_THE_NEWLY_CREATED_STATIC_ADDRESS --network-tier=STANDARD
```

Finally you need to add ingress (inbound) traffic for port 51820 UDP in the https://console.cloud.google.com/networking/firewalls with authorized IPs : 0.0.0.0/0

### Create and add ssh key:

#### 1/ Generate the key (no paraphrase)


On local cpu:
By default on linux $USER will give you your username (`echo $USER`), so you don't even need to specify it.
```
ssh-keygen -t rsa -f ~/.ssh/my_google_cloud_key -C $USER
```

#### 2/ Copy the key to Google Cloud metadata
```
cat /home/$USER/.ssh/my_google_cloud_key.pub
```

Select and copy it in https://console.cloud.google.com/compute/metadata/sshKeys (add key, then save)

#### 3/ Connect to your VM

Get the external IP of your instance at https://console.cloud.google.com/compute/instances

```
EXTERNAL_IP={{input your external ip}}
ssh-i ~/.ssh/my_google_cloud_key $USER@$EXTERNAL_IP
```

---
### Install Wireguard

Pretty straigt forward:
```
sudo apt update
sudo apt install wireguard
```

---
### Setup the Wireguard Cloud Server


#### Create the server public and private keys

The most important key will be the one the clients use to connect to your server. Let's generate the private and public keys:
```
mkdir keys
umask 077
wg genkey | tee keys/server_privatekey | wg pubkey > keys/server_publickey
```

If you check the content of server_privatekey you will have this kind of output:
```
2PSr88GjSkO1y/FPXAdVhHYuZnwpyw5kfhFtps5lflY=
```

#### Setup the config file

Just run the following command:
```
echo "
[Interface]
Address = 10.200.200.1/24
MTU = 1460
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = $(cat 'keys/server_privatekey')"  | sudo tee -a /etc/wireguard/wg0.conf > /dev/null
```
*Only one important thing: Check that the Private key correctly matches the one in keys/server_privatekey*  

#### Enable IP forwarding

Still on the remote server, you need to allow ip forward with:
```
sudo sysctl -w net.ipv4.ip_forward=1
```
**Without IP forwarding it will not work.**  

And also edit sysctl.conf to make the change permanent:
```
sudo nano /etc/sysctl.conf
uncomment:  net.ipv4.ip_forward = 1
sysctl -p /etc/sysctl.conf
```

#### Start the server

start the system service and enable it:
```
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

You can check that the server is running smoothly by running the following command:
```
sudo wg show
```

#### (Optional but recommended) Setup the firewall rules

```
sudo apt-get install ufw &&
sudo ufw allow 22/tcp &&
sudo ufw allow 51820/udp &&
sudo ufw status verbose
```

**Be very cautious to have allowed SSH port (22) to access your server again** and also port 51820 as it was set as the listening port for wireguard, but any other port than 51820 should work fine.  

After my warning, and having checked that port 22 is allowed, you may enable firewall using the following command:
```
sudo ufw enable
```
You will get the related warning:
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```
Type the letter 'y', then press Enter to activate the firewall and enable on system startup. You will get disconnected from your current ssh session.


---
### Generating Wireguard Client Config files

#### Give your client a name

Using ```${wgclient}``` we will get the value of the client you are currently creating as a tmp environment variable:
```
wgclient=client1
```
You can call it John, Bobby or whatever. This will be the user. It can also be a guid.

#### Generate client private and public keys

You will store the client public and private keys in the keys directory you created
```
umask 077
wg genkey | tee keys/${wgclient}_privatekey | wg pubkey > keys/${wgclient}_publickey
```

Then you have to register this new client to the server config file:
```
echo "
[Peer] #${wgclient}
PublicKey = $(cat keys/${wgclient}_publickey)
AllowedIPs = 10.200.200.2/32" | sudo tee -a /etc/wireguard/wg0.conf > /dev/null

&& wg addconf wg0 <(wg-quick strip wg0)
```
You have to change 10.200.200.*2*/32 number each time you add a client (increment the number).

**WARNING:** Don't forget that the use >> instead of > is mandatory. You want to append and not recreate the file!  

#### Generate client conf file

create the clients folder which will hold the .conf files:
```
mkdir clients
```

create the client conf:
```
echo "[Interface] #${wgclient}
Address = 10.200.200.2/32
MTU = 1460
PrivateKey = $(cat 'keys/${wgclient}_privatekey')
DNS = 1.1.1.1

[Peer]
PublicKey = $(cat 'keys/server_publickey')
Endpoint = 13.84.227.135:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 21" > clients/${wgclient}.conf
```

**NB: Make sure to replace 13.84.227.135 with your actual elastic IP**  

**3 important things:**  
* In [Interface] you have to *replace the address' last number* (before 32) with the one that you previously edited
* Here [Peer] is in fact the Wireguard server. *You should check that this PublicKey (named 'server_publickey' and stored in keys folder in this tutorial) match the one you have on your remote server*. ```cat /etc/wireguard/wg0.conf``` will also show you the PrivateKey that should correspond to server_privatekey.
* You have to change the endpoint to *match the elastic IP* you got from AWS  

#### (Optional) Generate a QR code from the .conf file

You can share QR files (squared bar codes pictures) instead of the .conf file by installing qrencode:
```
sudo apt install qrencode
```

Then create the QR code from the client file:
```
qrencode -o ${wgclient}.png -t png < clients/${wgclient}.conf
```

#### Download the QR code on your laptop

##### First method using scp

On your local cpu:
```
scp ubuntu@13.84.227.135:clients/client1.png Downloads/
```

##### Second method using base64

On the remote server:
```
base64 < clients/${wgclient}.png
```

On your local cpu:
```
base64 -d > clients/client1.png
```

Then paste the base64 from remote server and press CTRL+D

##### Setup client on Linux

Install wireguard:

```
sudo add-apt-repository ppa:wireguard/wireguard &&
sudo apt-get update &&
sudo apt install wireguard
```
Now copy the .conf file you got from your remote server to wireguard folder:
```
sudo cp client1.conf /etc/wireguard/
```

To avoid [this issue](https://superuser.com/questions/1500691/usr-bin-wg-quick-line-31-resolvconf-command-not-found-wireguard-debian/1500896) install:

```
sudo apt install openresolv
```


You have to make sure that the port is allowed at https://console.cloud.google.com/networking/firewalls/


##### Important: MTU related to Google Cloud

Google cloud use a default MTU of [1460](https://cloud.google.com/network-connectivity/docs/vpn/concepts/mtu-considerations?hl=en) while the default MTU of Wireguard is 1420. If you don't change the default MTU the handshake will happen but you will have trouble to connect to the internet


You should add `MTU = 1460` to the server and clients interface

If already running use the following scripts:

Set MTU to 1460 on server
```
sudo ip link set dev wg0 mtu 1460
```

and client:

```
sudo ip link set dev client1 mtu 1460
```

---

and finally activate your VPN client  with:
```
sudo wg-quick up client1
```



### Any question? Anything I missed?

Feel free to contact me at gavrel dot antonin at gmail dot com
