## Fork info

Forked because the original guide did not work completely with my setup.

My setup:
- CPU: Intel Core i5 7600K (x4)
- GPU: NVidia GTX1060 6GB (MSI)
- MB: MSI Z270 M3
  - Dual monitor option enabled, second monitor connected to the MB
- Host OS: Debian 13 rc1 + KDE Plasma + Wayland
  - Open source GPU driver `nouveau`
  - All packages up to date as of June 2025
- Guest OS: Windows 10
- Game tested: StarCraft II

-----------------

## Overview

This repository provides a comprehensive guide to setting up **GPU passthrough** on Linux systems. GPU passthrough allows a virtual machine (VM) to directly access the host's GPU, enabling high-performance graphics rendering in a VM environment. This is particularly useful for tasks like gaming, 3D rendering, and running GPU-intensive applications in a virtualized setup.

## Contents

- **Step-by-Step Setup Guides**
  - Detailed instructions for various Linux distributions
  - Configuration files and scripts
- **Troubleshooting**
  - Common issues and solutions
  - Tips for hardware compatibility
- **Performance Optimization**
  - Tweaks to maximize GPU performance
  - Best practices for virtualization settings
- **Resources**
  - Links to useful tools and external tutorials
  - Community forums and support channels

## Requirements

- **Hardware**
  - A CPU with virtualization support (Intel VT-x/AMD-V)
  - Motherboard and CPU support for IOMMU (Intel VT-d/AMD-Vi)
  - A dedicated GPU for the VM
  - Separate GPU for the host (optional but recommended)
- **Software**
  - Linux operating system with a recent kernel
  - QEMU/KVM or other virtualization software
  - Necessary drivers and firmware updates

## Getting Started

To begin setting up GPU passthrough:

### 1. **Check Hardware Compatibility**
   - **Virtualization Support**: Verify that your CPU supports virtualization (Intel VT-x/AMD-V). You can check this by running:
     ```bash
     grep -E -o '(vmx|svm)' /proc/cpuinfo
     ```
     If the output includes `vmx` (for Intel) or `svm` (for AMD), your CPU supports virtualization.
   - **IOMMU Support**: Your motherboard and CPU must support IOMMU (Intel VT-d/AMD-Vi). To verify, run the following command:
     ```bash
     sudo dmesg | grep -e DMAR -e IOMMU
     ```
     If IOMMU is supported, you should see relevant messages in the output.

### 2. **Enable Virtualization in BIOS/UEFI**
   - Reboot your system and enter BIOS/UEFI by pressing a designated key (usually `Delete`, `F2`, or `Esc`).
   - Locate the virtualization settings. This could be under a tab like `Advanced` or `CPU Configuration`.
   - Enable **Intel VT-x** or **AMD-V**, and also **Intel VT-d** or **AMD-Vi** for IOMMU.
   - Save the settings and reboot into your operating system.

### Enable IOMMU in Bootloader

To enable IOMMU, you first need to identify which bootloader your system uses. Common bootloaders include GRUB and systemd-boot. Follow the steps below to determine your bootloader and configure IOMMU accordingly.

#### Determine Your Bootloader
   - **Check Bootloader**:
     ```bash
     sudo test -e /boot/grub/grub.cfg && echo -e "\nGRUB detected" || sudo test -e /boot/loader/loader.conf && echo -e "\nsystemd-boot detected"
     ```
     > NOTE: On default Debian both are detected with this script but it ships with GRUB.

#### Enable IOMMU in GRUB

1. **Edit the GRUB Configuration**:
   - Open the GRUB configuration file for editing:
     ```bash
     sudo nano /etc/default/grub
     ```
   - Add the appropriate IOMMU settings to the end of the `GRUB_CMDLINE_LINUX_DEFAULT` line:
     - For **Intel** CPUs:
       ```bash
       GRUB_CMDLINE_LINUX_DEFAULT=".... intel_iommu=on iommu=pt"
       ```
     - For **AMD** CPUs:
       ```bash
       GRUB_CMDLINE_LINUX_DEFAULT=".... amd_iommu=on iommu=pt"
       ```

