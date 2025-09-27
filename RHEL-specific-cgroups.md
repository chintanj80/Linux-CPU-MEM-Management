# RHEL Specific Considerations for User CPU/Mem Quota Management



## **cgroups Version Differences**

**RHEL 8:** Uses cgroups v1 by default, but v2 can be enabled **RHEL 9:** Uses cgroups v2 by default

```bash
# Check current cgroups version
mount | grep cgroup

# RHEL 8: Enable cgroups v2 (requires reboot)
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
sudo reboot

# Verify cgroups v2 is active
ls /sys/fs/cgroup/cgroup.controllers
```

## **RHEL-Specific systemd Configuration**

RHEL has some different default paths and configurations:

```bash
# RHEL-specific systemd user slice configuration
sudo mkdir -p /etc/systemd/system/user-.slice.d/
sudo tee /etc/systemd/system/user-.slice.d/00-rhel-limits.conf << EOF
[Slice]
CPUQuota=200%
MemoryMax=100G
TasksMax=500
# RHEL-specific: Enable accounting
CPUAccounting=true
MemoryAccounting=true
TasksAccounting=true
EOF
```

## **SELinux Considerations**

RHEL enforces SELinux by default, which can affect resource management:

```bash
# Check SELinux status
getenforce

# SELinux contexts for systemd files
sudo restorecon -Rv /etc/systemd/system/

# If you encounter SELinux denials, check audit logs
sudo ausearch -m avc -ts recent

# Common SELinux boolean for resource limits
sudo setsebool -P domain_can_mmap_files 1
```

## RHEL-Optimized Resource Management Script

Here's a RHEL-specific version of the management script:

