# oci-decorator
Prestart Hook providing additional functionality to containers

Place oci-onload.cfg to /etc/oci-decorator/oci-decorator.d which 
dictates what files and devices should be available in the container.

Run cmake; make; make install


I have been working on a generic prestart hook for accelerator cards used 
in container runtimes like docker or cri-o (https://github.com/openshift-psap/oci-decorator).

The hook is now in a state where for each vendor,  a configuration file can be dropped
into /etc/oci-decorator/oci-decorator.d/ and the specific framework/feature/card is 
activated for the container. 

I have now two POCs running (1) SolarFlare and (2) NVIDIA.

Here is a sample json config for  NVIDIA (https://raw.githubusercontent.com/openshift-psap/oci-decorator/master/oci-nvidia.json)
DO NOT COPY & PASTE :) use the configs from the github repo.
```
{
    "version": "396.26",
    "log_level": "LOG_DEBUG",
    "activation_flag": "OCI_NVIDIA",
    "driver_feature": ["compute", "utility"],

    "inventory": {
        "compute": {
            "devices": ["nvidia0", "nvidiactl", "nvidia-uvm", "nvidia-uvm-tools"],
            "binaries": ["/usr/bin/nvidia-cuda-mps-control", "/usr/bin/nvidia-cuda-ms-server"],
            "libraries": ["/usr/lib64/nvidia/libcuda.so",
                          "/usr/lib64/nvidia/libcuda.so.396.26",
                          "/usr/lib64/nvidia/libnvidia-opencl.so.396.26",
                          "/usr/lib64/nvidia/libnvidia-ptxjitcompiler.so.396.26",
                          "/usr/lib64/nvidia/libnvidia-fatbinaryloader.so.396.26",
                          "/usr/lib64/nvidia/libnvidia-compiler.so.396.26"],
            "directories": ["/usr/lib64/nvidia"],
            "miscellaneous": ["/etc/ld.so.conf.d/nvidia-lib64.conf"] 
        },
        "utility": {
            "devices": ["nvidia0", "nvidiactl"],
            "binaries": ["/usr/bin/nvidia-smi", "/usr/bin/nvidia-debugdump", "/usr/bin/nvidia-presistenced"],
            "libraries": ["/usr/lib64/nvidia/libnvidia-ml.so.396.26",
                          "/usr/lib64/nvidia/libnvidia-ml.so",
                          "/usr/lib64/nvidia/libnvidia-cfg.so.396.26"],
            "directories": ["/usr/lib64/nvidia"],
            "miscellaneous": ["/etc/ld.so.conf.d/nvidia-lib64.conf"] 
        }
    }
}
```
1. How is the hook activated? See the "activation_flag" in the config? Just run

ip-172-31-25-15 oci-decorator (master*) #  docker run  -it -e OCI_NVIDIA=all centos:7 nvidia-smi 
Fri Jun 15 12:13:52 2018       
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.26                 Driver Version: 396.26                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   32C    P0    34W / 300W |    418MiB / 16160MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```
PRO:
- SELinux is of course enforcing, no need to alter the device files, library or binary files, everything is contained.
- No need to run a container or pod privileged because of some strange SELinux permission issues

2) Running a CUDA workload:
```
ip-172-31-25-15 oci-decorator (master*) # docker run  -it -e OCI_NVIDIA=all quay.io/zvonkok/cuda-9.2-centos7 /usr/local/cuda-9.2/samples/0_Simple/vectorAdd/vectorAdd   

[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```
3) Wanna run a Solarflare onload enabled containers? Drop the following https://raw.githubusercontent.com/openshift-psap/oci-decorator/master/oci-onload.json into 
the config directory
```
{
    "version": "201805",
    "log_level": "LOG_DEBUG",
    "activation_flag": "OCI_SOLARFLARE",
    "driver_feature": ["onload"],

    "inventory": {
        "onload": {
            "devices": ["onload", "onload_epoll", "sfc_char"],
            "binaries": ["/usr/bin/onload", "/usr/bin/sfnt-pingpong", "/usr/sbin/onload_tool", "/usr/lib64/libonload.so"],
            "libraries": ["/usr/lib64/libonload_ext.so",
                          "/usr/lib64/libonload_ext.so.1.1.0",
                          "/usr/lib64/libonload_zf.so",
                          "/usr/lib64/libonload_zf.so.1.1.1",
                          "/usr/lib64/libonload_zf_static.a",
                          "/usr/lib64/libsfcaffinity.so"],
            "directories": ["/usr/libexec/onload", "/usr/libexec/onload/profiles", "/usr/libexec/onload/apps"],
            "miscellaneous": ["/usr/libexec/onload/profiles/latency.opf", "/usr/libexec/onload/profiles/latency-best.opf"]
        }
    }
}
```
4) Run a onload enabled container

dell-r730-01 oci-decorator (master*) # docker run -it -e OCI_SOLARFLARE=all centos:7 onload

OpenOnload 201805
Copyright 2006-2018 Solarflare Communications, 2002-2005 Level 5 Networks
Built: Jun  8 2018 04:37:46 (release)
Kernel module: 201805

usage:
  onload [options] <command> <command-args>

options:
  -p,--profile=<profile> -- comma sep list of config profile(s)
  --force-profiles       -- profile settings override environment
  --no-app-handler       -- do not use app-specific settings
  --app=<app-name>       -- identify application to run under onload
  --version              -- print version information
  -v                     -- verbose
  -h --help              -- this help message


5) Enable NVIDIA and Solarflare for the container? Drop both configs into the config directory and 
docker run -it -e OCI_SOLARFLARE -e OCI_NVIDIA centos:7 bash 