2. **Update GRUB**:
   - Save the changes and update GRUB:
     - On Ubuntu/Debian-based Distributions:
       ```bash
       sudo update-grub
       ```
     - On Arch Linux:
       ```bash
       sudo grub-mkconfig -o /boot/grub/grub.cfg
       ```
     - On Fedora:
       ```bash
       sudo grub2-mkconfig -o /boot/grub2/grub.cfg
       ```

3. **Reboot Your System**:
   - Apply the changes by rebooting:
     ```bash
     sudo reboot
     ```

#### Enable IOMMU in systemd-boot

1. **Edit the systemd-boot Configuration**:
   - Open the boot loader configuration file:
     ```bash
     sudo nano /boot/loader/entries/your-entry.conf (your-entry.conf has other name in each distro)
     ```
   - Add the IOMMU settings to the `options` line:
     - For **Intel** CPUs:
       ```bash
       options intel_iommu=on
       ```
     - For **AMD** CPUs:
       ```bash
       options amd_iommu=on
       ```

2. **Reboot Your System**:
   - Apply the changes by rebooting:
     ```bash
     sudo reboot
     ```

#### Verify IOMMU Activation

After rebooting, check if IOMMU is enabled:
```bash
sudo dmesg | grep -e DMAR -e IOMMU
```
You should see a message similar to this:
```bash
DMAR: IOMMU enabled
```


### 3. **Configure the Host System**

#### Install Necessary Packages
- **For Ubuntu/Debian:**
   ```bash
   sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system virt-manager ovmf
- **For Arch Linux:**
   ```bash
  sudo pacman -S qemu-full virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat ebtables iptables libguestfs
- **For Fedora:**
   ```bash
   sudo dnf install qemu-kvm libvirt virt-manager virt-install ovmf
  ```

#### Setting Up the Virtual Machine

- **Enable and start libvirt services:**
```bash
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```
- **Ensure your user is in the libvirt group to have the necessary permissions to manage VMs:**
```bash
sudo usermod -aG libvirt $(whoami)
newgrp libvirt
```
- **List all networks:**
```bash
sudo virsh net-list --all
```
- **Start the default network:**
```bash
sudo virsh net-start default
```
- **Autostart the network (optional but recommended):**
```bash
sudo virsh net-autostart default
```

#### Setting Up the Virtual Machine Using Virt-Manager

1. **Open Virt-Manager**
   - Launch **Virt-Manager** from your application menu or by searching for it.