```bash
#!/bin/bash
# rhel_resource_management.sh - RHEL 8/9 optimized resource management

# RHEL version detection
RHEL_VERSION=$(cat /etc/redhat-release | grep -oE '[0-9]+' | head -1)
CGROUPS_V2_ENABLED=false

check_rhel_prerequisites() {
    echo "=== RHEL Resource Management Setup ==="
    echo "Detected RHEL version: $RHEL_VERSION"
    
    # Check if cgroups v2 is enabled
    if [ -f /sys/fs/cgroup/cgroup.controllers ]; then
        CGROUPS_V2_ENABLED=true
        echo "cgroups v2: ENABLED"
    else
        echo "cgroups v2: DISABLED (v1 in use)"
        if [ "$RHEL_VERSION" -eq 8 ]; then
            echo "Consider enabling cgroups v2 for better resource management"
            echo "Run: grubby --update-kernel=ALL --args='systemd.unified_cgroup_hierarchy=1'"
        fi
    fi
    
    # Check SELinux status
    local selinux_status=$(getenforce 2>/dev/null || echo "Unknown")
    echo "SELinux status: $selinux_status"
    
    # Check required packages
    echo "Checking required packages..."
    for pkg in systemd quota util-linux; do
        if rpm -q "$pkg" > /dev/null 2>&1; then
            echo "  ✓ $pkg installed"
        else
            echo "  ✗ $pkg NOT installed - install with: dnf install $pkg"
        fi
    done
    
    echo ""
}

setup_system_wide_limits() {
    echo "Setting up system-wide limits for RHEL $RHEL_VERSION..."
    
    # 1. systemd user slice defaults with RHEL optimizations
    echo "Configuring systemd user slice defaults..."
    sudo mkdir -p /etc/systemd/system/user-.slice.d/
    sudo tee /etc/systemd/system/user-.slice.d/00-rhel-resources.conf << 'EOF'
[Slice]
CPUQuota=200%
MemoryMax=100G
TasksMax=500
# RHEL-specific accounting
CPUAccounting=true
MemoryAccounting=true
TasksAccounting=true
IOAccounting=true
# RHEL tuning
Slice=user.slice
EOF
    
    # 2. PAM limits with RHEL considerations
    echo "Configuring PAM limits..."
    
    # Backup original limits.conf
    if [ ! -f /etc/security/limits.conf.backup ]; then
        sudo cp /etc/security/limits.conf /etc/security/limits.conf.backup
    fi
    
    sudo tee -a /etc/security/limits.conf << 'EOF'

# RHEL System-wide user resource limits
*        hard    cpu         120        # CPU time in minutes  
*        hard    rss         104857600  # 100GB RAM in KB
*        hard    nproc       200        # Max processes
*        hard    nofile      2048       # Max open files
*        hard    fsize       unlimited  # Max file size
*        hard    stack       8192       # Stack size in KB
*        soft    cpu         60         # Soft CPU time limit
*        soft    rss         52428800   # 50GB soft RAM limit
*        soft    nproc       100        # Soft process limit
*        soft    nofile      1024       # Soft file limit
EOF
    
    # 3. systemd system configuration for RHEL
    echo "Configuring systemd system defaults..."
    sudo mkdir -p /etc/systemd/system.conf.d/
    sudo tee /etc/systemd/system.conf.d/00-rhel-user-limits.conf << 'EOF'
[Manager]
DefaultCPUQuota=200%
DefaultMemoryMax=100G
DefaultTasksMax=500
DefaultLimitNOFILE=2048
DefaultLimitNPROC=200
DefaultLimitSTACK=8388608
# RHEL-specific settings
DefaultEnvironment="SYSTEMD_LOG_LEVEL=info"
EOF
    
    # 4. Configure logind for RHEL
    echo "Configuring systemd-logind..."
    sudo mkdir -p /etc/systemd/logind.conf.d/
    sudo tee /etc/systemd/logind.conf.d/00-rhel-user-limits.conf << 'EOF'
[Login]
UserTasksMax=500
KillUserProcesses=yes
RemoveIPC=yes
# RHEL-specific settings
HandleSuspendKey=ignore
HandleHibernateKey=ignore
HandleLidSwitch=ignore
EOF
    
    # 5. Apply SELinux contexts
    echo "Applying SELinux contexts..."
    sudo restorecon -Rv /etc/systemd/ 2>/dev/null || true
    
    # 6. Reload systemd configuration
    echo "Reloading systemd configuration..."
    sudo systemctl daemon-reload
    sudo systemctl restart systemd-logind
    
    echo "System-wide user limits configured for RHEL $RHEL_VERSION"
}

set_user_override_rhel() {
    local username=$1
    local cpu_quota=$2
    local memory_max=$3
    local tasks_max=$4
    
    if [ -z "$username" ] || [ -z "$cpu_quota" ] || [ -z "$memory_max" ] || [ -z "$tasks_max" ]; then
        echo "Usage: set_user_override_rhel <username> <cpu_quota> <memory_max> <tasks_max>"
        echo "Example: set_user_override_rhel admin 800% 500G 2000"
        return 1
    fi
    
    # Verify user exists
    local user_id
    user_id=$(id -u "$username" 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "Error: User $username not found"
        return 1
    fi
    
    echo "Setting RHEL-optimized override limits for user: $username (UID: $user_id)"
    
    # Create systemd override with RHEL-specific settings
    sudo mkdir -p "/etc/systemd/system/user-${user_id}.slice.d/"
    sudo tee "/etc/systemd/system/user-${user_id}.slice.d/00-rhel-override.conf" << EOF
[Slice]
CPUQuota=${cpu_quota}
MemoryMax=${memory_max}
TasksMax=${tasks_max}
# RHEL-specific accounting and tuning
CPUAccounting=true
MemoryAccounting=true
TasksAccounting=true
IOAccounting=true
# RHEL cgroup settings
Slice=user.slice
EOF
    
    # PAM limits override
    echo "Setting PAM limits override..."
    sudo sed -i "/^${username}[[:space:]]/d" /etc/security/limits.conf
    
    local memory_kb=$((${memory_max%G} * 1024 * 1024))
    local cpu_minutes=$((${cpu_quota%\%} * 60 / 100))
    
    sudo tee -a /etc/security/limits.conf << EOF
# RHEL override for user: $username
$username    hard    cpu         ${cpu_minutes}
$username    hard    rss         ${memory_kb}
$username    hard    nproc       ${tasks_max}
$username    hard    nofile      4096
$username    hard    stack       16384
EOF
    
    # Apply SELinux contexts
    sudo restorecon -Rv "/etc/systemd/system/user-${user_id}.slice.d/" 2>/dev/null || true
    
    # Reload systemd
    sudo systemctl daemon-reload
    
    echo "RHEL-optimized override limits set for $username:"
    echo "- CPU: $cpu_quota"
    echo "- Memory: $memory_max" 
    echo "- Tasks: $tasks_max"
    echo "User needs to log out and back in for changes to take effect."
}

verify_rhel_limits() {
    echo "=== RHEL Resource Limits Verification ==="
    
    # Check systemd configuration
    echo "1. systemd user slice defaults:"
    if [ -f /etc/systemd/system/user-.slice.d/00-rhel-resources.conf ]; then
        grep -E "(CPUQuota|MemoryMax|TasksMax)" /etc/systemd/system/user-.slice.d/00-rhel-resources.conf
    else
        echo "   No default user slice configuration found"
    fi
    
    # Check active user slices
    echo ""
    echo "2. Active user slices:"
    systemctl list-units --type=slice --state=active | grep user- | while read slice rest; do
        echo "   $slice:"
        systemctl show "$slice" 2>/dev/null | grep -E "(CPUQuota|MemoryMax|TasksMax)" | sed 's/^/      /'
    done
    
    # Check PAM limits
    echo ""
    echo "3. PAM limits configuration:"
    tail -20 /etc/security/limits.conf | grep -v "^#" | grep -v "^$"
    
    # Check cgroups version
    echo ""
    echo "4. cgroups version and controllers:"
    if $CGROUPS_V2_ENABLED; then
        echo "   cgroups v2 enabled"
        echo "   Available controllers: $(cat /sys/fs/cgroup/cgroup.controllers 2>/dev/null || echo 'Unable to read')"
    else
        echo "   cgroups v1 in use"
    fi
    
    # Resource usage overview
    echo ""
    echo "5. Current resource usage (systemd-cgtop):"
    timeout 3 systemd-cgtop --batch -n 1 2>/dev/null | head -10 || echo "   systemd-cgtop not available"
}

enable_rhel8_cgroups_v2() {
    if [ "$RHEL_VERSION" -ne 8 ]; then
        echo "This function is only for RHEL 8. RHEL 9 uses cgroups v2 by default."
        return 1
    fi
    
    echo "Enabling cgroups v2 on RHEL 8..."
    echo "This will require a system reboot."
    
    read -p "Do you want to proceed? (y/N): " confirm
    if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
        echo "Aborted."
        return 0
    fi
    
    # Enable cgroups v2
    sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
    
    echo "cgroups v2 will be enabled after reboot."
    echo "Please reboot the system and run this script again to verify."
}

# Main menu
case "$1" in
    "check")
        check_rhel_prerequisites
        ;;
    "setup")
        check_rhel_prerequisites
        setup_system_wide_limits
        ;;
    "set-user")
        set_user_override_rhel "$2" "$3" "$4" "$5"
        ;;
    "verify")
        verify_rhel_limits
        ;;
    "enable-cgroups-v2")
        enable_rhel8_cgroups_v2
        ;;
    *)
        echo "RHEL Resource Management Script"
        echo "Usage: $0 {check|setup|set-user|verify|enable-cgroups-v2}"
        echo ""
        echo "Commands:"
        echo "  check                                    - Check RHEL prerequisites"
        echo "  setup                                   - Setup system-wide limits"
        echo "  set-user <user> <cpu> <mem> <tasks>     - Set user-specific override"
        echo "  verify                                  - Verify current configuration"
        echo "  enable-cgroups-v2                      - Enable cgroups v2 (RHEL 8 only)"
        echo ""
        echo "Examples:"
        echo "  $0 check"
        echo "  $0 setup"
        echo "  $0 set-user admin 800% 500G 2000"
        echo "  $0 verify"
        ;;
esac
```

