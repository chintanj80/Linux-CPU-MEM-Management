# System-wide CPU and memory quotas that apply to all users

To set system-wide CPU and memory quotas that apply to all users, you have several approaches. Here are the most effective methods:

## 1. **systemd Default User Slice Configuration (Recommended)**

Create a default configuration that applies to all user slices:

```bash
# Create a drop-in directory for user slice defaults
sudo mkdir -p /etc/systemd/system/user-.slice.d/

# Create the resource limits configuration
sudo tee /etc/systemd/system/user-.slice.d/resources.conf << EOF
[Slice]
CPUQuota=200%
MemoryMax=100G
TasksMax=500
EOF

# Reload systemd configuration
sudo systemctl daemon-reload
```

This configuration will apply to all user slices (`user-1001.slice`, `user-1002.slice`, etc.).



## **2. PAM Limits for All Users**

Set system-wide limits in `/etc/security/limits.conf`:

```bash
sudo tee -a /etc/security/limits.conf << EOF
# System-wide user limits
*        hard    cpu         120        # CPU time in minutes
*        hard    rss         104857600  # 100GB in KB
*        hard    nproc       200        # Max processes per user
*        hard    nofile      2048       # Max open files
*        soft    cpu         60         # Soft CPU limit
*        soft    rss         52428800   # 50GB soft memory limit
*        soft    nproc       100        # Soft process limit
EOF
```



## **3. cgroups v2 System-wide Configuration**

Create a system-wide cgroup configuration:

```bash
#!/bin/bash
# system_wide_limits.sh

# Create the configuration directory
sudo mkdir -p /etc/systemd/system.conf.d/

# Create system-wide user limits
sudo tee /etc/systemd/system.conf.d/user-limits.conf << EOF
[Manager]
DefaultCPUQuota=200%
DefaultMemoryMax=100G
DefaultTasksMax=500
EOF

# Also configure logind for user sessions
sudo tee /etc/systemd/logind.conf.d/user-limits.conf << EOF
[Login]
UserTasksMax=500
EOF

# Reload systemd
sudo systemctl daemon-reload
sudo systemctl restart systemd-logind
```

## 4. Complete Setup Script

Here's a comprehensive script to set up system-wide limits:

```bash
#!/bin/bash
# setup_system_wide_limits.sh

echo "Setting up system-wide user resource limits..."

# 1. systemd user slice defaults
echo "Configuring systemd user slice defaults..."
sudo mkdir -p /etc/systemd/system/user-.slice.d/
sudo tee /etc/systemd/system/user-.slice.d/resources.conf << EOF
[Slice]
CPUQuota=200%
MemoryMax=100G
TasksMax=500
IOWeight=100
EOF

# 2. PAM limits
echo "Configuring PAM limits..."
sudo tee -a /etc/security/limits.conf << EOF

# System-wide user resource limits
*        hard    cpu         120        # CPU time in minutes  
*        hard    rss         104857600  # 100GB RAM in KB
*        hard    nproc       200        # Max processes
*        hard    nofile      2048       # Max open files
*        hard    fsize       unlimited  # Max file size
*        soft    cpu         60         # Soft CPU time limit
*        soft    rss         52428800   # 50GB soft RAM limit
*        soft    nproc       100        # Soft process limit
*        soft    nofile      1024       # Soft file limit
EOF

# 3. systemd system configuration
echo "Configuring systemd system defaults..."
sudo mkdir -p /etc/systemd/system.conf.d/
sudo tee /etc/systemd/system.conf.d/user-limits.conf << EOF
[Manager]
DefaultCPUQuota=200%
DefaultMemoryMax=100G
DefaultTasksMax=500
DefaultLimitNOFILE=2048
DefaultLimitNPROC=200
EOF

# 4. Configure logind
echo "Configuring systemd-logind..."
sudo mkdir -p /etc/systemd/logind.conf.d/
sudo tee /etc/systemd/logind.conf.d/user-limits.conf << EOF
[Login]
UserTasksMax=500
KillUserProcesses=yes
EOF

# 5. Reload systemd configuration
echo "Reloading systemd configuration..."
sudo systemctl daemon-reload
sudo systemctl restart systemd-logind

echo "System-wide user limits configured successfully!"
echo ""
echo "Limits set:"
echo "- CPU: Maximum 2 cores (200%) per user"
echo "- Memory: Maximum 100GB per user" 
echo "- Tasks: Maximum 500 processes per user"
echo "- Files: Maximum 2048 open files per user"
echo ""
echo "Users need to log out and back in for limits to take effect."
```

## **5. Verification and Monitoring**

Check that limits are applied:

```bash
# Check systemd user slice limits
systemctl show user-$(id -u).slice | grep -E "(CPU|Memory|Tasks)"

# Check current resource usage for all users
sudo systemd-cgtop

# Check PAM limits for current user
ulimit -a

# Monitor specific user
systemctl status user-1001.slice

# Check memory usage by user slice
sudo systemd-cgtop | grep user-
```