2. **Create a New Virtual Machine**
   - Click the **“Create a new virtual machine”** button.  
     ![image](https://github.com/user-attachments/assets/f64918cc-613c-4769-a97a-55d77bdaa339)

3. **Choose Installation Media**
   - Select **“Local install media (ISO image or CDROM)”** if you have an ISO file.
   - Click **“Forward”**.
   - Browse to and select your ISO file, then click **“Forward”**.
     - If it doesn't go forward **unselect** the "Automatically detect from the installation media/source" and write it on your own.

4. **Allocate Resources**
   - **Memory**: Allocate RAM (e.g., 8 GB).
   - **CPUs**: Allocate CPU cores (e.g., 4 cores).
   - Click **“Forward”**.

5. **Configure Storage**
   - **Create a new virtual disk**: Set the disk size (e.g., 40 GB) and format (e.g., QCOW2).
   - Click **“Forward”**.

6. **Give Name**
  - **Name your VM**.
  - **Check the box "Customize configuration before install"**.

7. **Set Up Networking**
   - Choose the network configuration:
     - **Default**: Use NAT to share the host’s IP address.
     - **Bridged**: If you need a separate IP address for the VM.
   - Click **“Finish”**.

8. **Customize Configuration**
    - In the configuration window, go to the **Firmware** section and select **UEFI (OVMF)** if your computer supports UEFI.
      - Select the configuration of the architecture you want to use (e.g. x64 for 64-bit or ia32 for 32-bit).
      - Choose the option that does **NOT** contain secure boot.
    - Click **“Apply”**.

9. **Add Virtio**
    - Download Virtio ISO from here **https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md**.
    - Click **Add Hardware**.
    - Click **Storage**.
    - Click **Create a disk image for the virtual machine**.
    - Change the value to **0.1GiB** (It doesn't matter how much size you add because we will delete it after the installation).
    - Click **Bus type**.
    - Select **VirtIO**.
    - Click **Finish**.
    - Click **Add Hardware**.
    - Click **Storage**.
    - Click **Select or create custom storage**.
    - Click **Manage** and then select Virtio ISO file.
    - Select in Device Type **CDROM device** and in BUS Type **SATA**.
    - Click **Finish**.

10. **Change Boot Options**
    - Click **Boot Options**.
    - Check **SATA CDROM 1**.
    - Click **Apply**.

11. **Begin Installation**
    - Click **“Begin Installation”** to start the virtual machine and follow the installation prompts to set up your operating system.

By following these steps, you'll have your virtual machine set up and ready for use with Virt-Manager.

12. **Install Virtio**

    ![Screenshot from 2024-09-07 13-21-33](https://github.com/user-attachments/assets/8d380e73-878b-4b3d-a601-9a4609346aff)

    - Select this Disk and then execute and install this file.

      ![Screenshot from 2024-09-07 13-23-56](https://github.com/user-attachments/assets/bd481f9c-6d0a-44ff-9b72-628e7a1f859b)

    - Shut Down the System.

    - Remove the Drivers we added above
      - Maybe due to a mistake but a second Disk appeared with the same size as the CD. Not removing it causes an error later on.

      ![Screenshot from 2024-09-10 10-43-34](https://github.com/user-attachments/assets/b91f43dd-b4c3-4cd5-8ccc-ad88bc353f71)

13. **Enable XML**
    - Go back to the Virtual Machine Manager.
    - Go to **Edit**.
    - Click **Preferences**.
    - Click **Enable XML editing**.

14. **Change the XML For SATA**
    - Change **bus="sata"** to **bus="virtio"**.
    - Change **type="drive"** to **type="pci"**.

![Screenshot from 2024-09-07 13-31-49](https://github.com/user-attachments/assets/3107a02a-c9c8-472f-abcb-26596c231bd8)

15. **Set VNC (optional)**

    - Old procedure no longer works - a config without Spice cannot be applied
      > Error changing VM configuration: unsupported configuration: chardev 'spicevmc' not supported without spice graphics
      > 
      > Refer to https://bbs.archlinux.org/viewtopic.php?id=277087 for details

      - Old procedure
        - Go to **Display Spice**.
        - Change Type to **VNC server**.
        - Change Address to **All interfaces**

          ![Screenshot from 2024-09-07 13-38-43](https://github.com/user-attachments/assets/24d333cf-5e6a-4eae-a72e-d90476301d91)

      - Fixed procedure
        - Instead of editing **Display Spice**, create a new graphics device and configure it to be the same as above
        - Change **Display Spice** listen type to **None**

16. **Verify VNC**

    Before proceeding it is good to check if the VNC works because it can be used for debug.

    - Disable auto port and choose your own (suggested default of 5900 worked for me)
    - Download a viewer on another machine in the local network
      - I used RealVNC Viewer (first search result)
    - Assuming the VM has default network settings, follow this to connect
      - File -> New connection
      - In the field `VNC Server` put the IP of your host and the port like so `192.168.X.X:PORT`
      - Set a name for the connection and press OK
    - Ignore the unecrypted connection warning

    This worked for me despite still having **Display Spice** enabled.

18. **Setting Up libvirt hooks**

       
    - Verify that modprobe is available.
        ```bash
        which modprobe
        ```
     
        If this does not return anything, modprobe must be either installed,
        ```bash
        sudo apt install kmod
        ```
        or added to path, which worked in my case.
        ```bash
        export PATH=$PATH:/sbin:/usr/sbin
        ```
        source: [bashcommands.com](https://bashcommands.com/bash-modprobe-command-not-found)


    - Create /etc/libvirt/hooks
      ```bash
      sudo mkdir -p /etc/libvirt/hooks
      ```
    - Run the following command to install the hook manager and make it executable
      ```bash
      sudo wget 'https://raw.githubusercontent.com/PassthroughPOST/VFIO-Tools/master/libvirt_hooks/qemu' \
            -O /etc/libvirt/hooks/qemu
      sudo chmod +x /etc/libvirt/hooks/qemu

      ```
    - Restart libvirtd
      ```bash
      sudo service libvirtd restart
                OR
      sudo systemctl restart libvirtd
      ```


      ## Start Script

      - Make the start script:
      ```bash
      sudo mkdir -p /etc/libvirt/hooks/qemu.d/{VM Name}/prepare/begin
      ```

      ```bash
      sudo nano /etc/libvirt/hooks/qemu.d/{VM Name}/prepare/begin/start.sh
      ```

       - Add the following script:

      ```bash
      #!/bin/bash
      # Helpful to read output when debugging
      set -x

      # Stop display manager
      systemctl stop display-manager.service
      # Uncomment the following line if you use GDM
      #killall gdm-x-session
      # sudo rmmod nvidia_drm
      # sudo rmmod nvidia_uvm
      # sudo rmmod nvidia_modeset
      # sudo rmmod nvidia
      sudo rmmod nouveau
      # why not modprobe -r nouveau?

      # Unbind VTconsoles
      echo 0 > /sys/class/vtconsole/vtcon0/bind
      echo 0 > /sys/class/vtconsole/vtcon1/bind

      # Unbind EFI-Framebuffer
      echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

      # Avoid a Race condition by waiting 2 seconds. This can be calibrated to be shorter or longer if required for your system
      sleep 3

      # Unbind the GPU from display driver
      virsh nodedev-detach pci_0000_01_00_0
      virsh nodedev-detach pci_0000_01_00_1

      # Load VFIO Kernel Module
      modprobe vfio-pci
      ```

      - **Save and make it Executable**:
      ```bash
      sudo chmod +x /etc/libvirt/hooks/qemu.d/{VMName}/prepare/begin/start.sh
      ```




      ## End Script

      - Make the end script:
       ```bash
      sudo mkdir -p /etc/libvirt/hooks/qemu.d/{VMName}/release/end
       ```

       ```bash
       sudo nano /etc/libvirt/hooks/qemu.d/{VMName}/release/end/revert.sh
       ```

       - Add the following script:

      ```bash
      #!/bin/bash
      echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind
      set -x

      # Re-Bind GPU to Driver
      virsh nodedev-reattach pci_0000_01_00_0
      virsh nodedev-reattach pci_0000_01_00_1

      sleep 2

      # Reload modules
      modprobe -r vfio-pci
      modprobe nouveau
      # modprobe nvidia
      # modprobe nvidia_modeset
      # modprobe nvidia_uvm
      # modprobe nvidia_drm

      # Rebind VT consoles
      # Some machines might have more than 1 virtual console. Add a line for each corresponding VTConsole
      echo 1 > /sys/class/vtconsole/vtcon0/bind
      echo 1 > /sys/class/vtconsole/vtcon1/bind

      # commenting out
      # because nvidia-xconfig does not exist on my Debian 13 + KDE + Wayland + Nouveau
      # nvidia-xconfig --query-gpu-info > /dev/null 2>&1
      echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

      # Restart Display Manager
      systemctl start display-manager.service
      ```

      - **Save and make it Executable**:
      ```bash
      sudo chmod +x /etc/libvirt/hooks/qemu.d/{VMName}/release/end/revert.sh
      ```


19. **Customize the files**
- Change the marked numbers with yours
![Screenshot from 2024-09-07 14-25-11](https://github.com/user-attachments/assets/fec73398-66f0-4bdf-b426-07d69b311375)

- To find those numbers for your specific system type this command
  ```bash
  lspci | grep -i nvidia  # your GPU vendor
  ```
  In my case those are the numbers (You probably have different numbers or even more that two PCIs)


![Screenshot from 2024-09-07 14-30-04](https://github.com/user-attachments/assets/2bb46be2-23eb-4229-a65d-873e5e37aa9b)


- Same goes for the modules. If you have **AMD** or **Intel** find those modules for your system and replace them.

  ![Screenshot from 2024-09-07 14-34-03](https://github.com/user-attachments/assets/0852a2ad-6d45-4fc3-a631-613780cd8fc9)
  
  To determine which modules must be replaced, use `lsmod` and grep the output. Open source NVidia driver will not be found if searching for `nvidia` because it is called `nouveau`. My Intel HD Graphics driver is called `i915`.

  Which modules must be stopped when using `nouveau` is yet to be determined as of now.

18. **Add the Hardware to the VM**
- Click on Add Hardware and select **PCI Host Device**.
- Select everything related to your **Graphics Card**.
- Click Finish.
- Click on Add Hardware and select **USB Host Device**.
- Select your **Mouse** and **Keyboard**.
- Click Finish.

![Screenshot from 2024-09-07 14-41-43](https://github.com/user-attachments/assets/460c694d-7604-4fdc-9da7-d8f72b4f2b79)

19. **Single GPU Passthrough**

- When you start your VM you will probably not have any output because your graphics card drivers are not installed yet.
- You can just wait until the drivers are automatically downloaded. If after a while you still have no output, connect from an other device to your VM with VNC and install them manually.
- After the drivers are installed and you have output, you can remove **Display VNC / Display Spice** and **Video QXL / Video Bochs** and use the GPU directly.  

 ![remove](https://github.com/user-attachments/assets/281d0499-4d43-4339-8219-bc7a0f53e410)




------------------------------
## Test log


### First attempt

**Conditions:**

- Only the `nouveau` module in the libvirt hooks.

**Report:**

- Fail.
- Another monitor (connected to MB) also went black.
    - Could this be somehow related to `systemctl stop display-manager.service` in `start.sh`?
- Windows did not take over the GPU.
- VNC was not available at the time to debug


### Attempt 2

**Conditions:**

- exactly the same as above but with VNC connection to another machine (see the `Verify VNC` chapter) 

**Report:**

- Success.
- Several seconds after the screen went black, I connected to Windows via VNC
- Notifications appeared about my mouse and keyboard being set up
- Opened device manager and looked for issues, found an unrecognized display adaptor
- After a while Windows automatically recognized it as my NVidia GPU and installed the drivers
- The screen activated and revealed Windows in full resolution
- The only issue is that my mouse did not work (pointing had to be conducted via VNC)
- After shutting down Windows, Debian + Plasma returned to the screen
- All windows / applications have been closed (somewhat unexpected) and the mouse still did not work


### Attempt 3

**Report:**

- ~~Mouse problem resolved itself~~
    - However during audio tweaks it still sometimes broke after returning to Debian
- I tested audio in the VM and it is not working (I am not using the GPU audio interface that's been passed through)
  - Audio is tested by opening Firefox and playing a YouTube video

**Next steps:**

- Use `lspci | grep -i audio` to find the correct audio device
    - In my case that's `00:1f.3 Audio device: Intel Corporation 200 Series PCH HD Audio`
- Use `lspci -v` to find which kernel modules use this device

      00:1f.3 Audio device: Intel Corporation 200 Series PCH HD Audio
              Subsystem: Micro-Star International Co., Ltd. [MSI] Device da62
              Flags: bus master, fast devsel, latency 32, IRQ 148, IOMMU group 13
              Memory at 2fff020000 (64-bit, non-prefetchable) [size=16K]
              Memory at 2fff000000 (64-bit, non-prefetchable) [size=64K]
              Capabilities: <access denied>
              Kernel driver in use: snd_hda_intel
              Kernel modules: snd_hda_intel, snd_soc_avs

- Edit libvirt hooks to also stop and restart those modules (similar to `nouveau`)
    - `start.sh`: `sudo rmmod snd_hda_intel`, `sudo rmmod snd_soc_avs`
    - `revert.sh`: `modprobe snd_hda_intel`, `modprobe snd_soc_avs`
 
- Edit the VM settings again
    - Remove any `Sound` devices
        - I tried using them and simply editing their PCIe info but that did not work 
    - Click `Add Hardware`, select `PCI Host Device` and select the Audio device


### Attempt with audio via PCIE

**Report:**

- Something crashed, Windows never started and instead what I saw was Debian login screen
- Probably something to do with IOMMU device grouping (?)
- At least audio and the mouse both work
 
**Next steps:**
 
- Remove that PCIe device and go back to the `Sound` one, maybe it will work with that module unloading


### Attempt #2 with ich9 audio

**Report:**
 
- Windows booted up but with neither audio nor mouse (interestingly enough even the pointer was missing)
- No issues found in Device Manager
- Back in Debian audio worked but pointing didn't

**Next steps:**

- Mouse issue:
    - Lenghten the sleep time in the libvirt **start** hook from 3 to 5 seconds.
        - This time KDE windows were still visible for several seconds so maybe something was not closed properly
     

### Mouse fix attempt #1

**Report (mouse issue):**
- Mouse worked in Windows but not back in Debian

**Next steps**
- Set the time to 5s in the exit script


### Mouse fix attempt #2

**Report (mouse issue):**
- After 6 reboots mouse stopped working in Debian only once
- The results are inconsistent and the 5s delay in both scripts makes testing very cumbersome
- Reverting back to 3s in both scripts


### Back to the audio issue

**Research**
- Figure out why `ich9` is the newest available option in the VM manager and whether I should even be attempting to use it
    - (It's from 2007) [en.wikipedia.org](https://en.wikipedia.org/wiki/I/O_Controller_Hub#ICH9)
- My MB specs lists `Realtek ALC1220 Codec` as rear panel audio
- Used `aplay -lL`, this excerpt confirms that `Realtek ALC1220` is recognized. There were also entries called `ALC1220 Digital` but I am not currently using those.

      ...
      hw:CARD=PCH,DEV=0
          HDA Intel PCH, ALC1220 Analog
          Direct hardware device without any conversions
      ...
      plughw:CARD=PCH,DEV=0
          HDA Intel PCH, ALC1220 Analog
          Hardware device with all software conversions
      ...
      **** List of PLAYBACK Hardware Devices ****
      card 0: PCH [HDA Intel PCH], device 0: ALC1220 Analog [ALC1220 Analog]
        Subdevices: 1/1
        Subdevice #0: subdevice #0
      ...

- Used `lsmod | grep ^snd;`, the output points to more modules to unload in libvirt hooks

      snd_seq_dummy          12288  0
      snd_hrtimer            12288  1
      snd_seq               110592  7 snd_seq_dummy
      snd_seq_device         16384  1 snd_seq
      snd_soc_avs           212992  0
      snd_soc_hda_codec      24576  1 snd_soc_avs
      snd_hda_ext_core       36864  2 snd_soc_avs,snd_soc_hda_codec
      snd_hda_codec_realtek   217088  1
      snd_soc_core          421888  2 snd_soc_avs,snd_soc_hda_codec
      snd_hda_codec_generic   114688  1 snd_hda_codec_realtek
      snd_hda_scodec_component    20480  1 snd_hda_codec_realtek
      snd_hda_codec_hdmi     98304  2
      snd_compress           28672  2 snd_soc_avs,snd_soc_core
      snd_pcm_dmaengine      16384  1 snd_soc_core
      snd_hda_intel          61440  2
      snd_intel_dspcfg       40960  2 snd_soc_avs,snd_hda_intel
      snd_intel_sdw_acpi     16384  1 snd_intel_dspcfg
      snd_hda_codec         217088  6 snd_hda_codec_generic,snd_soc_avs,snd_hda_codec_hdmi,snd_soc_hda_codec,snd_hda_intel,snd_hda_codec_realtek
      snd_hda_core          143360  8 snd_hda_codec_generic,snd_soc_avs,snd_hda_codec_hdmi,snd_soc_hda_codec,snd_hda_intel,snd_hda_ext_core,snd_hda_codec,snd_hda_codec_realtek
      snd_hwdep              20480  1 snd_hda_codec
      snd_pcm               184320  8 snd_soc_avs,snd_hda_codec_hdmi,snd_hda_intel,snd_hda_codec,snd_compress,snd_soc_core,snd_hda_core,snd_pcm_dmaengine
      snd_timer              53248  3 snd_seq,snd_hrtimer,snd_pcm
      snd                   151552  18 snd_hda_codec_generic,snd_seq,snd_seq_device,snd_hda_codec_hdmi,snd_hwdep,snd_hda_intel,snd_hda_codec,snd_hda_codec_realtek,snd_timer,snd_compress,snd_soc_core,snd_pcm

- Command `sudo fuser -v /dev/snd/*` outputs processes responsible for audio
      - This is well explained in this [archlinux.org thread]([url](https://bbs.archlinux.org/viewtopic.php?pid=1639728#p1639728)) even though my setup is different.
      - Most importantly this reveals that `pipewire` is used.

                              USER        PID ACCESS COMMAND
        /dev/snd/controlC0:  disken     1184 F.... wireplumber
        /dev/snd/controlC1:  disken     1184 F.... wireplumber
        /dev/snd/seq:        disken     1181 F.... pipewire

    - If removing modules does not work I should make sure that those do not interfere

**Next steps (audio issue):**
- Changes in the setup script
    - Kill all `pipewire` and `wireplumber` processes
        ```bash
        pkill -f pipewire
        pkill -f wireplumber
        ```
    - Remove modules `snd_hda_codec_realtek`, `snd_hda_codec` and `snd_intel_dspcfg`
        ```bash
        sudo rmmod snd_hda_codec_realtek
        sudo rmmod snd_hda_codec
        sudo rmmod snd_intel_dspcfg
        ```
- Changes in the revert script
    - Restart removed modules
        ```bash
        modprobe snd_hda_codec_realtek
        modprobe snd_hda_codec
        modprobe snd_intel_dspcfg
        ```
    - Restart pipewire
        ```bash
        pipewire
        ```


### Attempt #3 with ich9 audio

**Report:**
- No audio in Windows, works back in Debian
- No changes in Device Mananger

**Next steps:**
- Connect my spare USB audio card and check if it can be passed through more easily
    - Adjust physical hardware
    - Open VM settings
      - Remove `sound` device
      - Add `USB Host Device` and select the sound card


### Attempt #1 with USB sound card

**Report:**
- Audio works in the system but in game it stops for about 1s every few minutes
    - Further tests required
- In-game FPS meter reports stable numbers but the actual performance feels worse

**Next steps:**
- Adjust CPU topology
    - Change `3:1:1` (sockets:cores:threads) to `1:3:1` without proactively checking if those numbers make sense


### Attempts with new CPU topology

**Report**
- After several failed attempts with different topologies, I think `1:3:1` provides somewhat improved performance but more testing is needed because this coincided with some additional tweaks, primarily in game settings.
- Mouse failure after returning to Debian is more consistent but I can restore basic functionality by re-plugging the cable (cannot use the builtin DPI switch for some reason and the indicator lights are off).
- Audio issues intensified during the early part of the game but this improved after several minutes.
