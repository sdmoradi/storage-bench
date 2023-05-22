# storage-benchmark in kubernetes
### FIO-BASED STORAGE BENCHMARK
* This job can be used to run storage benchmark tests using the popular I/O tool fio.
* The image used is alpine-based & is inspired by sotoaster/dbench
* Consists of pre-defined fio templates, with the job params tuned as per standard storage performance tests (used by the community)

### WHAT DOES IT RUN

#### The job runs in three modes, defined by the DBENCH_TYPE ENV, namely:

* quick: Runs random workload (read & write tests)
* detailed: Runs the latency-oriented (random)(read & write), mixed profiles & sequential workloads (read & write)
* custom: Runs a one-off fio job with the params provided by the user
* In both the quick & detailed mode, some of the default params can be tuned/overridden, while custom allows for a complete user-defined profile.

### HOW TO RUN IT

```angular2html
saeed@ThinkBook# git clone https://github.com/sdmoradi/storage-bench.git

saeed@ThinkBook# cd storage-bench
```

* Ensure the storageclasses is already installed.
* Create an PVC/PV of desired storageclass beforehand
* Edit storageclass name in dbech-pvc.yaml file
```angular2html
saeed@ThinkBook# kubectl create -f dbench-pvc.yaml
```

* Deploy the fio job manifest

```angular2html
saeed@ThinkBook# kubectl create -f fio-job.yaml
```

### HOW TO CHECK PROGRESS & VIEW RESULTS

* Check the logs of the ongoing/completed job
```
saeed@ThinkBook# kubectl logs -f dbench-njlxh-bslsv

Working dir: /data

Testing Read IOPS...
```

* In case of quick/detailed job types, the fio results are parsed and summary provided:
```angular2html
All tests complete.

==================
= Dbench Summary =
==================
Random Read/Write IOPS: 14.3k/728. BW: 595MiB/s / 81.5MiB/s
Average Latency (usec) Read/Write: 860.90/5126.41
Sequential Read/Write: 547MiB/s / 252MiB/s
Mixed Random Read/Write IOPS: 2371/785
```