## **6. Handle Existing Users**

Apply limits to users already logged in:

```bash
#!/bin/bash
# apply_to_existing_users.sh

echo "Applying limits to existing user sessions..."

# Get all active user slices
for slice in $(systemctl list-units --type=slice --state=active | grep "user-" | awk '{print $1}'); do
    echo "Applying limits to $slice..."
    sudo systemctl set-property "$slice" \
        CPUQuota=200% \
        MemoryMax=100G \
        TasksMax=500
done

echo "Limits applied to existing user sessions."
echo "Note: Users may need to restart their applications."
```

## Exception Handling

Create exceptions for specific users or groups:

```bash
# Allow admin users higher limits
sudo mkdir -p /etc/systemd/system/user-0.slice.d/
sudo tee /etc/systemd/system/user-0.slice.d/resources.conf << EOF
[Slice]
CPUQuota=800%
MemoryMax=500G
TasksMax=2000
EOF

# Or in PAM limits for specific users
echo "admin        hard    rss         524288000  # 500GB for admin" | sudo tee -a /etc/security/limits.conf
```

#### **Override systemd User Slice Limits**

Create user-specific configurations that override the default:

```bash
# Override limits for user with UID 1001 (higher limits)
sudo mkdir -p /etc/systemd/system/user-1001.slice.d/
sudo tee /etc/systemd/system/user-1001.slice.d/override.conf << EOF
[Slice]
CPUQuota=800%          # 8 CPU cores instead of 2
MemoryMax=500G         # 500GB instead of 100GB
TasksMax=2000          # 2000 tasks instead of 500
EOF

# Lower limits for a specific user (UID 1002)
sudo mkdir -p /etc/systemd/system/user-1002.slice.d/
sudo tee /etc/systemd/system/user-1002.slice.d/override.conf << EOF
[Slice]
CPUQuota=50%           # 0.5 CPU cores
MemoryMax=10G          # 10GB only
TasksMax=100           # 100 tasks only
EOF

# Apply the changes
sudo systemctl daemon-reload
```

#### **Override PAM Limits for Specific Users**

In `/etc/security/limits.conf`, specific user entries override wildcard entries: 

```bash
sudo tee -a /etc/security/limits.conf << EOF
# System-wide defaults (applied first)
*        hard    cpu         120
*        hard    rss         104857600  # 100GB
*        hard    nproc       200

# User-specific overrides (applied after wildcards)
admin    hard    cpu         unlimited  # No CPU time limit
admin    hard    rss         524288000  # 500GB
admin    hard    nproc       1000       # More processes

john     hard    cpu         30         # Stricter limit
john     hard    rss         10485760   # 10GB only
john     hard    nproc       50         # Fewer processes

# Group overrides
@developers hard rss         209715200  # 200GB for developers group
@guests     hard rss         5242880    # 5GB for guests
EOF
```

## **Complete Override Management Script**

Here's a script to manage user-specific overrides:

