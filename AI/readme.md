# Hướng dẫn thiết lập hệ thống Proxmox, Kubernetes, Ray, JupyterHub, Apache với Supermicro 4029GP-TRT và Dell R640 nâng cấp, Ceph Backup, VLAN 50 Management, mạng 10Gbps, NVMe/SSD Cache

## Mục tiêu
- Tạo cụm Proxmox trên **4 server Dell R640** và tích hợp **Supermicro 4029GP-TRT**.
- Tối ưu 2 workloads:
  - **Workload 1 (Realtime Alpha)**: Chạy 50,000 alpha realtime, gọi <1,000 datapoint (<2MB), kết quả trong 1-2 giây (tối ưu CPU Dell R640, Ray Actors/Tasks, Ceph fast tier NVMe low-latency, Redis cache).
  - **Workload 2 (Research Training)**: Cho 50 researcher training alpha với mô hình đơn giản (Optuna, GBoosting, Ridge, DT, RF) và DL (CNN, BNN, LSTM) với ~100,000 datapoint (>20MB, truy xuất không thường xuyên, 1s/read) (tối ưu GPU V100 Supermicro, Ray Train, Ceph slow tier SSD for large data).
- Sử dụng **Kubernetes** bare-metal để quản lý container, đảm bảo HA và mở rộng.
- Triển khai **Ray on Kubernetes** cho tính toán phân tán và ML/AI (DL, BNN, CNN, ma trận tối ưu).
- Triển khai **JupyterHub** trên Kubernetes với môi trường Python, tích hợp Ray và GPU cho ML/AI (namespace riêng cho research).
- Sử dụng **Apache** làm reverse proxy.
- Đảm bảo job chạy liên tục nếu node lỗi.
- Chuyển **Management** sang **VLAN 50**, giữ **User Access** trên **VLAN 10**.
- Bổ sung **Cisco Nexus C3172PQ-10GE 48 port 10GB** for switch.
- Sử dụng NVMe làm cache (L2ARC/ZIL) và Ceph for shared backup/snapshot (replication 3 for fast tier, EC 4+2 for slow tier).
- Bảo mật Ceph và hệ thống.

## Chuẩn bị
- **Phần cứng**:
  - **Supermicro 4029GP-TRT**: 2x CPU Xeon Platinum 8173M 28 Core, 5x GPU Tesla V100, 1TB RAM 2666Mhz, card RAID LSI 9364-8i 1GB, 4x SSD NVMe PCIe 4.0 Samsung PM1735 1.62 TB, 4x SSD Micron 5300 Pro 1.92TB SATA III, 2x SSD Micron 5300 Pro 960GB SATA III (for OS).
  - **Dell R640**: 4 server, mỗi server 2x CPU Xeon Platinum 8173M 28 Core, 256GB RAM 2666Mhz, 6x SSD U2 Samsung PM1735 1.6 TB, 2x SSD Micron 5300 Pro 960GB SATA III (for OS), 1x SSD Micron 5300 Pro 1.92TB SATA III (cho Ceph slow tier, thêm vào mỗi server), RAID Dell H730.
  - **Cisco Nexus C3172PQ-10GE**: 48 port 10GB for switch quang.
  - **Card quang**: 10Gbps SFP+ NIC (ví dụ: Mellanox ConnectX-4) cho Dell/Supermicro nếu cần.
  - **Cáp quang**: Multimode LC-LC.
- **Mạng**: Router/switch hỗ trợ VLAN, kết nối 10Gbps (node) và 1Gbps (user).
- **Phần mềm**: Ubuntu 22.04 template, Kubernetes, Docker, NVIDIA driver, nvme-cli, Ray, Ceph, Redis.
- **Công cụ**: Màn hình, bàn phím, máy tính cá nhân.

**Tối ưu RAID/Ceph**:
- **RAID**: RAID 0 for NVMe (tốc độ cao for realtime workload 1), RAID 1 for OS SSD (an toàn).
- **Ceph**: Shared storage replication 3 for fast/hot tier (NVMe, workload 1 small/frequent), EC 4+2 for slow/cold tier (SSD Micron 5300 Pro trên Dell R640 và Supermicro, overhead 1.5x, chịu 2 lỗi). Ceph crush map tự tiering, replication/EC kết hợp giảm storage overhead 30-50% so replication 3 toàn bộ.

## Kế hoạch chi tiết

### Bước 1: Thiết kế và chia tách dải IP
- **VLAN 10 (User Access)**: `192.168.10.0/24` (PC researcher).
- **VLAN 20 (Kubernetes/JupyterHub)**: `192.168.20.0/24`.
- **VLAN 50 (Management)**: `192.168.50.0/24`
  - Node 1 (Dell R640): `192.168.50.10`
  - Node 2 (Dell R640): `192.168.50.11`
  - Node 3 (Dell R640): `192.168.50.12`
  - Node 4 (Dell R640, dự phòng): `192.168.50.13`
  - Supermicro 4029GP-TRT: `192.168.50.14`

