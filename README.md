# openshift-local-gpu-passthrough
1. Install Host OS
2. Setup Openshift Local 
3. Prepare Host for GPU Passthrough
4. Passthrough GPU to Openshift Local CRC VM
5. Install and Run NVIDIA GPU Operator

## 1. Install Host OS

First, choose a Host OS to install. This guide will be based on Fedora Workstation 39. The process is similar and tested for other RHEL-based OS, such as AlmaLinux. However, for other RHEL-based OS, some packages are not pre-installed and therefore requires additional mirroring.

Note that for an air-gap setup, this guide will use an internet connected host to mirror files. Using the same user account and OS for both hosts is assumed for this guide. Steps specifically for air-gap installations will be included at the end of each step. 

Download the .iso for Fedora workstation and provision for both Internet and Air-Gapped hosts. 

Fedora Workstation:

[Get Fedora Workstation](https://fedoraproject.org/workstation/)

## 2. Setup Openshift Local

Openshift Local can be setup easily on the Internet host, following the steps here:

[Get Openshift Local](https://console.redhat.com/openshift/create/local)

In short, just download the binary, `mv`or `cp` the extracted `crc` binary file into $PATH, which should be `usr/local/bin`. 

```
sudo mv crc /usr/local/bin
```

Next, in the internet connected host, run the following commands:

```
crc setup
```

```
crc start
```

When running `crc start`, enter the downloaded pull secret into the terminal when prompted. 

Keep the pull secret to reuse in the air-gapped host if needed.

Wait for the commands to finish, and Openshift Local installation is complete.

The credentials for the kubeadmin and developer accounts are printed in the output of `crc start` command. Generally, we will access the cluster through a web-based console, logging in as kubeadmin. You can start the console in Firefox by running:

```
crc console
```

To use CLI to interact with Openshift local through kubeapi, you can enable `oc` commands by running:

```
eval $(crc oc-env)
```

Optionally, you can copy the `oc` binary into $PATH for persistence.

When the web-based console is available, Openshift Local is installed successfully on the host.

### For Fully Air-Gapped Hosts:

Copy these files from the internet mirror into a removable media:
- crc binary
- pull secret
- ~/.crc directory (see below for rsync steps)

Transfer the crc binary into the $PATH (`/usr/local/bin`) of the air-gapped host.

Copy the pull secret anywhere that is accessible to be copy and pasted later.

Running `crc start` in the air-gapped host directly will fail as it will attempt to download the required files from the Internet when it is not already present. Hence, we will copy the files downloaded on Internet host before setting up Openshift Local.

To copy over downloaded files from the Internet connected host to the air-gapped hosts, use the following `rsync` command to copy the `~/.crc` folder into a removable media, and from the removable media into the same directory on the air-gapped host  (`~/` or `$HOME/` ). 

`rsync` is needed to copy as there are symbolic links, which will fail when using `cp`.  Use the following rsync command:

```
sudo rsync -av --copy-links --no-perms --no-owner --no-group --exclude='crc-http.sock' ~/.crc <path to your external media to transfer the files>
```

Do note that the `.crc` is hidden. We can check if the files are present using `ls -a` from it's parent directory.

From the removable media, we can copy the files into the air-gapped host, likewise using `rsync`:

```
sudo rsync -av --copy-links --no-perms --no-owner --no-group  <path to your external media hosting>/.crc ~/
```

Once the files are copied, we need to change the ownership of the files:

```
sudo chown -R <your local user>:<your local user> ~/.crc
```

After which, running `crc setup` will run properly. 

Before running `crc start` delete the cached crc instance by running:

```
crc delete
```

Finally, running `crc start` should run smoothly and you should be able to access the cluster, same as the Internet-host.

### Configuring Openshift Local:

Once Openshift Local is accessible in the desired host(s), we can configure the resources allocated to it. First stop the crc instance:

```
crc stop
```

Then run the following commands to configure the cpus, memory, and storage respectively:

```
crc config set disk-size <number of GiB>
```

```
crc config set cpus <number of vCPUs>
```

```
crc config set memory <memory in MiB>
```

To check if configured correctly:
```
crc config view
```

Min number for memory is 9216.

## 3. Setup the Host for GPU Passthrough

Check if IOMMU is enabled on the desired host:
```
sudo virt-host-validate
```

```
QEMU: Checking for hardware virtualization                                 : PASS
QEMU: Checking if device /dev/kvm exists                                   : PASS
QEMU: Checking if device /dev/kvm is accessible                            : PASS
QEMU: Checking if device /dev/vhost-net exists                             : PASS
QEMU: Checking if device /dev/net/tun exists                               : PASS
QEMU: Checking for cgroup 'cpu' controller support                         : PASS
QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
QEMU: Checking for cgroup 'memory' controller support                      : PASS
QEMU: Checking for cgroup 'devices' controller support                     : PASS
QEMU: Checking for cgroup 'blkio' controller support                       : PASS
QEMU: Checking for device assignment IOMMU support                         : PASS
QEMU: Checking if IOMMU is enabled by kernel                               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
```
We need IOMMU to be enabled. IOMMU tends to be enabled by default in AMD processors but not Intel processes.

To confirm the status of IOMMU, you can also look the /sys/class/iommu/ is populated, if empty, IOMMU is not enabled:

```
sudo find /sys | grep dmar | head -10
```

If there is no output, IOMMU is not enabled.

Next, we can enable IOMMU and disable the Nouveau driver by adding arguments to the kernel.

Before we make the changes, we can check the pre-existing kernel arguments by running:

```
sudo grubby --info=DEFAULT
```

Output should look similiar to this:
```
index=0
kernel="/boot/vmlinuz-6.8.6-200.fc39.x86_64"
args="ro rootflags=subvol=root rhgb quiet amd_iommu=on"
root="UUID=b50d0db6-f6f9-4a04-92ee-59e0b7a7727f"
initrd="/boot/initramfs-6.8.6-200.fc39.x86_64.img"
title="Fedora Linux (6.8.6-200.fc39.x86_64) 39 (Workstation Edition)"
id="1bf6083ed0104139a7cd628f5754dcb3-6.8.6-200.fc39.x86_64"
```

To blacklist Noveau and enable IOMMU, run the following command for Intel systems:

```
sudo grubby --args="rd.driver.blacklist=nouveau intel_iommu=on iommu=pt" --update-kernel DEFAULT
```

Or, for AMD:

```
sudo grubby --args="rd.driver.blacklist=nouveau amd_iommu=on iommu=pt" --update-kernel DEFAULT
```

Check the kernel arguments again to see if they were appended properly by re-running:
```
sudo grubby --info=DEFAULT
```

Create the `/usr/lib/modprobe.d/blacklist-nouveau.conf` file and add the following information to the file:
blacklist nouveau

```
options nouveau modeset=0
```

Re-generate initramfs.
```
$ sudo dracut --force
```
Reboot the machine.

After rebooting, check if IOMMU is enabled by re-running:

```
sudo virt-host-validate
```

This time round, output should be something like:

```
QEMU: Checking for hardware virtualization                                 : PASS
QEMU: Checking if device /dev/kvm exists                                   : PASS
QEMU: Checking if device /dev/kvm is accessible                            : PASS
QEMU: Checking if device /dev/vhost-net exists                             : PASS
QEMU: Checking if device /dev/net/tun exists                               : PASS
QEMU: Checking for cgroup 'cpu' controller support                         : PASS
QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
QEMU: Checking for cgroup 'memory' controller support                      : PASS
QEMU: Checking for cgroup 'devices' controller support                     : PASS
QEMU: Checking for cgroup 'blkio' controller support                       : PASS
QEMU: Checking for device assignment IOMMU support                         : PASS
QEMU: Checking if IOMMU is enabled by kernel                               : PASS
```

We can also check the DMAR table again:

```
sudo find /sys | grep dmar | head -10
```

It should be populated this time round.


## 4. Passthrough the GPU to Openshift Local

Check the NVIDIA GPU on the desired host(s):
```
sudo lshw -class display -businfo
```

The output should show the pci information. For example:
```
Bus info          Device          Class          Description
============================================================
pci@0000:01:00.0                  display        GA106M [GeForce RTX 3060 Mobile
pci@0000:06:00.0                  display        Cezanne [Radeon Vega Series / R
```

From the above, we know to passthrough `pci@0000:01:00.0`.

Next, we can retrieve the XML dump of the pci card.

```
virsh nodedev-dumpxml pci_0000_01_00_0
```

Take note of the address domain information from the output. For example:
```
<device>
  <name>pci_0000_01_00_0</name>
  <path>/sys/devices/pci0000:00/0000:00:01.1/0000:01:00.0</path>
  <parent>pci_0000_00_01_1</parent>
  <driver>
    <name>vfio-pci</name>
  </driver>
  <capability type='pci'>
    <class>0x030000</class>
    <domain>0</domain>
    <bus>1</bus>
    <slot>0</slot>
    <function>0</function>
    <product id='0x2560'>GA106M [GeForce RTX 3060 Mobile / Max-Q]</product>
    <vendor id='0x10de'>NVIDIA Corporation</vendor>
    <iommuGroup number='10'>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
    </iommuGroup>
  </capability>
</device>
```

Also take note of the driver section of the ouput. It should list `vfio-pci` instead of `noveau`. 

We can focus on the address domain from the above, in this snippet:
```
<iommuGroup number='10'>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
</iommuGroup>
```

Now we will detach the GPU from the host so that it can be attached to Openshift local vm.

```
sudo virsh nodedev-dettach pci_0000_01_00_0
```

After detaching, make sure Openshift Local is not active by running `crc stop`.

Assign the GPU to Openshift Local by running:
```
sudo virsh edit crc
```

Enter the address information under `<devices> ... </devices>` like so:

```
...
<devices>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x00'/>
    </source>
  </hostdev>
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      <address domain='0x0000' bus='0x01' slot='0x00' function='0x01'/>
    </source>
  </hostdev>
...
</devices>
```


You should have a warning, type `i`:
```
sudo virsh edit crc
setlocale: No such file or directory
error: XML document failed to validate against schema: Unable to validate doc against /usr/share/libvirt/schemas/domain.rng
Extra element devices in interleave
Element domain failed to validate content

Failed. Try again? [y,n,i,f,?]: 
y - yes, start editor again
n - no, throw away my changes
i - turn off validation and try to redefine again
f - force, try to redefine again
? - print this help
Failed. Try again? [y,n,i,f,?]: i
Domain crc XML configuration edited.
```

Restart the machine, and run `crc start` again. You should see the driver information as INFO upon starting crc. 

You can also verify that by running `oc get node`, starting a debug pod by running `oc debug node/<insert the node id eg.crc-w6th5-master-0>`. This starts a terminal, which you can run `lspci -nn | grep -i nvidia` to check on the gpu.

## 5. Install and Run the GPU Operator

### For Fully Air-Gapped Hosts:

Steps are required to air-gap the operator images into a mirror registry, and to configure Openshift Local to download the required images from the mirror registry.

Guide for this setup  will be included in the near future.

### Installing the Operator:

Login to the cluster console, by running `crc console` and logging in as `kubeadmin`. 

From the `Administrator` view, click Operators > OperatorHub. Search and install the `NVIDIA GPU Operator`.

Once installed, go to Operators > Installed Operators. Click on `ClusterPolicy` tab and create a ClusterPolicy.  The default values are sufficient to start the driver.

Verify clicking Workloads > Pods. Select the Pod with the prefix `nvidia-driver-daemonset`. Go to it's terminal and run `nvidia-smi` to verify the output.