```bash
#!/bin/bash
# manage_user_overrides.sh

set_user_override() {
    local username=$1
    local cpu_quota=$2
    local memory_max=$3
    local tasks_max=$4
    
    if [ -z "$username" ]; then
        echo "Usage: set_user_override <username> <cpu_quota> <memory_max> <tasks_max>"
        return 1
    fi
    
    # Get user ID
    local user_id=$(id -u "$username" 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "User $username not found"
        return 1
    fi
    
    echo "Setting override limits for user: $username (UID: $user_id)"
    
    # Create systemd override
    sudo mkdir -p "/etc/systemd/system/user-${user_id}.slice.d/"
    sudo tee "/etc/systemd/system/user-${user_id}.slice.d/override.conf" << EOF
[Slice]
CPUQuota=${cpu_quota}
MemoryMax=${memory_max}
TasksMax=${tasks_max}
EOF
    
    # Add PAM limits override
    # First remove any existing entries for this user
    sudo sed -i "/^${username}[[:space:]]/d" /etc/security/limits.conf
    
    # Add new entries
    local memory_kb=$((${memory_max%G} * 1024 * 1024))
    sudo tee -a /etc/security/limits.conf << EOF
$username    hard    cpu         $((${cpu_quota%\%} * 60 / 100))
$username    hard    rss         ${memory_kb}
$username    hard    nproc       ${tasks_max}
$username    hard    nofile      4096
EOF
    
    # Reload systemd
    sudo systemctl daemon-reload
    
    echo "Override limits set for $username:"
    echo "- CPU: $cpu_quota"
    echo "- Memory: $memory_max" 
    echo "- Tasks: $tasks_max"
    echo "User needs to log out and back in for changes to take effect."
}

remove_user_override() {
    local username=$1
    
    if [ -z "$username" ]; then
        echo "Usage: remove_user_override <username>"
        return 1
    fi
    
    local user_id=$(id -u "$username" 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "User $username not found"
        return 1
    fi
    
    echo "Removing override limits for user: $username"
    
    # Remove systemd override
    sudo rm -rf "/etc/systemd/system/user-${user_id}.slice.d/"
    
    # Remove PAM limits
    sudo sed -i "/^${username}[[:space:]]/d" /etc/security/limits.conf
    
    sudo systemctl daemon-reload
    
    echo "Override limits removed for $username. System defaults will now apply."
}

list_user_overrides() {
    echo "=== systemd User Slice Overrides ==="
    for override_dir in /etc/systemd/system/user-*.slice.d/; do
        if [ -d "$override_dir" ]; then
            slice_name=$(basename "${override_dir%.d}")
            echo "Override for $slice_name:"
            if [ -f "$override_dir/override.conf" ]; then
                grep -E "(CPUQuota|MemoryMax|TasksMax)" "$override_dir/override.conf" | sed 's/^/  /'
            fi
            echo ""
        fi
    done
    
    echo "=== PAM Limits Overrides ==="
    echo "User-specific entries in /etc/security/limits.conf:"
    grep -E "^[^*#@]" /etc/security/limits.conf | grep -v "^$" | sed 's/^/  /'
}

# Usage examples
case "$1" in
    "set")
        set_user_override "$2" "$3" "$4" "$5"
        ;;
    "remove") 
        remove_user_override "$2"
        ;;
    "list")
        list_user_overrides
        ;;
    *)
        echo "Usage: $0 {set|remove|list}"
        echo ""
        echo "Examples:"
        echo "  $0 set admin 800% 500G 2000     # Give admin 8 cores, 500GB RAM"
        echo "  $0 set guest 50% 5G 100         # Limit guest to 0.5 core, 5GB RAM"
        echo "  $0 remove john                  # Remove overrides for john"
        echo "  $0 list                         # List all current overrides"
        ;;
esac
```

## **Group-Based Overrides**

Create overrides based on user groups:

```bash
# In /etc/security/limits.conf
@admin       hard    rss         524288000  # 500GB for admin group
@developers  hard    rss         209715200  # 200GB for developers
@interns     hard    rss         20971520   # 20GB for interns

# For systemd, you can create template units
sudo mkdir -p /etc/systemd/system/user@.service.d/
sudo tee /etc/systemd/system/user@.service.d/resource-limits.conf << EOF
[Service]
# This will be applied to all user sessions
# Individual user overrides will still take precedence
EOF
```

## Dynamic Override Application

Apply overrides to currently running user sessions:

```bash
#!/bin/bash
# apply_override_live.sh

apply_live_override() {
    local username=$1
    local cpu_quota=$2
    local memory_max=$3
    local tasks_max=$4
    
    local user_id=$(id -u "$username" 2>/dev/null)
    if [ $? -ne 0 ]; then
        echo "User $username not found"
        return 1
    fi
    
    # Apply to currently running user slice
    if systemctl is-active --quiet "user-${user_id}.slice"; then
        echo "Applying live override to user-${user_id}.slice..."
        sudo systemctl set-property "user-${user_id}.slice" \
            CPUQuota="$cpu_quota" \
            MemoryMax="$memory_max" \
            TasksMax="$tasks_max"
        
        echo "Live override applied. Changes are immediate but temporary."
        echo "Use the management script to make changes persistent."
    else
        echo "User $username is not currently logged in."
    fi
}

# Example usage
apply_live_override admin 800% 500G 2000
```

### Verification

Check that overrides are working:

```bash
# Check specific user slice
systemctl show user-1001.slice | grep -E "(CPUQuota|MemoryMax|TasksMax)"

# Check PAM limits for a specific user
sudo -u admin bash -c 'ulimit -a'

# List all user slices and their limits
for slice in $(systemctl list-units --type=slice | grep user- | awk '{print $1}'); do
    echo "=== $slice ==="
    systemctl show "$slice" | grep -E "(CPUQuota|MemoryMax|TasksMax)"
    echo ""
done
```

## Key Points:

- **Order matters**: Specific user configs override system defaults
- **Both systemd AND PAM**: Override both for complete coverage
- **Live application**: Use `systemctl set-property` for immediate changes
- **Persistence**: Create config files for permanent overrides
- **Group-based**: Use groups in PAM limits for easier management
- **Verification**: Always test that overrides are working correctly

The override system is very flexible - you can give privileged users more resources while restricting others even further than the system defaults.

## Key Points:

- **systemd approach** is most effective for modern Linux systems
- **Users must log out/in** for some limits to take effect
- **Monitor regularly** with `systemd-cgtop` and `systemctl status`
- **Test thoroughly** before deploying in production
- **Document exceptions** for administrative users
- **Combine approaches** for comprehensive coverage

The systemd user slice method is generally the most reliable for enforcing these limits system-wide on modern Linux distributions.