#### 1.1. Cấu hình VLAN và switch quang
- Đăng nhập router/switch (`192.168.1.1`).
- Tạo VLAN:
  ```plaintext
  VLAN 10: User Access (192.168.10.0/24, Gateway: 192.168.10.1)
  VLAN 20: Kubernetes/JupyterHub (192.168.20.0/24, Gateway: 192.168.20.1)
  VLAN 50: Management (192.168.50.0/24, Gateway: 192.168.50.1)
  ```
- Cấu hình Cisco Nexus C3172PQ-10GE:
  - Kết nối node (cổng SFP+) vào switch quang.
  - Gán cổng SFP+ cho VLAN 50.
  - Gán cổng RJ45 cho VLAN 10 và VLAN 20.
- Firewall:
  ```plaintext
  Allow VLAN10 -> VLAN20 (port 80/443)
  Allow VLAN50 -> VLAN20
  Deny VLAN10 -> VLAN50
  ```

#### 1.2. Cài đặt card quang 10Gbps
- Trên Dell R640/Supermicro nếu cần:
  1. Lắp card quang 10Gbps (Mellanox ConnectX-4) vào khe PCIe.
  2. Kết nối cáp quang LC-LC từ card đến switch quang.
  3. Kiểm tra:
     ```bash
     lspci | grep Mellanox
     ip link show
     ```

### Bước 2: Cài đặt và cấu hình NVMe/SSD với Ceph
- Trên Dell R640 (4 server):
  1. Lắp 6x SSD U2 PM1735 1.6TB, 2x SSD Micron 960GB SATA III (for OS), 1x SSD Micron 5300 Pro 1.92TB SATA III (cho Ceph slow tier).
  2. Cấu hình RAID Dell H730 (BIOS F2 > Device Settings > RAID):
     - RAID 0 for 6x SSD U2 PM1735 (~9.6TB, tốc độ cao).
     - RAID 1 for 2x SSD 960GB OS (960GB).
     - Single OSD for 1x SSD Micron 5300 Pro 1.92TB (không RAID cho Ceph Bluestore).
  3. Kiểm tra:
     ```bash
     lsblk
     ```

- Trên Supermicro 4029GP-TRT:
  1. Lắp 4x NVMe PM1735 1.62TB, 4x SSD Micron 5300 Pro 1.92TB SATA III, 2x SSD Micron 5300 Pro 960GB SATA III (for OS), card RAID LSI 9364-8i for SATA.
  2. Cấu hình RAID LSI 9364-8i (BIOS Ctrl+H):
     - RAID 0 for 4x NVMe HHHL (~6.48TB).
     - RAID 5 for 4x SSD Micron 5300 Pro (~5.76TB).
     - RAID 1 for 2x SSD 960GB OS (960GB).
  3. Kiểm tra:
     ```bash
     lsblk
     ```

- Cài Ceph on Proxmox (web UI: Datacenter > Ceph):
  1. Cài Ceph packages trên all node:
     ```bash
     pveam update
     apt install ceph ceph-mgr
     ```
  2. Tạo Ceph cluster (UI: Ceph > Create Cluster).
  3. Thêm Monitor on 3 node (Dell 1,2,3), Manager on all.
  4. Thêm OSD on NVMe RAID 0 (fast tier, low-latency for workload 1).
  5. Thêm OSD on SSD Micron 5300 Pro (slow tier, large data for workload 2) từ cả Supermicro và Dell R640.
- Tối ưu Ceph:
  1. Cấu hình EC profile for slow tier:
     ```bash
     ceph osd erasure-code-profile set ec_profile k=4 m=2 crush-failure-domain=osd
     ```
  2. Pool for workload 1 (replication 3, fast):
     ```bash
     ceph osd pool create realtime_pool 128 128 replicated
     ceph osd pool set realtime_pool size 3
     ceph osd crush rule create-replicated fast_tier default host nvme
     ceph osd pool set realtime_pool crush_rule fast_tier
     ```
  3. Pool for workload 2 (EC 4+2, slow):
     ```bash
     ceph osd pool create research_pool 128 128 erasure ec_profile
     ceph osd crush rule create-erasure slow_tier ec_profile
     ceph osd pool set research_pool crush_rule slow_tier
     ```
  4. Tích hợp Proxmox: **Storage > Add > CephFS/RBD** (ID: `ceph-storage`, Pool: realtime_pool/research_pool).

