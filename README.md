# Khung Chương Trình Học Proxmox VE

**Từ Cơ Bản Đến Nâng Cao**

**Phiên bản:** 2.0  
**Cập nhật:** Tháng 1, 2026  
**Mục tiêu:** Cung cấp lộ trình học tập toàn diện từ không có kiến thức đến thành thạo Proxmox VE

## Phần I: Giai Đoạn 1 - Nền Tảng Hệ Điều Hành (2-3 tuần)

### 1.1 Mục Tiêu Giai Đoạn
- Nắm vững kiến thức cơ bản Linux/Debian
- Hiểu rõ về ảo hóa và hypervisor
- Chuẩn bị môi trường lab

### 1.2 Nội Dung Chi Tiết
**Linux Fundamentals**  
- Kernel Linux cơ bản: Process, Memory Management, File System  
- Lệnh dòng lệnh: ls, cd, mkdir, chmod, chown, grep, sed, awk  
- Package Management: apt, apt-get, dpkg (Debian/Ubuntu)  
- User & Permission: sudo, groups, file permissions  
- Network Basics: ifconfig, ip, netstat, ping, ssh

**Virtualization Concepts**  
- Ảo hóa là gì? Hypervisor Type 1 vs Type 2  
- KVM (Kernel-based Virtual Machine): Hiểu cơ chế  
- QEMU: Quick Emulator basics  
- Container vs VM: Sự khác biệt và use cases

**Chuẩn Bị Phần Cứng**  
- Yêu cầu tối thiểu: CPU hỗ trợ virtualization, RAM 16GB+, Disk 100GB+
- Kiểm tra: CPU flags (vmx hoặc svm)              
- Lab setup options: Physical server hoặc nested virtualization

### 1.3 Tài Liệu & Học Liệu
| Tài Liệu                      | Loại     | Link/Ghi Chú                                      |
|-------------------------------|----------|---------------------------------------------------|
| Linux Academy - Linux Basics  | Video    | https://www.udemy.com/course/linux-basics/ |
| Linux Handbook                | Sách/Blog| https://linuxhandbook.com/                |
| KVM Hypervisor Basics         | Tài liệu | https://www.redhat.com/en/topics/virtualization/what-is-KVM |
| Proxmox Architecture Overview | Chính thức | https://pve.proxmox.com/wiki/Architecture |
| YouTube: Linux Terminal Tutorial | Video | https://www.youtube.com/results?search_query=linux+terminal+basics |

### 1.4 Bài Tập Thực Hành
- [ ] Cài đặt Ubuntu Server trên VM  
- [ ] Thành thạo 20 lệnh Linux cơ bản  
- [ ] Cấu hình SSH key-based authentication  
- [ ] Kiểm tra CPU flags hỗ trợ virtualization  
- [ ] Tạo user mới với sudo privileges

## Phần II: Giai Đoạn 2 - Cơ Bản Proxmox (3-4 tuần)

### 2.1 Mục Tiêu Giai Đoạn
- Cài đặt và cấu hình Proxmox VE thành công  
- Tạo và quản lý VM & LXC containers đơn giản  
- Hiểu web GUI và CLI cơ bản

### 2.2 Nội Dung Chi Tiết
**Installation & Initial Setup**  
- System Requirements: CPU, RAM, Storage, Network  
- Proxmox ISO Download & Burn: Tạo USB cài đặt  
- Installation Process: Các step chi tiết  
- Post-installation: Updates, subscription (disable nếu community)

**Creating VMs**  
- VM Lifecycle: Create, Start, Stop, Delete  
- Resource Allocation: vCPU, Memory, Disk sizing  
- OS Installation: Windows, Ubuntu, CentOS  
- QEMU Guest Agent: Cài đặt và cấu hình  
- VM Snapshots: Tạo, restore, delete

**Creating LXC Containers**  
- Container vs VM: Khác biệt chi tiết  
- Container Creation: Sử dụng templates  
- Container Management: Start, stop, exec

