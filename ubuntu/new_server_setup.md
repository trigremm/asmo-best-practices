# New server setup guide

### set hostname
```shell
hostnamectl set-hostname <new-hostname>

```

### general setup
```shell
sudo apt update
sudo apt upgrade -y

sudo apt install -y htop vim tree
sudo apt install -y wget curl rsync
sudo apt install -y git make
sudo apt install -y nginx
sudo apt install -y certbot python3-certbot-nginx

```

### install docker
[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

sudo usermod -aG docker ubuntu

### add swap
```shell
sudo swapon --show

sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf


sudo swapon --show
free -h

```

### mkdir workspace dir
```shell

sudo mkdir -p /workspace
sudo chown -R ubuntu:ubuntu /workspace

```

### mount ebs dir 
```
# find ebs device 
lsblk -o NAME,SIZE,MOUNTPOINT /dev/nvme1n1   # should show ~200G, no mountpoint
# format
sudo mkfs.ext4 /dev/nvme1n1

# mount
sudo mkdir -p /mnt/data
sudo mount /dev/nvme1n1 /mnt/data

# persist by UUID (nofail so a missing volume never blocks boot)
sudo blkid /dev/nvme1n1
echo 'UUID=<paste-uuid> /mnt/data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
# echo 'UUID="6d83dc36-6ba7-4834-973e-73513f0654e3" /mnt/data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab


# test fstab before trusting reboot
sudo umount /mnt/data && sudo mount -a && df -h /mnt/data

# subdirs + symlink + ownership
sudo mkdir -p /mnt/data/workspace /mnt/data/postgres /mnt/data/minio
sudo ln -s /mnt/data/workspace /workspace
sudo chown -R ubuntu:ubuntu /mnt/data

```

### rsync script 
```
# dedicated migration key, no passphrase so rsync runs non-interactively
ssh-keygen -t ed25519 -f ~/.ssh/migrate_ed25519 -C "migrate-new-to-old" -N ""

# print the PUBLIC key — copy this whole line
cat ~/.ssh/migrate_ed25519.pub

# replace the quoted string with the full ssh-ed25519 ... line from above
echo "ssh-ed25519 AAAA...migrate-new-to-old" >> ~/.ssh/authorized_keys

# make sure perms are sane (ssh refuses keys if these are too open)
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys

OLD_PRIVATE_IP=10.x.x.x                       # run `hostname -I` on the old box
KEY=~/.ssh/migrate_ed25519
REPO=/workspace/

# dry-run first

sudo rsync -aP --numeric-ids --dry-run \
  --exclude 'node_modules/' \
  --exclude '.venv/' --exclude 'venv/' \
  --exclude '__pycache__/' --exclude '*.pyc' \
  --exclude '.ruff_cache/' --exclude '.mypy_cache/' \
  --exclude 'dist/' --exclude 'build/' --exclude '.next/' \
  -e "ssh -i $KEY -o StrictHostKeyChecking=accept-new" \
  --rsync-path="sudo rsync" \
  ubuntu@$OLD_PRIVATE_IP:$REPO/ \
  $REPO/
```
