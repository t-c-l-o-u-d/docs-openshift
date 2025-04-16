<!-- GNU Affero General Public License v3.0 or later (see COPYING or https://www.gnu.org/licenses/agpl.txt) -->

1. Install OpenShift Virtualization Operator  
    **TODO:**
    ```
    How-to Install Operator via CLI
      Including setup of hyperconverged
    Documentation for LVM operator installation and configuration
    ```


2. Apply a **MachineConfig** to enable IOMMU
    ```bash
    $ oc apply -f 100-master-kernel-arg-enable-iommu.yaml
    ```

  + The system should automatically reboot after a few minutes of receiving the **MachineConfig**. Once it comes online, enter a debug shell on the node to verify the change was applied.

    ```
    $ oc debug node/janus
    Starting pod/janus-debug-dqttc ...
    To use host binaries, run `chroot /host`
    Pod IP: 192.168.1.88
    If you don't see a command prompt, try pressing enter.
    
    sh-4.4# chroot /host
    sh-4.4# dmesg | grep -i -e DMAR -e IOMMU
    ```
    **Note:** You should see several lines of output and most importantly `AMD-Vi: AMD IOMMUv2 loaded and initialized`.

  + Also verify that the kernel arguments were updated by looking for `amd_iommu=on`.

    ```
    sh-4.4# cat /proc/cmdline | grep --color "amd_iommu=on"
      BOOT_IMAGE=(hd1,gpt3)/ostree/rhcos-fb69c95d7920e1368d0ce7bfd0ecd59229b63d56e362614b980497a3540f9fa3/vmlinuz-5.14.0-284.43.1.el9_2.x86_64 ignition.platform.id=metal ostree=/ostree/boot.0/rhcos/fb69c95d7920e1368d0ce7bfd0ecd59229b63d56e362614b980497a3540f9fa3/0 ip=eno1:dhcp root=UUID=e48b13f5-dc49-4368-b943-c834ed6a2f03 rw rootflags=prjquota boot=UUID=8b59af66-455c-42c3-8628-a7fe8ad7c58e video=efifb:off amd_iommu=on systemd.unified_cgroup_hierarchy=1 cgroup_no_v1=all psi=1
    ```

  + Verify remapping is enabled.

    ```
    $ dmesg | grep --color "remapping"
    [    0.346080] AMD-Vi: Interrupt remapping enabled
    ```


3. Disable the efi framebuffer to prevent the drivers from loading before vfio.

    ```
    $ oc apply -f 100-master-kernel-arg-disable-framebuffer.yaml
    ```

  + The system will automatically reboot after ~2 minutes of receiving the machine config. Once it comes online, enter a debug shell on the node to verify the change was applied.

    ```
    $ oc debug node/janus
    Starting pod/janus-debug-dqttc ...
    To use host binaries, run `chroot /host`
    Pod IP: 192.168.1.88
    If you don't see a command prompt, try pressing enter.

    sh-4.4# chroot /host
    sh-4.4# cat /proc/cmdline | grep --color "video=efifb:off"
      BOOT_IMAGE=(hd1,gpt3)/ostree/rhcos-fb69c95d7920e1368d0ce7bfd0ecd59229b63d56e362614b980497a3540f9fa3/vmlinuz-5.14.0-284.43.1.el9_2.x86_64 ignition.platform.id=metal ostree=/ostree/boot.0/rhcos/fb69c95d7920e1368d0ce7bfd0ecd59229b63d56e362614b980497a3540f9fa3/0 ip=eno1:dhcp root=UUID=e48b13f5-dc49-4368-b943-c834ed6a2f03 rw rootflags=prjquota boot=UUID=8b59af66-455c-42c3-8628-a7fe8ad7c58e video=efifb:off amd_iommu=on systemd.unified_cgroup_hierarchy=1 cgroup_no_v1=all psi=1
    ```

    **Note:** You should see `video=efifb:off` in the output as well as the monitor(s) will be blank plugged into the node.


4. Generate a yaml file via butane for the PCI devices

    ```
    $ butane 100-master-vfio-pci.bu -o 100-master-vfio-pci.yaml
    ```

5. Change the driver for the PCI ids to use the vfio-pci driver

    ```
    $ oc apply -f 100-master-vfio-pci.yaml
    ```

  + The system will automatically reboot after ~2 minutes of receiving the machine config. Once it comes online, enter a debug shell on the node to verify the change was applied.

    ```
    $ oc debug node/janus
      Starting pod/janus-debug-cltfh ...
      To use host binaries, run `chroot /host`
      Pod IP: 192.168.1.88
      If you don't see a command prompt, try pressing enter.
    sh-4.4# chroot /host
    sh-5.1# cat /etc/modules-load.d/vfio-pci.conf
    vfio-pci
    ```

6. Modify HyperConvered CR to support the devices
    ```
    apiVersion: hco.kubevirt.io/v1
    kind: HyperConverged
    metadata:
      name: kubevirt-hyperconverged
      namespace: openshift-cnv
    spec:
      permittedHostDevices:
        pciHostDevices:
        - pciDeviceSelector: "1002:744c"
          resourceName: "amd.com/7900_XTX"
        - pciDeviceSelector: "1002:ab30"
          resourceName: "amd.com/7900_XTX_AUDIO"
    ```