**Basic Storage**  
- Storage Types: Local, NFS, iSCSI, LVM  
- Local Storage: Directory, LVM, ZFS basics

### 2.3 Tài Liệu & Học Liệu
| Tài Liệu                        | Loại       | Link/Ghi Chú                                           |
|---------------------------------|------------|--------------------------------------------------------|
| Proxmox Official Installation   | Chính thức | https://pve.proxmox.com/wiki/Installation     |
| Proxmox Admin Guide (Part 1-3)  | Chính thức | https://pve.proxmox.com/pve-docs/pve-admin-guide.html |
| WunderTech Beginner's Guide     | Video      | https://www.youtube.com/watch?v=lFzWDJcRsqo  |

### 2.4 Bài Tập Thực Hành
- [ ] Cài đặt Proxmox VE trên server/VM  
- [ ] Tạo VM Ubuntu 22.04 LTS  
- [ ] Tạo LXC container Debian  
- [ ] Thực hiện snapshot và restore VM

## Phần III: Giai Đoạn 3 - Trung Cấp (4-5 tuần)

### 3.1 Mục Tiêu Giai Đoạn
- Cấu hình networking nâng cao
- Quản lý storage hiệu quả
- Triển khai backup strategy
- Tối ưu hóa performance

### 3.2 Nội Dung Chi Tiết

**Advanced Networking**
- Network Bridges: Tạo và cấu hình multiple bridges
- VLANs: Virtual LAN configuration, tagging
- Network Bonding: Active-active, Active-passive, Balance modes
- Firewall: Proxmox built-in firewall, rules, zones
- SDN: Software Defined Networking VNets, Zones cơ bản
- DNS & DHCP: Cấu hình, troubleshooting

**Storage Management (Intermediate)**
- Storage Architecture: Giải thích từng loại
- LVM: Logical Volume Manager PV, VG, LV - tạo, mở rộng, thu nhỏ
- ZFS Fundamentals: RAID types, Datasets, Snapshots
- NFS Setup: Tạo NFS server, mount từ Proxmox
- iSCSI: Basics, configuration cho storage

**Backup & Disaster Recovery**
- Backup Types: Full, Incremental, Differential
- Proxmox Backup Server (PBS): Giới thiệu, setup cơ bản
- Backup Strategies: Scheduling, retention policies

**Performance Optimization**
- CPU: CPU pinning, NUMA awareness
- Memory: Ballooning, Swap, Over-commit
- Disk I/O: Cache modes (writethrough, writeback), Scheduler

### 3.3 Tài Liệu & Học Liệu
| Tài Liệu | Loại | Link/Ghi Chú |
|----------|------|--------------|
| Proxmox Admin Guide (Networking) | Chính thức | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_networking |
| Proxmox Admin Guide (Storage) | Chính thức | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_storage |
| LVM Tutorial | Blog | https://www.howtogeek.com/howto/40702/how-to-manage-and-use-lvm-logical-volume-manager/ |
| ZFS on Linux | Tài liệu | https://openzfs.org/wiki/Documentation |
| Proxmox Backup Server | Chính thức | https://proxmox.com/en/proxmox-backup-server/overview |

### 3.4 Bài Tập Thực Hành
- [ ] Tạo VLAN và cấu hình VM trên các VLAN khác nhau
- [ ] Cấu hình Network Bonding cho high availability
- [ ] Tạo LVM storage pool từ additional disk
- [ ] Setup NFS server và mount từ Proxmox
- [ ] Cấu hình Firewall rules cho VMs
- [ ] Implement backup schedule cho VM
- [ ] Tối ưu hóa VM performance (pinning, cache modes)

---

## Phần IV: Giai Đoạn 4 - Cluster & Ceph (5-6 tuần)

### 4.1 Mục Tiêu Giai Đoạn
- Xây dựng Proxmox Cluster
- Triển khai Ceph storage
- Quản lý high availability
- Thực hiện live migration

### 4.2 Nội Dung Chi Tiết

