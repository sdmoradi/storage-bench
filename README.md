# Storage Bench

A FIO-based storage benchmarking tool for measuring storage performance in Kubernetes Persistent Volume Claims (PVCs) and Linux local storage.

Storage Bench uses FIO to measure random and sequential read/write performance, IOPS, bandwidth, and latency.

## Features

- **Kubernetes PVC benchmarking** using Kubernetes Jobs and Persistent Volume Claims.
- **Linux local storage benchmarking** using Docker Compose and host directory bind mounts.
- Multiple benchmark profiles: `quick`, `detailed`, and `custom`.
- Configurable test size, number of jobs, I/O engine, and other FIO parameters.
- Benchmark results printed directly to the terminal.
- Reuses the existing `sdmoradi/fio-bench` Docker image for both execution modes.

## Requirements

### Kubernetes mode

- A Kubernetes cluster.
- `kubectl` configured to access the cluster.
- A StorageClass that supports provisioning the required PVC.
- Sufficient permissions to create PVCs and Jobs.

### Local mode

- Linux host.
- Docker Engine and Docker Compose v2.
- A writable directory on the storage device you want to benchmark.
- Sufficient free disk space for the benchmark test file.

Podman with Compose compatibility may also work, depending on the installed version and configuration.

## Benchmark Profiles

| Profile | Description |
|---|---|
| `quick` | Runs a shorter benchmark covering basic random read/write performance and bandwidth. |
| `detailed` | Runs a more comprehensive benchmark, including latency, sequential workloads, and mixed random workloads. |
| `custom` | Runs a user-defined FIO configuration. Review the entrypoint behavior and cleanup logic before using this profile. |

The exact workloads, block sizes, durations, and other parameters are defined by the current `docker-entrypoint.sh` implementation.

---

## 1. Kubernetes PVC Benchmark

This mode measures storage performance through a Kubernetes Persistent Volume Claim.

### Step 1: Configure the PVC

Edit `dbench-pvc.yaml` and set the StorageClass available in your cluster.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dbench
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: your-storage-class
  volumeMode: Filesystem
```

Replace `your-storage-class` with the appropriate StorageClass name.

### Step 2: Create the PVC

```bash
kubectl apply -f dbench-pvc.yaml
```

Check the PVC status:

```bash
kubectl get pvc dbench
```

Wait until the PVC is `Bound` before continuing, unless your StorageClass uses delayed volume binding.

### Step 3: Configure the benchmark Job

Edit `fio-job.yaml` to configure the benchmark environment variables.

The existing configuration includes:

```yaml
env:
  - name: DBENCH_MOUNTPOINT
    value: /data
  - name: DBENCH_TYPE
    value: detailed
  - name: FIO_SIZE
    value: 2G
  - name: FIO_DIRECT
    value: "1"
  - name: FIO_NUMJOBS
    value: "1"
  - name: FIO_OFFSET_INCREMENT
    value: 250M
  - name: FIO_IOENGINE
    value: libaio
  - name: OPTIONS
    value: ""
  - name: CUSTOM
    value: ""
```

Ensure the Job mounts the PVC at `/data` and uses the intended image:

```yaml
image: sdmoradi/fio-bench:latest
```

### Step 4: Run the benchmark

```bash
kubectl apply -f fio-job.yaml
```

Monitor the Job:

```bash
kubectl get jobs,pods
```

Follow the benchmark output:

```bash
kubectl logs -f job/fio-bench
```

Use the actual Job name defined in `fio-job.yaml` if it differs from `fio-bench`.

### Step 5: Review the results

The benchmark summary is printed in the Job logs. For example:

```text
==================
= Dbench Summary =
==================
Random Read/Write IOPS: 5009/4987. BW: 500MiB/s / 500MiB/s
Average Latency (usec) Read/Write: 734.47/798.70
Sequential Read/Write: 500MiB/s / 218MiB/s
Mixed Random Read/Write IOPS: 6009/1978
```

These values are illustrative. Actual performance depends on the storage backend, volume configuration, workload, and cluster environment.

Clean up the Job and PVC when finished:

```bash
kubectl delete -f fio-job.yaml
kubectl delete -f dbench-pvc.yaml
```

**Warning:** Deleting the PVC may lead to data loss, depending on the storage backend and reclaim policy. Only delete it when it is safe to do so.

---

## 2. Linux Local Storage Benchmark with Docker Compose

This mode benchmarks a directory on the Linux host by mounting it inside the container at `/data`.

It is useful for testing local disks, SSDs, NVMe devices, and mounted filesystems without deploying a Kubernetes Job.

### Step 1: Choose a benchmark directory

Create a dedicated directory on the filesystem you want to test.

```bash
mkdir -p /mnt/benchmark
```

Make sure `/mnt/benchmark` is actually located on the storage device you intend to measure.

For example, check the filesystem and mount point:

```bash
findmnt -T /mnt/benchmark
df -h /mnt/benchmark
```

Use an empty, dedicated test directory. Do not point the benchmark at a directory containing important data.

### Step 2: Create `docker-compose.yml`

Create a `docker-compose.yml` file with the following configuration:

```yaml
services:
  storage-bench:
    image: sdmoradi/fio-bench:latest
    container_name: storage-bench
    environment:
      BENCH_PATH: /mnt/benchmark
      DBENCH_MOUNTPOINT: /data
      DBENCH_TYPE: ${DBENCH_TYPE:-detailed}
      FIO_SIZE: ${FIO_SIZE:-2G}
      FIO_DIRECT: ${FIO_DIRECT:-1}
      FIO_NUMJOBS: ${FIO_NUMJOBS:-1}
      FIO_OFFSET_INCREMENT: ${FIO_OFFSET_INCREMENT:-250M}
      FIO_IOENGINE: ${FIO_IOENGINE:-libaio}
      OPTIONS: ${OPTIONS:-}
      CUSTOM: ${CUSTOM:-}
    volumes:
      - type: bind
        source: /mnt/benchmark
        target: /data
        bind:
          create_host_path: false
    restart: "no"
