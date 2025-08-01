# Steps-to-Deploy-an-AlmaLinux-10-PXE-Server
AlmaLinux 10, as a RHEL-compatible distribution, follows a similar PXE network installation process to RHEL 9. This requires configuring DHCP, TFTP services, and preparing an installation source. Below are the detailed deployment steps:  


#### **1. Environment Preparation**  
1. **Server Requirements**  
   - A server running AlmaLinux 10 (to act as the PXE server) with a static IP address (example: `192.168.1.100/24`).  
   - The server must have `dhcp-server`, `tftp-server`, and an HTTP service (e.g., `httpd`) installed to provide the installation source.  
   - Clients must support network booting (enable PXE boot in BIOS/UEFI).  

2. **Install Required Software**  
   ```bash
   dnf install dhcp-server tftp-server httpd -y
   ```  


#### **2. Configure the DHCP Server (DHCPv4)**  
The DHCP server assigns IP addresses to clients and specifies the TFTP server address and boot file path.  

1. **Edit the DHCP Configuration File**  
   ```bash
   vim /etc/dhcp/dhcpd.conf
   ```  

2. **Add the Following Configuration** (modify IP ranges, gateways, etc., for your network):  
   ```conf
   option architecture-type code 93 = unsigned integer 16;

   subnet 192.168.1.0 netmask 255.255.255.0 {
     option routers 192.168.1.1;  # Gateway address
     option domain-name-servers 114.114.114.114;  # DNS server
     range 192.168.1.101 192.168.1.200;  # Client IP address range
     
     # For PXE clients (BIOS/UEFI)
     class "pxeclients" {
       match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
       next-server 192.168.1.100;  # PXE server (TFTP) IP
       
       # Differentiate between UEFI and BIOS architectures
       if option architecture-type = 00:07 {  # UEFI 64-bit
         filename "alma/efi/BOOT/BOOTX64.EFI";
       } else {  # BIOS
         filename "pxelinux/pxelinux.0";
       }
     }
   }
   ```  

3. **Start and Enable the DHCP Service**  
   ```bash
   systemctl enable --now dhcpd
   firewall-cmd --add-service=dhcp --permanent
   firewall-cmd --reload
   ```  


#### **3. Configure the TFTP Server**  
The TFTP server provides boot files (e.g., `pxelinux.0`, UEFI bootloaders).  

1. **TFTP Root Directory**  
   AlmaLinux’s default TFTP root directory is `/var/lib/tftpboot`, where boot files will be stored.  

2. **Open Firewall Ports**  
   ```bash
   firewall-cmd --add-service=tftp --permanent
   firewall-cmd --reload
   ```  

3. **Mount the AlmaLinux 10 ISO and Copy Boot Files**  
   - Download the AlmaLinux 10 ISO (e.g., `AlmaLinux-10.0-x86_64-dvd.iso`) and mount it on the server:  
     ```bash
     mkdir /mnt/almaiso
     mount -t iso9660 -o loop /path/to/AlmaLinux-10.0-x86_64-dvd.iso /mnt/almaiso
     ```  


#### **4. Prepare Boot Files for BIOS Clients**  
1. **Copy the BIOS Bootloader `pxelinux.0`**  
   `pxelinux.0` is included in the `syslinux-tftpboot` package. Install and extract it:  
   ```bash
   dnf install syslinux-tftpboot -y
   cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/pxelinux/
   ```  

2. **Create PXE Configuration Directories**  
   ```bash
   mkdir -p /var/lib/tftpboot/pxelinux/{pxelinux.cfg,images/alma10}
   ```  

3. **Copy Kernel and Initialization Images**  
   Copy `vmlinuz` and `initrd.img` from the ISO to the TFTP directory:  
   ```bash
   cp /mnt/almaiso/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/images/alma10/
   ```  

4. **Create the PXE Menu Configuration File**  
   ```bash
   vim /var/lib/tftpboot/pxelinux/pxelinux.cfg/default
   ```  
   Add the following content:  
   ```conf
   default vesamenu.c32
   prompt 1
   timeout 600

   label alma10-install
     menu label ^Install AlmaLinux 10
     menu default
     kernel images/alma10/vmlinuz
     append initrd=images/alma10/initrd.img ip=dhcp inst.repo=http://192.168.1.100/alma10  # HTTP installation source path
   label local
     menu label Boot from ^local drive
     localboot 0xffff
   ```  


#### **5. Prepare Boot Files for UEFI Clients**  
1. **Copy UEFI Bootloaders**  
   Copy EFI boot files from the ISO to the TFTP directory:  
   ```bash
   mkdir -p /var/lib/tftpboot/alma/efi/BOOT
   cp -r /mnt/almaiso/EFI/BOOT/* /var/lib/tftpboot/alma/efi/BOOT/
   ```  

2. **Modify UEFI Boot Configuration**  
   Edit `grub.cfg` to specify the installation source:  
   ```bash
   vim /var/lib/tftpboot/alma/efi/BOOT/grub.cfg
   ```  
   Add the following content:  
   ```conf
   set timeout=60
   menuentry 'Install AlmaLinux 10' {
     linuxefi /images/alma10/vmlinuz ip=dhcp inst.repo=http://192.168.1.100/alma10
     initrdefi /images/alma10/initrd.img
   }
   ```  

3. **Copy Kernel and Initialization Images (Shared with BIOS)**  
   Skip if already completed for BIOS; otherwise:  
   ```bash
   mkdir -p /var/lib/tftpboot/images/alma10
   cp /mnt/almaiso/images/pxeboot/{vmlinuz,initrd.img} /var/lib/tftpboot/images/alma10/
   ```  


#### **6. Configure the HTTP Installation Source**  
The HTTP server hosts the AlmaLinux installation tree (ISO contents).  

1. **Copy ISO Contents to the HTTP Root Directory**  
   ```bash
   mkdir /var/www/html/alma10
   cp -r /mnt/almaiso/* /var/www/html/alma10/
   chmod -R 755 /var/www/html/alma10
   ```  

2. **Start the HTTP Service**  
   ```bash
   systemctl enable --now httpd
   firewall-cmd --add-service=http --permanent
   firewall-cmd --reload
   ```  


#### **7. Start the TFTP Service and Test**  
1. **Start the TFTP Service**  
   ```bash
   systemctl enable --now tftp.socket
   ```  

2. **Client Testing**  
   - Boot the client, enter BIOS/UEFI, and select network boot.  
   - The client will obtain an IP via DHCP, load boot files from TFTP, access the installation source via HTTP, and enter the AlmaLinux 10 installation interface.  


### Notes  
- Ensure the `inst.repo` path matches the HTTP installation source (example: `http://192.168.1.100/alma10`).   
- All paths and IP addresses must be modified to match your actual network environment.