**Proxmox Clustering**
- Cluster Architecture: Corosync, pmxcfs, Pacemaker concepts
- Creating Cluster: Setup node pertama, join node lần 2+
- Cluster Networking: Separate corosync traffic, latency requirements
- High Availability (HA): Failover policies, resource groups

**Ceph Storage Basics**
- Ceph Architecture: MONs, OSDs, Object storage
- Ceph Installation: Từ Proxmox GUI, requirement kiểm tra
- OSD Management: Add, remove, reweight OSDs
- Ceph Pools: Tạo pools cho VM disks, containers
- Ceph RBD: VM disk storage

### 4.3 Tài Liệu & Học Liệu
| Tài Liệu | Loại | Link/Ghi Chú |
|----------|------|--------------|
| Proxmox Admin Guide (Cluster) | Chính thức | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_cluster |
| Proxmox Admin Guide (Ceph) | Chính thức | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_ceph |
| Ceph Official Documentation | Chính thức | https://docs.ceph.com/ |
| Proxmox Training Module 2 | Video | https://www.proxmox.com/en/services/training-courses/training |

### 4.4 Bài Tập Thực Hành
- [ ] Tạo Proxmox cluster với 3 nodes (có thể nested VMs)
- [ ] Verify corosync clustering communication
- [ ] Cấu hình HA policies cho VM
- [ ] Triển khai Ceph cluster (MON, OSD nodes)
- [ ] Tạo Ceph pool cho VM storage
- [ ] Thực hiện VM live migration giữa nodes

---

## Phần V: Giai Đoạn 5 - Nâng Cao (4-6 tuần)

### 5.1 Mục Tiêu Giai Đoạn
- Automation & Infrastructure as Code
- Advanced security
- Performance tuning & optimization
- Multi-site & Disaster Recovery

### 5.2 Nội Dung Chi Tiết

**Automation & Infrastructure as Code**
- Proxmox API: REST API deep dive, curl/Python examples
- Terraform: Proxmox Provider, Infrastructure as Code
- Ansible: Playbooks cho Proxmox management

**Advanced Security**
- SSL/TLS Certificates: Self-signed vs Let's Encrypt
- 2FA/MFA: Enabling multi-factor authentication
- LDAP/AD Integration: Enterprise authentication

**Performance Tuning**
- CPU Optimization: Pinning, hyperthreading, NUMA tuning
- Ceph Optimization: OSD tuning, PG optimization

### 5.3 Tài Liệu & Học Liệu
| Tài Liệu | Loại | Link/Ghi Chú |
|----------|------|--------------|
| Proxmox API Documentation | Chính thức | https://pve.proxmox.com/pve-docs/api-viewer/ |
| Terraform Proxmox Provider | GitHub | https://github.com/Telmate/terraform-provider-proxmox |
| Ansible Proxmox Collection | Ansible | https://github.com/ansible-collections/community.general |

### 5.4 Bài Tập Thực Hành
- [ ] Viết Terraform code để tạo 3 VMs
- [ ] Tạo Ansible playbook cấu hình VMs automatically
- [ ] Setup SSL/TLS với Let's Encrypt
- [ ] Setup Kubernetes cluster trên Proxmox

---

## Phần VI: Tài Liệu Tham Khảo Tổng Hợp

### Tài Liệu Chính Thức Proxmox
- **Proxmox VE Administration Guide**: https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- **Proxmox API Viewer**: https://pve.proxmox.com/pve-docs/api-viewer/
- **Proxmox Forum**: https://forum.proxmox.com/

### YouTube Channels & Tutorials
- **WunderTech**: https://www.youtube.com/c/WunderTechTutorials
- **Proxmox Official**: Official training videos
- **Techno Tim**: Homelab tutorials

### Community Resources
- **Community Helper Scripts**: https://community-scripts.github.io/ProxmoxVE/
- **Proxmox Reddit**: https://www.reddit.com/r/Proxmox/

---

## Phần VII: Lộ Trình Chi Tiết Theo Thời Gian

**Khuyến Nghị Tiến Độ**:
