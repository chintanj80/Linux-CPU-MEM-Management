# CPU Quota Management in Linux - RHEL

## Method 1 - taskset

`taskset` is a Linux command-line utility that allows you to view and set the CPU affinity of processes. CPU affinity determines which CPU cores a process is allowed to run on in a multi-core system.

```bash
# View CPU affinity:
taskset -p <pid>
# Check current CPU affinity of process with PID 1234
taskset -p 1234


# Set CPU affinity
taskset -p <cpu_mask> <command>
# Run a command on CPU cores 0 and 1 only
taskset -c 0,1 my_program
# Pin an existing process to CPU core 2
taskset -p -c 2 1234
# Run a CPU-intensive task on cores 4-7
taskset -c 4-7 ./cpu_intensive_program
```

## Common Use Cases

- **Performance optimization:** Pin CPU-intensive processes to specific cores
- **NUMA optimization:** Keep processes on CPUs close to their memory
- **Real-time applications:** Isolate critical processes from system interference
- **Testing:** Control which cores are used for benchmarking
- **Resource management:** Prevent processes from using all available CPUs

The `taskset` command is particularly useful in high-performance computing environments where you need fine control over CPU resource allocation.



## Method 2 - cgroups v2 with systemd

Modern Linux systems use cgroups v2, which can limit CPU usage more dynamically:

```bash
# Create a systemd scope that limits CPU usage to 2 cores worth of CPU time
systemd-run --scope -p CPUQuota=200% my_program

# Or use systemctl to set CPU limits
systemctl set-property my_service.service CPUQuota=200%


```

## Method 3 - cpulimit

This tool limits CPU usage by percentage rather than pinning to specific cores:

```bash
# Limit a process to use equivalent of 2 CPU cores (200% on multi-core)
cpulimit -l 200 my_program

# Or limit an existing process
cpulimit -p <pid> -l 200
```

## Method 4 - **cset (CPU sets)**

Part of the `cpuset` package, provides more advanced CPU management:

```bash
# Create a CPU set with 2 cores
cset set --cpu=2-3 --set=my_set

# Run process in that CPU set
cset proc --set=my_set --exec my_program
```

Method 5 - 