### Bước 3: Cài đặt Proxmox VE
#### 3.1. Chuẩn bị USB
- Tải ISO Proxmox VE 8.x từ [trang chính thức](https://www.proxmox.com/en/downloads).
- Ghi ISO vào USB (Windows: Rufus; Linux/Mac: `dd`).

#### 3.2. Cài đặt trên Dell R640
- Trên mỗi server (1, 2, 3, 4):
  1. Kết nối màn hình, bàn phím, USB.
  2. Bật server, vào BIOS (`F2`), chọn boot từ USB.
  3. Chọn **Install Proxmox VE**.
  4. Cấu hình mạng:
     - Server 1: `192.168.50.10/24`
     - Server 2: `192.168.50.11/24`
     - Server 3: `192.168.50.12/24`
     - Server 4: `192.168.50.13/24`
  5. Đặt mật khẩu root, cài đặt (10-15 phút).
  6. Tháo USB, khởi động lại.
  7. Kiểm tra web: `https://<IP>:8006`.
  8. Cập nhật:
     ```bash
     apt update && apt full-upgrade
     ```

#### 3.3. Cài đặt trên Supermicro 4029GP-TRT
- Tương tự:
  1. Kết nối màn hình, bàn phím, USB.
  2. Bật, vào BIOS, boot từ USB.
  3. Chọn **Install Proxmox VE**.
  4. Cấu hình mạng: `192.168.50.14/24`.
  5. Đặt mật khẩu root, cài đặt.
  6. Tháo USB, khởi động lại.
  7. Kiểm tra web: `https://192.168.50.14:8006`.
  8. Cập nhật:
     ```bash
     apt update && apt full-upgrade
     ```

#### 3.4. Tạo cụm Proxmox
- Trên server 1 (Dell R640, `192.168.50.10`):
  1. SSH:
     ```bash
     ssh root@192.168.50.10
     ```
  2. Tạo cụm:
     ```bash
     pvecm create my-cluster
     ```
- Thêm server 2, 3, 4 (Dell R640):
  1. Trên server 2:
     ```bash
     ssh root@192.168.50.11
     pvecm add 192.168.50.10
     ```
  2. Lặp lại cho server 3, 4.
- Thêm Supermicro 4029GP-TRT (`192.168.50.14`):
  ```bash
  ssh root@192.168.50.14
  pvecm add 192.168.50.10
  ```
- Kiểm tra toàn cụm:
  ```bash
  pvecm status
  ```

### Bước 4: Cấu hình GPU Passthrough trên Supermicro 4029GP-TRT
#### 4.1. Kích hoạt IOMMU
- Trên Supermicro 4029GP-TRT:
  ```bash
  nano /etc/default/grub
  ```
  Thêm:
  ```plaintext
  GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
  ```
  ```bash
  update-grub
  reboot
  ```

#### 4.2. Cấu hình PCI Passthrough
- Tìm ID GPU:
  ```bash
  lspci | grep NVIDIA
  ```
  Ví dụ: `01:00.0 VGA compatible controller: NVIDIA Corporation GP100GL [Tesla V100 PCIe 32GB]`.
- Kiểm tra IOMMU group:
  ```bash
  find /sys/kernel/iommu_groups/ -type l
  ```
- Nếu dùng VM:
  1. Chỉnh sửa file VM (ID 100):
     ```bash
     nano /etc/pve/qemu-server/100.conf
     ```
     Thêm:
     ```plaintext
     hostpci0: 01:00.0
     ```
  2. Khởi động lại VM:
     ```bash
     qm stop 100
     qm start 100
     ```
  3. Kiểm tra GPU trong VM:
     ```bash
     lspci | grep NVIDIA
     ```

### Bước 5: Cài đặt Kubernetes
#### 5.1. Cài đặt Docker và Kubernetes
- Trên tất cả node:
  ```bash
  apt update
  apt install -y docker.io kubelet kubeadm kubectl
  systemctl enable --now docker
  ```

#### 5.2. Khởi tạo Kubernetes
- Trên node 1:
  ```bash
  kubeadm init --pod-network-cidr=10.244.0.0/16
  mkdir -p $HOME/.kube
  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  ```
- Thêm node 2, 3, 4 và 4029GP-TRT:
  ```bash
  kubeadm join 192.168.50.10:6443 --token <token> --discovery-token-ca-cert-hash <hash>
  ```

#### 5.3. Cài NVIDIA GPU Operator
- Trên node 1:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/nvidia-device-plugin.yml
  ```
- Cấu hình Time-Slicing (oversubscription):
  - Chỉnh `/etc/kubernetes/nvidia-device-plugin/config.yaml`:
    ```yaml
    version: v1
    flags:
      oversubscription: true
    ```
  - Restart plugin:
    ```bash
    kubectl delete pod -n kube-system -l name=nvidia-device-plugin-ds
    ```

### Bước 6: Triển khai Ray on Kubernetes
#### 6.1. Cài Ray K8s Operator
- Trên node 1:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/ray-project/kuberay/master/ray-operator/config/default/raycluster.autoscaler.yaml
  ```
- Tạo `ray-cluster.yaml`:
  ```bash
  nano ray-cluster.yaml
  ```
  Dán code, lưu:
  ```yaml
  apiVersion: ray.io/v1
  kind: RayCluster
  metadata:
    name: ray-cluster
  spec:
    rayVersion: '2.9.0'
    headGroupSpec:
      replicas: 1
      rayStartParams:
        dashboard-host: '0.0.0.0'
        num-cpus: '2'
        num-gpus: '0'
      template:
        spec:
          containers:
          - name: ray-head
            image: rayproject/ray:2.9.0
            resources:
              limits:
                cpu: '2'
                memory: '4Gi'
              requests:
                cpu: '1'
                memory: '2Gi'
            volumeMounts:
            - name: backup-storage
              mountPath: "/backup-storage"
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
    workerGroupSpecs:
    - replicas: 5  # Workload 1: Realtime alpha inference
      minReplicas: 1
      maxReplicas: 10
      groupName: worker-group-realtime
      rayStartParams:
        num-cpus: '2'
        num-gpus: '0.1'  # Fractional-like request with MPS/Time-Slicing
      template:
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: topology.kubernetes.io/numa-node
                    operator: In
                    values: ["0"]
          tolerations:
          - key: "numa"
            operator: "Exists"
            effect: "NoSchedule"
          containers:
          - name: ray-worker
            image: rayproject/ray:2.9.0
            resources:
              limits:
                cpu: '8'
                memory: '16Gi'
                nvidia.com/gpu: 1  # Logical limit, shared via MPS
              requests:
                cpu: '1'
                memory: '4Gi'
                nvidia.com/gpu: 0.1  # Fractional-like request
            env:
            - name: RAY_USE_NUMA_AWARE_SCHEDULING
              value: "1"
            - name: NVIDIA_VISIBLE_DEVICES
              value: "all"  # Enable MPS sharing
            - name: NVIDIA_DRIVER_CAPABILITIES
              value: "compute,utility"
            - name: NVIDIA_MPS_ACTIVE_THREAD_PERCENTAGE
              value: "10"  # Limit to ~10% GPU per process
            command: ["/bin/sh", "-c", "numactl --cpunodebind=0 --membind=0 ray start --address=$RAY_HEAD_SERVICE_HOST:$RAY_HEAD_SERVICE_PORT --block"]
          extraContainers:
          - name: mps-daemon
            image: nvidia/cuda:11.8.0-base-ubuntu20.04
            command: ["/bin/bash", "-c", "nvidia-cuda-mps-control -d"]
            securityContext:
              privileged: true
            volumeMounts:
            - name: mps-volume
              mountPath: /tmp/nvidia-mps
          volumes:
          - name: mps-volume
            emptyDir: {}
    - replicas: 2  # Workload 2: Research training
      minReplicas: 1
      maxReplicas: 5
      groupName: worker-group-training
      rayStartParams:
        num-cpus: '6'
        num-gpus: '1'  # Full GPU allocation
      template:
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: topology.kubernetes.io/numa-node
                    operator: In
                    values: ["0"]
          tolerations:
          - key: "numa"
            operator: "Exists"
            effect: "NoSchedule"
          containers:
          - name: ray-worker
            image: rayproject/ray:2.9.0
            resources:
              limits:
                cpu: '8'
                memory: '16Gi'
                nvidia.com/gpu: 1  # Full GPU
              requests:
                cpu: '1'
                memory: '4Gi'
                nvidia.com/gpu: 1  # Full request
            env:
            - name: RAY_USE_NUMA_AWARE_SCHEDULING
              value: "1"
            command: ["/bin/sh", "-c", "numactl --cpunodebind=0 --membind=0 ray start --address=$RAY_HEAD_SERVICE_HOST:$RAY_HEAD_SERVICE_PORT --block"]
            volumeMounts:
            - name: backup-storage
              mountPath: "/backup-storage"
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
  ```
- Áp dụng:
  ```bash
  kubectl apply -f ray-cluster.yaml
  ```

#### 6.2. Cấu hình MPS trên node GPU
- Trên Supermicro 4029GP-TRT:
  1. Chuyển GPU sang compute mode:
     ```bash
     nvidia-smi -c 1
     ```
  2. Khởi động MPS daemon:
     ```bash
     nvidia-cuda-mps-control -d
     ```
  3. Kiểm tra:
     ```bash
     nvidia-smi
     ```

#### 6.3. Cài đặt và Cấu hình Horizontal Pod Autoscaler (HPA)
- **Cài đặt Metrics Server** (để HPA theo dõi CPU/memory):
  - Trên node 1:
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```
  - Chỉnh file `components.yaml` (nếu cần) để thêm:
    ```yaml
    args:
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP
    ```
  - Restart Metrics Server:
    ```bash
    kubectl delete pod -n kube-system -l k8s-app=metrics-server
    ```

- **Cấu hình HPA cho `worker-group-realtime`** (Workload 1):
  - Tạo file `hpa-realtime.yaml`:
    ```bash
    nano hpa-realtime.yaml
    ```
    Dán code, lưu:
    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: ray-worker-realtime-hpa
    spec:
      scaleTargetRef:
        apiVersion: ray.io/v1
        kind: RayCluster
        name: ray-cluster
      minReplicas: 1
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70  # Scale khi CPU usage > 70%
      - type: Resource
        resource:
          name: memory
          target:
            type: Utilization
            averageUtilization: 80  # Scale khi memory usage > 80%
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 30
          policies:
          - type: Pods
            value: 2
            periodSeconds: 15
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
          - type: Pods
            value: 1
            periodSeconds: 300
    ```
  - Áp dụng:
    ```bash
    kubectl apply -f hpa-realtime.yaml
    ```

- **Cấu hình HPA cho `worker-group-training`** (Workload 2):
  - Tạo file `hpa-training.yaml`:
    ```bash
    nano hpa-training.yaml
    ```
    Dán code, lưu:
    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: ray-worker-training-hpa
    spec:
      scaleTargetRef:
        apiVersion: ray.io/v1
        kind: RayCluster
        name: ray-cluster
      minReplicas: 1
      maxReplicas: 5
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 80  # Scale khi CPU usage > 80%
      - type: Resource
        resource:
          name: nvidia.com/gpu
          target:
            type: Utilization
            averageUtilization: 90  # Scale khi GPU usage > 90% (nếu có metrics GPU)
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 60
          policies:
          - type: Pods
            value: 1
            periodSeconds: 30
        scaleDown:
          stabilizationWindowSeconds: 600
          policies:
          - type: Pods
            value: 1
            periodSeconds: 600
    ```
  - Áp dụng:
    ```bash
    kubectl apply -f hpa-training.yaml
    ```

- **Lưu ý**: 
  - Hiện tại, Metrics Server chỉ hỗ trợ CPU/memory. Để monitor GPU utilization, cần cài thêm NVIDIA GPU Metrics Exporter (DCGM-Exporter) và cấu hình custom metrics với Prometheus (xem Bước 13).
  - Kiểm tra HPA:
    ```bash
    kubectl get hpa
    ```

### Bước 7: Triển khai JupyterHub trên Kubernetes
#### 7.1. Tạo Docker image
- Trên node 1:
  ```bash
  mkdir jupyterhub-docker
  cd jupyterhub-docker
  nano Dockerfile
  ```
  Dán code, lưu:
  ```dockerfile
  FROM jupyter/base-notebook:latest
  RUN pip install jupyterhub jupyterlab numpy pandas matplotlib scikit-learn tensorflow ray[train,serve,data] cudf-cu11 cuml-cu11 --extra-index-url=https://pypi.nvidia.com
  RUN npm install -g configurable-http-proxy
  RUN apt update && apt install -y nvidia-driver-535 nvidia-utils-535
  ```
- Xây dựng:
  ```bash
  docker build -t jupyterhub-custom:latest .
  ```

#### 7.2. Cấu hình JupyterHub
- Tạo `jupyterhub_config.py`:
  ```bash
  nano jupyterhub_config.py
  ```
  Dán code, lưu:
  ```python
  c.JupyterHub.bind_url = 'http://0.0.0.0:8000'
  c.JupyterHub.authenticator_class = 'pam'
  c.Spawner.args = ['--LabApp.disable_clipboard=True']
  c.JupyterHub.spawner_class = 'kubespawner.KubeSpawner'
  c.KubeSpawner.image = 'jupyterhub-custom:latest'

  # Dynamic resource allocation
  c.KubeSpawner.cpu_limit = 8  # Giới hạn tối đa khi chạy mã
  c.KubeSpawner.mem_limit = '16G'  # Giới hạn tối đa khi chạy mã
  c.KubeSpawner.cpu_request = 0.5  # Tài nguyên cơ bản khi idle
  c.KubeSpawner.mem_request = '1G'  # Tài nguyên cơ bản khi idle

  # Hooks để tăng tài nguyên khi chạy mã (dynamic)
  def pre_spawn_hook(spawner):
      if spawner.user.options.get('run_code', False):
          spawner.cpu_request = 6.5  # Tăng CPU khi chạy
          spawner.mem_request = '12G'  # Tăng RAM khi chạy
      else:
          spawner.cpu_request = 0.5  # Giữ thấp khi idle
          spawner.mem_request = '1G'  # Giữ thấp khi idle
  c.KubeSpawner.pre_spawn_hook = pre_spawn_hook

  # Cấu hình profile để user chọn chế độ chạy mã
  c.KubeSpawner.profile_list = [{
      'display_name': 'Python with GPU and Ray (Idle)',
      'default': True,
      'kubespawner_override': {
          'image': 'jupyterhub-custom:latest',
          'environment': {'JUPYTERHUB_SINGLEUSER_APP': 'jupyterlab', 'RAY_ADDRESS': 'ray://ray-cluster-head-svc:10001'}
      }
  }, {
      'display_name': 'Python with GPU and Ray (Run Code)',
      'kubespawner_override': {
          'image': 'jupyterhub-custom:latest',
          'environment': {'JUPYTERHUB_SINGLEUSER_APP': 'jupyterlab', 'RAY_ADDRESS': 'ray://ray-cluster-head-svc:10001', 'run_code': 'true'}
      }
  }]

  # Cấu hình multi-GPU sharing với NVIDIA MPS
  c.KubeSpawner.extra_resource_limits = {'nvidia.com/gpu': '1'}  # Giới hạn 1 GPU logic
  c.KubeSpawner.extra_resource_requests = {'nvidia.com/gpu': '0.1'}  # Request fractional-like khi idle
  c.KubeSpawner.environment = {
      'NVIDIA_VISIBLE_DEVICES': 'all',  # Cho phép thấy tất cả GPU
      'NVIDIA_DRIVER_CAPABILITIES': 'compute,utility',  # Cấu hình driver
      'NVIDIA_MPS_ACTIVE_THREAD_PERCENTAGE': '10',  # Giới hạn 10% GPU/process
  }

  # Cấu hình GPU sharing qua MPS (cần setup node trước)
  c.KubeSpawner.extra_containers = [{
      'name': 'mps-container',
      'image': 'nvidia/cuda:11.8.0-base-ubuntu20.04',
      'command': ['/bin/bash', '-c', 'nvidia-cuda-mps-control -d'],  # Khởi động MPS daemon
      'securityContext': {'privileged': True},
      'volumeMounts': [{
          'name': 'mps-volume',
          'mountPath': '/tmp/nvidia-mps'
      }]
  }]
  c.KubeSpawner.volumes = [{
      'name': 'mps-volume',
      'emptyDir': {}
  }]

  c.KubeSpawner.start_timeout = 300
  c.KubeSpawner.pvc_name = 'backup-pvc'
  ```
- Tạo `jupyterhub-deployment.yaml`:
  ```bash
  nano jupyterhub-deployment.yaml
  ```
  Dán code, lưu:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: jupyterhub
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: jupyterhub
    template:
      metadata:
        labels:
          app: jupyterhub
      spec:
        containers:
        - name: jupyterhub
          image: jupyterhub-custom:latest
          ports:
          - containerPort: 8000
          resources:
            limits:
              cpu: "8"
              memory: "16Gi"
              nvidia.com/gpu: 1
            requests:
              cpu: "0.5"  # Giảm khi idle
              memory: "1Gi"  # Giảm khi idle
              nvidia.com/gpu: 0.1  # Giảm GPU request khi idle
          command: ["jupyterhub", "-f", "/etc/jupyterhub/jupyterhub_config.py"]
          volumeMounts:
          - name: backup-storage
            mountPath: "/backup-storage"
          - name: mps-volume
            mountPath: "/tmp/nvidia-mps"
        volumes:
        - name: backup-storage
          persistentVolumeClaim:
            claimName: backup-pvc
        - name: mps-volume
          emptyDir: {}
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: jupyterhub-service
  spec:
    selector:
      app: jupyterhub
    ports:
    - port: 8000
    type: ClusterIP
  ```
- Áp dụng:
  ```bash
  kubectl apply -f jupyterhub-deployment.yaml
  ```

#### 7.3. Mount SSD Micron vào JupyterHub
- Tạo `backup-pv.yaml`:
  ```bash
  nano backup-pv.yaml
  ```
  Dán code, lưu:
  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: backup-pv
  spec:
    capacity:
      storage: 1.92Ti
    accessModes:
      - ReadWriteMany
    hostPath:
      path: "/backup-storage"
  ---
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: backup-pvc
  spec:
    accessModes:
      - ReadWriteMany
    storageClassName: ""
    resources:
      requests:
        storage: 1.92Ti
  ```
- Áp dụng:
  ```bash
  kubectl apply -f backup-pv.yaml
  ```

### Bước 8: Triển khai Apache
#### 8.1. Tạo Docker image
- Trên node 1:
  ```bash
  mkdir apache-docker
  cd apache-docker
  nano Dockerfile
  ```
  Dán code, lưu:
  ```dockerfile
  FROM httpd:2.4
  RUN apt update && apt install -y libapache2-mod-proxy-html
  RUN a2enmod proxy proxy_http proxy_wstunnel rewrite ssl
  COPY jupyterhub.conf /usr/local/apache2/conf/extra/jupyterhub.conf
  RUN echo "Include conf/extra/jupyterhub.conf" >> /usr/local/apache2/conf/httpd.conf
  ```
- Tạo `jupyterhub.conf`:
  ```bash
  nano jupyterhub.conf
  ```
  Dán code, lưu:
  ```apache
  <VirtualHost *:80>
      ServerName jupyterhub.example.com
      ProxyPreserveHost On
      ProxyPass / http://jupyterhub-service:8000/
      ProxyPassReverse / http://jupyterhub-service:8000/
      RewriteEngine On
      RewriteCond %{HTTP:Upgrade} websocket [NC]
      RewriteCond %{HTTP:Connection} upgrade [NC]
      RewriteRule ^/(.*) ws://jupyterhub-service:8000/$1 [P,L]
      <Location />
          AuthType Basic
          AuthName "Restricted Access"
          AuthUserFile /usr/local/apache2/conf/.htpasswd
          Require valid-user
      </Location>
  </VirtualHost>
  ```
- Tạo `.htpasswd`:
  ```bash
  htpasswd -c .htpasswd researcher1
  ```
  (Nhập mật khẩu khi được hỏi, lặp lại cho researcher2: `htpasswd .htpasswd researcher2`).
- Xây dựng:
  ```bash
  docker build -t apache-jupyter:latest .
  ```

#### 8.2. Triển khai Apache
- Tạo `apache-deployment.yaml`:
  ```bash
  nano apache-deployment.yaml
  ```
  Dán code, lưu:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: apache
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: apache
    template:
      metadata:
        labels:
          app: apache
      spec:
        containers:
        - name: apache
          image: apache-jupyter:latest
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: apache-service
  spec:
    selector:
      app: apache
    ports:
    - port: 80
    type: LoadBalancer
  ```
- Áp dụng:
  ```bash
  kubectl apply -f apache-deployment.yaml
  ```

### Bước 9: Tích hợp Ray với GPU
- Trong JupyterHub:
  ```python
  import ray
  import tensorflow as tf
  import time
  import numpy as np

  # Khởi tạo Ray cluster
  ray.init(address='ray://ray-cluster-head-svc:10001')
  print(f"Ray cluster connected at {time.ctime()} (10:18 AM +07, 21/09/2025)")
  print("Available GPUs:", tf.config.list_physical_devices('GPU'))

  # Test Workload 1: 50,000 alpha realtime (small tasks, 0.1 GPU)
  @ray.remote(num_cpus=2, num_gpus=0.1)
  def compute_alpha(data):
      model = tf.keras.models.load_model("alpha_model.h5")  # Load mô hình đã train
      time.sleep(0.001)  # Simulasi tính toán 1ms
      return model.predict(data)  # Inference realtime

  # Tạo dữ liệu giả lập cho 50,000 alpha
  data = [np.random.rand(1000) for _ in range(50000)]  # 50,000 tasks, mỗi task <2MB
  start_time = time.time()
  results = ray.get([compute_alpha.remote(d) for d in data])
  workload1_time = time.time() - start_time
  print(f"Workload 1 completed in {workload1_time:.2f} seconds for 50,000 alpha tasks")

  # Test Workload 2: Research training (6 CPU, 1 GPU)
  @ray.remote(num_cpus=6, num_gpus=1)
  def train_model(data):
      model = tf.keras.Sequential([
          tf.keras.layers.Dense(128, activation='relu'),
          tf.keras.layers.Dense(10, activation='softmax'
      ])
      model.compile(optimizer='adam', loss='mse')
      model.fit(data, np.zeros((len(data), 10)), epochs=1, batch_size=32)
      return model.evaluate(data, np.zeros((len(data), 10)))

  # Tạo dữ liệu giả lập cho training (~100,000 datapoint)
  large_data = np.random.rand(100000, 10)  # >20MB
  start_time = time.time()
  result = ray.get(train_model.remote(large_data))
  workload2_time = time.time() - start_time
  print(f"Workload 2 completed in {workload2_time:.2f} seconds with 6 CPU and 1 GPU")

  # Kiểm tra tài nguyên
  print("Ray cluster status:", ray.cluster_resources())
  ```

### Bước 10: Đảm bảo HA và mở rộng
- **HA**: Kubernetes tự động di chuyển pod nếu node lỗi. SSD Micron và NVMe đảm bảo dữ liệu bền vững. Ray K8s Operator hỗ trợ autoscaling.
- **Mở rộng**: Tăng replicas trong `ray-cluster.yaml` hoặc thêm workstation. Switch 10Gbps và NVMe hỗ trợ băng thông/I/O cao.

### Bước 11: Đảm bảo bảo mật
- Clipboard vô hiệu hóa trong `jupyterhub_config.py`.
- NetworkPolicy:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: jupyterhub-policy
  spec:
    podSelector:
      matchLabels:
        app: jupyterhub
    ingress:
    - from:
      - ipBlock:
          cidr: 192.168.10.0/24
      ports:
      - protocol: TCP
        port: 8000
  ```
- Áp dụng:
  ```bash
  kubectl apply -f network-policy.yaml
  ```
- Bảo mật SSD Micron:
  - Phân quyền folder trên ZFS.
  - Bật mã hóa ZFS.

### Bước 12: Kiểm tra hệ thống
- Truy cập `http://jupyterhub.example.com` từ VLAN 10.
- Chạy notebook với Ray và GPU (xem Bước 9).
- Kiểm tra SSD Micron:
  - Trong JupyterHub, mở file từ `/backup-storage`.
- Kiểm tra NVMe:
  ```bash
  iostat -x 1
  ```
  Đảm bảo tốc độ đọc/ghi cao (~8000MB/s).
- Kiểm tra cache:
  ```bash
  zpool status storage
  zpool status nvme-storage
  ```
- **Xác minh HA**:
  1. Đăng nhập Proxmox web (`https://192.168.50.10:8006`).
  2. Chọn node 2 (`192.168.50.11`), vào **Shutdown** hoặc ngắt nguồn.
  3. Kiểm tra pod:
     ```bash
     kubectl get pods -o wide
     ```
     Đảm bảo pod JupyterHub/Apache/Ray di chuyển sang node khác.
  4. Truy cập `http://jupyterhub.example.com`, chạy notebook, kiểm tra SSD Micron.
  5. Kiểm tra Proxmox:
     ```bash
     pvecm status
     ```
  6. Kiểm tra Kubernetes:
     ```bash
     kubectl get nodes
     ```
     Node 2 phải ở trạng thái `NotReady`.
  7. Bật lại node 2, kiểm tra:
     ```bash
     pvecm status
     kubectl get nodes
     ```
- Kiểm tra tốc độ mạng:
  ```bash
  iperf3 -c 192.168.50.50
  ```
  Đảm bảo tốc độ ~10Gbps.
- **Kiểm tra MPS/Time-Slicing**:
  1. Chạy test Workload 1 (50,000 alpha) trong notebook.
  2. Kiểm tra `nvidia-smi` trên Supermicro: Xác nhận 5-10 process share GPU, mỗi process ~10% util.
  3. Chạy test Workload 2 (training), kiểm tra `nvidia-smi`: Xác nhận 80-90% GPU util.
- **Kiểm tra HPA**:
  1. Tạo tải giả lập cho Workload 1 (e.g., tăng số alpha lên 100,000 trong test code).
  2. Kiểm tra:
     ```bash
     kubectl get hpa
     kubectl describe hpa ray-worker-realtime-hpa
     ```
     Xác nhận replicas tăng từ 5 lên max 10 nếu CPU/memory vượt ngưỡng.
  3. Tạo tải giả lập cho Workload 2 (e.g., chạy nhiều training job).
  4. Kiểm tra:
     ```bash
     kubectl describe hpa ray-worker-training-hpa
     ```
     Xác nhận replicas tăng từ 2 lên max 5 nếu CPU/GPU vượt ngưỡng.

### Bước 13: Tối ưu hóa và sao lưu
- **Sao lưu**: Backup lưu trên SSD Micron (`/backup-storage/backup`), snapshot sao chép sang SSD (`/backup-storage/snapshots`).
- **Giám sát**: Cài Prometheus:
  ```bash
  kubectl apply -f https://github.com/prometheus-operator/kube-prometheus/releases/download/v0.14.0/bundle.yaml
  ```

- **Bổ sung NVIDIA GPU Metrics Exporter (DCGM-Exporter)**:
  - Cài Helm nếu chưa có:
    ```bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```
  - Thêm repo Helm và cài DCGM-Exporter:
    ```bash
    helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
    helm repo update
    helm install dcgm-exporter gpu-helm-charts/dcgm-exporter --namespace monitoring --create-namespace
    ```
  - Cấu hình custom metrics với Prometheus:
    - Tạo file `service-monitor.yaml`:
      ```bash
      nano service-monitor.yaml
      ```
      Dán code, lưu:
      ```yaml
      apiVersion: monitoring.coreos.com/v1
      kind: ServiceMonitor
      metadata:
        name: dcgm-exporter
        namespace: monitoring
        labels:
          release: prometheus
      spec:
        selector:
          matchLabels:
            app.kubernetes.io/name: dcgm-exporter
        endpoints:
        - port: metrics
          interval: 15s
      ```
    - Áp dụng:
      ```bash
      kubectl apply -f service-monitor.yaml
      ```
  - Kiểm tra metrics:
    ```bash
    kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090
    ```
    Truy cập http://localhost:9090, query `DCGM_FI_DEV_GPU_UTIL` để xem GPU utilization.

- **Cấu hình HPA với custom metrics GPU** (nếu cần):
  - Sử dụng adapter cho custom metrics (e.g., Prometheus Adapter):
    ```bash
    helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring --set prometheus.url=http://prometheus-operated.monitoring.svc:9090
    ```
  - Update HPA training với GPU metric:
    ```yaml
    metrics:
    - type: Pods
      pods:
        metric:
          name: dcgm_gpu_utilization  # Custom metric từ DCGM
        target:
          type: AverageValue
          averageValue: 90
    ```
  - Áp dụng HPA mới.

## Lưu ý
- **SSD Micron Backup**: Tốt hơn NAS cho tốc độ (560MB/s vs 200MB/s HDD), thấp độ trễ, tốt cho backup NVMe nhanh. Với 4 NVMe (6.4TB), cần 4-5 SSD Micron (RAID 5, dung lượng khả dụng ~5.76TB). Không cần nếu NVMe RAID, nhưng khuyến nghị cho backup dữ liệu ML/AI.
- **CPU**: Xeon Platinum 8173M (Cascade Lake-SP, thế hệ 2) tốt hơn 8168 (Skylake-SP, thế hệ 1) 10-15% ở ML/AI.
- **SSD/NVMe Cache**: L2ARC/ZIL tăng tốc I/O.
- **GPU Passthrough**: Mã ID GPU (`lspci | grep NVIDIA`) dùng cho `hostpci0` in VM (nếu cần).
- **JupyterHub**: Chạy trên Kubernetes, tích hợp Ray, không cài trực tiếp trên node/workstation.
- Switch 10Gbps và NVMe tối ưu truyền dữ liệu/I/O.
- VLAN 50 cho management tăng bảo mật.
- User trên VLAN 10 không truy cập trực tiếp SSD Micron; mount qua JupyterHub.
- User ở site khác cần VPN.
