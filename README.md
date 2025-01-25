# Comprehensive Hadoop Cluster Deployment Guide for Ubuntu

## Preparation Phase

### Step 1: System Requirements

Minimum Specifications:

- Ubuntu Server 24.04 LTS
- 4 Servers (1 NameNode, 3 DataNodes)
- Minimum Specs per Node:
  * CPU: 4 cores
  * RAM: 16 GB
  * Storage: 
    - NameNode: 500 GB SSD
    - DataNodes: 1-2 TB HDD

### Step 2: Pre-Installation Preparation

1. Update All Servers
   
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

2. Install Basic Dependencies
   
   ```bash
   sudo apt-get install -y openssh-server ssh net-tools wget curl
   ```

3. Configure Hostname (Do this on EACH server)
   
   ```bash
   # On NameNode
   sudo hostnamectl set-hostname hadoop-namenode
   # On Datanodes 
   sudo hostnamectl set-hostname hadoop-datanode-1
   sudo hostnamectl set-hostname hadoop-datanode-2
   sudo hostnamectl set-hostname hadoop-datanode-3
   ```

## Step 3: Network Configuration

```bash
#Edit `/etc/hosts` on ALL servers:

sudo nano /etc/hosts

# Add these lines (use actual IP addresses)
192.168.18.1 hadoop-namenode
192.168.18.11 hadoop-datanode-1
192.168.18.12 hadoop-datanode-2
192.168.18.13 hadoop-datanode-3
```

### Step 4: Java Installation

```bash
# Install Java 8 or Java 11
sudo apt-get install -y openjdk-8-jdk

# Verify Java installation
java -version
javac -version
```

### Step 5: Create Hadoop User (Optional but Recommended)

```bash
# Create hadoop group
sudo groupadd hadoop

# Create hadoop user
sudo useradd -m -s /bin/bash -g hadoop hdadmin

# Add to sudoers
sudo usermod -aG sudo hdadmin

# Set password
sudo passwd hdadmin
```

### Step 6: SSH Passwordless Configuration

On NameNode:

```bash
# Switch to hdadmin user
sudo su - hdadmin

# Generate SSH Key
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa

# Copy public key to all nodes
ssh-copy-id hdadmin@hadoop-namenode
ssh-copy-id hdadmin@hadoop-datanode-1
ssh-copy-id hdadmin@hadoop-datanode-2
ssh-copy-id hdadmin@hadoop-datanode-3

# Test SSH connectivity
ssh hadoop-datanode-1 hostname
```

### Step 7: Hadoop Download and Installation

```bash
# Download Hadoop (use latest stable version)
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz

# Extract Hadoop
tar -xzvf hadoop-3.4.0.tar.gz

# Move to installation directory
sudo mv hadoop-3.4.0 /opt/hadoop

# Set ownership
sudo chown -R hdadmin:hadoop /opt/hadoop
```

### Step 8: Environment Configuration

Edit `~/.bashrc` for hdadmin user:

```bash
# Hadoop Environment Variables
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_HOME=/opt/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin

# Source the file
source ~/.bashrc
```

### Step 9: Hadoop Configuration Files

```bash
# /opt/hadoop/etc/hadoop/core-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop-namenode:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/hadoop/tmp</value>
    </property>
</configuration>

# /opt/hadoop/etc/hadoop/hdfs-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///opt/hadoop/hdfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///opt/hadoop/hdfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
        <value>false</value>
    </property>
</configuration>

# /opt/hadoop/etc/hadoop/yarn-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop-namenode</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
</configuration>

# /opt/hadoop/etc/hadoop/mapred-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=/opt/hadoop</value>
    </property>
</configuration>

# /opt/hadoop/etc/hadoop/workers
hadoop-datanode-1
hadoop-datanode-2
hadoop-datanode-3


# /opt/hadoop/etc/hadoop/masters
hadoop-namenode
```

### Step 10: Directory Preparation

```bash
# Create necessary directories
sudo mkdir -p /opt/hadoop/hdfs/name
sudo mkdir -p /opt/hadoop/hdfs/data
sudo mkdir -p /opt/hadoop/tmp

# Set correct permissions
sudo chown -R hdadmin:hadoop /opt/hadoop
```