```

The `source` path is the host directory to benchmark. The container accesses it through `/data`, which is configured using `DBENCH_MOUNTPOINT`.

`BENCH_PATH` is retained in the environment for convenience, but the existing entrypoint uses `DBENCH_MOUNTPOINT` for its test-file location. The bind mount's `source` is what determines which host storage is tested.

### Step 3: Run a quick benchmark

```bash
DBENCH_TYPE=quick \
FIO_SIZE=512M \
docker compose run --rm storage-bench
```

### Step 4: Run a detailed benchmark

```bash
DBENCH_TYPE=detailed \
FIO_SIZE=2G \
docker compose run --rm storage-bench
```

The container runs the existing benchmark entrypoint and prints the summary to the terminal.

### Step 5: Save the results

To display the output and save it to a log file:

```bash
DBENCH_TYPE=detailed \
FIO_SIZE=2G \
docker compose run --rm storage-bench 2>&1 | tee benchmark.log
```

Each run uses a fresh container, and `--rm` removes the container after it exits. It does not remove files created on the host-mounted storage.

**Important:** The current custom-mode cleanup logic may delete files under the mounted directory. Do not use the `custom` profile on a directory containing important files until that behavior has been reviewed and corrected.

### Testing another directory

Update the Compose bind mount's `source` to the desired host directory:

```yaml
volumes:
  - type: bind
    source: /path/to/your/storage-test
    target: /data
    bind:
      create_host_path: false
```

Create that directory before running Compose, and ensure it resides on the intended filesystem.

---

## 3. Configuration Reference

| Variable | Default | Description |
|---|---|---|
| `DBENCH_TYPE` | `detailed` | Benchmark profile: `quick`, `detailed`, or `custom`. |
| `FIO_SIZE` | `2G` | Configured test size. |
| `FIO_DIRECT` | `1` | FIO direct-I/O setting used by the entrypoint. |
| `FIO_NUMJOBS` | `1` | Number of FIO jobs, where supported by the profile. |
| `FIO_OFFSET_INCREMENT` | `250M` | Per-job file offset increment, where used by the profile. |
| `FIO_IOENGINE` | `libaio` | FIO I/O engine used by the entrypoint. |
| `OPTIONS` | Empty | Additional options supported by the existing entrypoint. |
| `CUSTOM` | Empty | User-defined FIO arguments for custom mode. |
| `DBENCH_MOUNTPOINT` | `/data` in these examples | Directory inside the container where benchmark files are created. |

Actual behavior depends on which variables and options are consumed by the current entrypoint.

---

## 4. Comparing Kubernetes and Local Storage Results

You can use Storage Bench to compare storage performance across different environments, provided the workload settings are equivalent.

For meaningful comparisons:

1. Use the same benchmark profile.
2. Use the same FIO test size and compatible FIO parameters.
3. Compare the same workload type: random read/write, sequential read/write, or mixed I/O.
4. Run multiple tests and compare repeatable results.
5. Record the execution environment, storage type, filesystem, and container configuration.
6. Avoid running competing workloads on the storage device during benchmarking.

### Metrics

| Metric | Meaning |
|---|---|
| IOPS | Number of input/output operations per second. |
| Bandwidth | Amount of data transferred per second. |
| Latency | Time required to complete I/O operations. |
| Sequential performance | Performance for mostly contiguous I/O. |
| Mixed workload | Performance when reads and writes are performed together. |

A containerized local test and a Kubernetes PVC test do not necessarily measure the same storage layer. Kubernetes may add a storage backend, network, virtualization, or filesystem layer, depending on the environment.

Also, the same `FIO_SIZE` does not guarantee identical tests if the profiles use different block sizes, queue depths, job counts, or durations.

---

## 5. Safety and Recommendations

- Always benchmark a dedicated test directory or PVC.
- Check free disk space before running a benchmark.
- Do not benchmark directories containing important files.
- Be particularly careful with `custom` mode because its current cleanup logic may remove files from the mounted directory.
- Use consistent benchmark settings when comparing storage providers.
- Do not interpret one test run as a definitive measurement of the storage device's maximum performance.

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.
