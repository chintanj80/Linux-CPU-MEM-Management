# Managing CPU and Memory on RHEL Linux

### Limit CPU usage on a Linux System

Use systemd slice to limit the CPU usage for a process

Sample config of a  systemd slice File

Location = /etc/systemd/system

Filename = slice400.slice  # 400 is to limit to 400% of CPU which is 4 CPU cores, 100% is 1 cpu core

```bash
cd /etc/systemd/system
vi slice400.slice
```

slice400.slice

```systemd
[Unit]
Description=Limit Processes attached to this slice to 4 CPU Cores

[Slice]
CPUQuota=400%
```

Sample service to use up to 8 cpu cores at 100%

cpuhog.service

```systemd
[Unit]
Description=Sample CPU hog Service

[Service]
ExecStart=/usr/bin/stress --cpu 8
```

```bash
systemctl daemon-reload
systemctl start cpuhog
```

Limiting cpuhog service to 400%

```systemd
```systemd
[Unit]
Description=Sample CPU hog Service

[Service]
Slice=slice400.slice
ExecStart=/usr/bin/stress --cpu 8
```

```
Setting cpu affinity for a systemd slice

Below slice is set to use only CPU cores 0 thru 3, 4 cores in total

slice400.slice

```systemd
[Unit]
Description=Limit Processes attached to this slice to 4 CPU Cores

[Slice]
CPUQuota=400%
AllowedCPUs=0-3
```

Splitting the slice into child slice with variable percentage of resources

Create two child slices for the slice400.slice with 75% and 25% of the quota

slice400-child25.slice

```systemd
```systemd
[Unit]
Description=Use 25% of the available cores for parent slice

[Slice]
CPUQuota=400%
```

```
### Start a process with limited CPU Quota using systemd-run scope

```shell
sudo systemd-run --scope -p CPUQuota=50% -- stress --cpu 4
```

CPUQuota = Percent of CPU allocated to the Process

100% = 1 entire CPU

We have asked stress command to run this process on 4 cpus

stress is a linux tool that allows you to simulate cpu usage like a real world process. It can be controlled via parameters

As we ask stress to distribute the load across 4 CPUs but systemd-scope has limited the available resources to this process to the equivalent of 50% cpu. This 50% gets divided across 4 CPU's. This results in each CPU getting utilized only 12.5 %.

### systemd User Slices

systemd automatically creates user slices and allows you to set per-user limits:

```bash
# Set CPU quota for a specific user (200% = 2 CPU cores worth)
sudo systemctl set-property user-1001.slice CPUQuota=200%

# Set memory limit for a user (2GB)
sudo systemctl set-property user-1001.slice MemoryMax=2G

# Set both CPU and memory limits
sudo systemctl set-property user-1001.slice CPUQuota=150% MemoryMax=1G

# View current user slice properties
systemctl show user-1001.slice
```

Make these persistent by creating drop-in files:

```bash
sudo mkdir -p /etc/systemd/system/user-1001.slice.d/
sudo tee /etc/systemd/system/user-1001.slice.d/resources.conf << EOF
[Slice]
CPUQuota=200%
MemoryMax=2G
TasksMax=500
EOF

sudo systemctl daemon-reload
```

### PAM Limits (/etc/security/limits.conf)

Set various resource limits per user or group

```bash
sudo vi /etc/security/limits.conf

# Format: <domain> <type> <item> <value>
# CPU time limits (in minutes)
john        hard    cpu         60
@developers soft    cpu         30

# Memory limits (in KB)
john        hard    rss         2097152    # 2GB
@users      soft    rss         1048576    # 1GB

# Process/thread limits
john        hard    nproc       100
@users      soft    nproc       50

# File descriptor limits
john        hard    nofile      1024
@users      soft    nofile      512

# Maximum file size (in KB)
john        hard    fsize       1048576    # 1GB
```

### **cgroups v2 with Custom Configuration**

Create custom cgroups for users:

```bash
# Create a cgroup for a user
sudo mkdir -p /sys/fs/cgroup/user_limits/john

# Set CPU limit (200000 = 200% = 2 cores worth over 100ms period)
echo "200000" | sudo tee /sys/fs/cgroup/user_limits/john/cpu.max

# Set memory limit (2GB)
echo "2147483648" | sudo tee /sys/fs/cgroup/user_limits/john/memory.max

# Add user's processes to the cgroup
echo $$ | sudo tee /sys/fs/cgroup/user_limits/john/cgroup.procs
```

### **Disk Quotas**

Limit disk usage per user:

```bash
# Enable quotas on filesystem (add usrquota to /etc/fstab)
sudo quotacheck -cum /home
sudo quotaon /home

# Set disk quota for user john (1GB soft, 1.2GB hard limit)
sudo setquota -u john 1000000 1200000 0 0 /home

# Check quota usage
quota -u john
```

### **Complete User Management Script**

Here's a script to set comprehensive user limits:

```bash
#!/bin/bash
# set_user_limits.sh

USER=$1
CPU_QUOTA=${2:-100%}    # Default 1 CPU core
MEMORY_LIMIT=${3:-1G}   # Default 1GB RAM
DISK_QUOTA=${4:-1000000} # Default 1GB disk

if [ -z "$USER" ]; then
    echo "Usage: $0 <username> [cpu_quota] [memory_limit] [disk_quota_kb]"
    exit 1
fi

# Get user ID
USER_ID=$(id -u $USER 2>/dev/null)
if [ $? -ne 0 ]; then
    echo "User $USER not found"
    exit 1
fi

echo "Setting limits for user: $USER (UID: $USER_ID)"

# Set systemd user slice limits
echo "Setting systemd limits..."
sudo systemctl set-property user-${USER_ID}.slice \
    CPUQuota=$CPU_QUOTA \
    MemoryMax=$MEMORY_LIMIT \
    TasksMax=200

# Set PAM limits
echo "Setting PAM limits..."
sudo tee -a /etc/security/limits.conf << EOF
$USER        hard    cpu         30
$USER        hard    rss         $((${MEMORY_LIMIT%G} * 1024 * 1024))
$USER        hard    nproc       100
$USER        hard    nofile      1024
EOF

# Set disk quota (if quotas are enabled)
if command -v setquota > /dev/null; then
    echo "Setting disk quota..."
    sudo setquota -u $USER $DISK_QUOTA $((DISK_QUOTA + 200000)) 0 0 /home 2>/dev/null || \
    echo "Disk quotas not enabled on /home"
fi

echo "Limits set for user $USER"
echo "User needs to log out and back in for some limits to take effect"
```

### Monitor Resource Usage

Track user resource consumption:

```bash
# Monitor per-user CPU and memory usage
sudo systemd-cgtop

# Check specific user slice
systemctl status user-1001.slice

# Monitor disk usage
repquota -u /home

# Real-time process monitoring by user
top -u john
```

## Best Practices

1. **Use systemd user slices** for CPU and memory limits on modern systems
2. **Combine with PAM limits** for process and file descriptor limits
3. **Enable disk quotas** for storage limits
4. **Test limits** in a non-production environment first
5. **Monitor regularly** to ensure limits are appropriate
6. **Document** the limits you've set for each user/group

The systemd approach is generally recommended for modern Linux distributions as it integrates well with the system's resource management and provides dynamic control.