## **RHEL-Specific Package Management**

Ensure required packages are installed:

```bash
# RHEL 8/9 package installation
sudo dnf install -y systemd quota util-linux

# For older RHEL 8 versions, you might need:
sudo dnf install -y systemd-container

# Enable and start quota services if using disk quotas
sudo systemctl enable quotaon
```

## **RHEL Subscription Manager Considerations**

If using resource limits in containerized environments:

```bash
# Check subscription status
sudo subscription-manager status

# For RHEL with container tools
sudo dnf module install -y container-tools
```

## **RHEL-Specific Monitoring**

RHEL includes specific monitoring tools:

```bash
# RHEL-specific performance monitoring
sudo dnf install -y sysstat iotop htop

# Use RHEL's built-in monitoring
systemctl status systemd-machined
systemctl status systemd-logind

# RHEL 9 specific: Check systemd version
systemctl --version
```

## **Firewall and Network Considerations**

RHEL uses firewalld by default:

```bash
# Ensure firewall doesn't interfere with resource monitoring
sudo firewall-cmd --list-all

# If using remote monitoring tools
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

## **Key RHEL-Specific Best Practices:**

1. **Always check cgroups version** - RHEL 8 defaults to v1, RHEL 9 to v2
2. **Consider SELinux implications** - Resource management files need proper contexts
3. **Use DNF package manager** - Not YUM (though YUM is aliased)
4. **Test in RHEL environment** - Some distributions handle systemd differently
5. **Check RHEL documentation** - Red Hat has specific tuning guides
6. **Use Red Hat support** - If you have a subscription, leverage official support
7. **Consider RHEL-specific tuning profiles** - Use `tuned` for system optimization

## **RHEL Version Differences:**

| Feature         | RHEL 8    | RHEL 9    |
| --------------- | --------- | --------- |
| Default cgroups | v1        | v2        |
| systemd version | 239+      | 250+      |
| Default shell   | bash 4.4  | bash 5.1  |
| SELinux         | Enforcing | Enforcing |
| Package manager | DNF       | DNF       |

The resource management strategies work well on both RHEL 8 and 9, but pay special attention to the cgroups version and ensure you're using the RHEL-optimized script above for best results.