### Step 11: Distributed File Copy

On NameNode:

```bash
# Copy configuration to all nodes
for host in hadoop-namenode hadoop-datanode-1 hadoop-datanode-2 hadoop-datanode-3; do
    scp /opt/hadoop/etc/hadoop/core-site.xml $host:/opt/hadoop/etc/hadoop/
    scp /opt/hadoop/etc/hadoop/hdfs-site.xml $host:/opt/hadoop/etc/hadoop/
    scp /opt/hadoop/etc/hadoop/yarn-site.xml $host:/opt/hadoop/etc/hadoop/
    scp /opt/hadoop/etc/hadoop/mapred-site.xml $host:/opt/hadoop/etc/hadoop/
    scp /opt/hadoop/etc/hadoop/workers $host:/opt/hadoop/etc/hadoop/
done
```

### Step 12: Firewall Configuration

```bash
# Allow Hadoop ports
sudo ufw allow 9000   # NameNode RPC
sudo ufw allow 9870   # NameNode Web UI
sudo ufw allow 8088   # YARN ResourceManager
sudo ufw allow 22     # SSH
```

### Step 13: Cluster Initialization

On NameNode:

```bash
# Format NameNode (ONLY ONCE)
hdfs namenode -format

# Start HDFS
start-dfs.sh

# Start YARN
start-yarn.sh
```

### Step 14: Verification

```bash
# Check running services
jps

# HDFS Cluster Report
hdfs dfsadmin -report

# List DataNodes
hdfs dfsadmin -listDatanodeReport

# Test HDFS
hdfs dfs -mkdir /test
hdfs dfs -touchz /test/testfile.txt
```

![](/home/magician/snap/marktext/9/.config/marktext/images/2025-01-26-01-18-40-image.png) 

### Step 15: Basic MapReduce Test

```bash
# Run Pi Calculation
yarn jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar pi 10 100
```

![](/home/magician/snap/marktext/9/.config/marktext/images/2025-01-26-01-25-54-image.png)

![](/home/magician/snap/marktext/9/.config/marktext/images/2025-01-26-01-25-37-image.png)

## 

## Post-Deployment Recommendations

### Monitoring

View the details on NameNode IP with browser

http://<hadoop-namenode>:9870

![](/home/magician/snap/marktext/9/.config/marktext/images/2025-01-26-01-21-05-image.png)

![](/home/magician/snap/marktext/9/.config/marktext/images/2025-01-26-01-22-26-image.png)

1. Install monitoring tools
   
   ```bash
   # Optional: Install Ganglia for monitoring
   sudo apt-get install -y ganglia-monitor
   ```

### Backup Strategy

1. Regular NameNode Metadata Backup
   
   ```bash
   # Backup NameNode metadata
   hdfs dfsadmin -metasave backup_filename
   ```

### Performance Tuning

Edit `hadoop-env.sh`:

```bash
# Adjust JVM Heap Size
export HADOOP_HEAPSIZE=8192  # 8 GB
export YARN_HEAPSIZE=4096    # 4 GB
```

## Troubleshooting Common Issues

1. Check Logs
   
   ```bash
   # NameNode Logs
   tail -f $HADOOP_HOME/logs/hadoop-*-namenode-*.log
   #DataNode Logs
   tail -f $HADOOP_HOME/logs/hadoop-*-datanode-*.log
   ```



2. Common Startup Issues
- Verify all nodes can resolve hostnames
- Check SSH passwordless configuration
- Ensure consistent Java versions
- Verify file permissions

## Security Considerations

- Implement Kerberos Authentication
- Use encrypted communication
- Regular security patches
- Restrict file system permissions

## Scaling Considerations

- Add more DataNodes for horizontal scaling
- Increase node resources for vertical scaling

## Documentation

- Document cluster configuration
- Track hardware specifications
- Record version and patch levels

## Recommended Reading

1. Apache Hadoop Documentation
2. Hadoop: The Definitive Guide (O'Reilly)
3. Professional Hadoop (Wrox)

## Maintenance Checklist

- Monthly security updates
- Quarterly performance review
- Regular backup verification
- Monitor cluster health
- Plan for capacity expansion
