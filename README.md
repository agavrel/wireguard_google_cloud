# wireguard_google_cloud
Instructions to have your own Wireguard VPN with Google Cloud Free Tier

### Create Google Cloud Compute instance

[Documentation for free Tier](https://cloud.google.com/free/docs/gcp-free-tier/#compute)

Now go create the instance at https://console.cloud.google.com/compute/instancesAdd
* region: us-east1 or one of the region indicated in the documentation.
* Select General Purpose -> N1 -> F1-micro. You will see "Your first 744 hours of f1-micro instance usage are free this month"
* Select Boot disk -> public image -> ubuntu -> 20.04LS -> boot disk type: Standard persistent disk (HDD) -> size 30gb (or as per documentation)
* Allow http and https traffic (or don't check the boxes, if you don't intend to use port 80 and 443)
* Click on Create


You can check "view billing report" to make sure you did it right.

### Install ssh key:

### 1/ Generate the key (no paraphrase)


On local cpu:
By default on linux $USER will give you your username (`echo $USER`), so you don't even need to specify it.
```
ssh-keygen -t rsa -f ~/.ssh/my_google_cloud_key -C $USER
```

### 2/ Copy the key to Google Cloud metadata
```
cat /home/$USER/.ssh/my_google_cloud_key.pub
```

Select and copy it in https://console.cloud.google.com/compute/metadata/sshKeys (add key, then save)

### 3/ Connect to your VM

Get the external IP of your instance at https://console.cloud.google.com/compute/instances

```
EXTERNAL_IP={{input your external ip}}
ssh-i ~/.ssh/my_google_cloud_key $USER@$EXTERNAL_IP
```

---

# Install Wireguard on your local computer or device and on the remote google cloud server

```
sudo add-apt-repository ppa:wireguard/wireguard
sudo apt-get update
sudo apt-get install wireguard
```
