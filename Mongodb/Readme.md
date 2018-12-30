``` Add MongoDB 4.0 repository to Fedora 29 
vi /etc/yum.repos.d/mongodb-org-4.0.repo


# Start & Enable MongoDB Service
sudo systemctl enable mongod.service
sudo systemctl start mongod.service

# Check status by running
sudo systemctl status mongod.service

# check the version of MongoDB
mongod --version 

# If you have SELinux in enforcing mode, you may need to label port 27017
sudo semanage port -a -t mongod_port_t -p tcp 27017

# Allow MongoDB Port on the firewall
sudo firewall-cmd --add-port=27017/tcp --permanent
sudo firewall-cmd --reload

# or 

sudo firewall-cmd --permanent --add-rich-rule "rule family="ipv4" \
source address="192.168.11.0/24" port protocol="tcp" port="27017" accept"


# Using secondary disk for MongoDB data (Optional)
# => Step 1: Partition secondary disk for MongoDB data:

$ lsblk  | grep vdb
vdb             252:16   0  50G  0 disk

# => Step 2: Create a GPT partition table for the secondary disk, it can be more than one disk

sudo parted -s -a optimal -- /dev/vdb mklabel gpt
sudo parted -s -a optimal -- /dev/vdb mkpart primary 0% 100%
sudo parted -s -- /dev/vdb align-check optimal 1

# => Step 3: Create LVM volume, this will make it easy to extend the partition

sudo pvcreate  /dev/vdb1
sudo vgcreate vg0 /dev/vdb1
sudo lvcreate -n mongo -l 100%FREE vg0

# => Step 4: Create XFS filesystem on the Logical Volume created

$ sudo mkfs.xfs /dev/mapper/vg0-mongo
meta-data=/dev/mapper/vg0-mongo isize=512    agcount=4, agsize=6553344 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=26213376, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=12799, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# => Step 5: Create a mount point and mount the partition

echo "/dev/mapper/vg0-mongo /var/lib/mongo xfs defaults 0 0" | sudo tee -a /etc/fstab
sudo mount -a
sudo chown -R mongod:mongod /var/lib/mongo
sudo chmod -R 775 /data/mongo

# => Step 6: Confirm that the partition mount was successful:

# df -hT | grep  /var/lib/mongo
/dev/mapper/vg0-mongo xfs        50G   33M   50G   1% /var/lib/mongo

# => Step 7: Set MongoDB data store location

$ sudo vim /etc/mongod.conf
storage:
dbPath: /var/lib/mongo 
journal:
enabled: true


mongodb-org-server – This provides MongoDB daemon mongod
mongodb-org-mongos – This is a MongoDB Shard daemon
mongodb-org-shell – This provides a shell to MongoDB
mongodb-org-tools – MongoDB tools used for export, dump, import e.t.c ´´´
