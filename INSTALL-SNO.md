# Hướng Dẫn Cài Đặt OpenNebula 7.4 SNO (Single Node OpenNebula)

Tài liệu này hướng dẫn chi tiết quy trình triển khai **OpenNebula 7.4 SNO** (Single Node OpenNebula / All-in-One) lên **Mini PC Ubuntu** bằng bộ công cụ tự động hóa chính thức **`one-deploy` (Ansible)**.

Xem thêm [Flow và cơ chế hoạt động của SNO với Ansible](docs/SNO-ANSIBLE-FLOW.md) để hiểu thứ tự role và hành vi của mã hiện tại. Một số ví dụ bên dưới dùng cấu hình cũ; tài liệu flow chỉ rõ khác biệt về Prometheus, tên host đăng ký, endpoint OneKS và phần gateway/NAT private cần cấu hình riêng.

---

## 1. Thông Số Hệ Thống & Kiến Trúc SNO

### 1.1. Thông số môi trường Mini PC
| Thông số | Giá trị |
| :--- | :--- |
| **Hostname** | `mini-ubuntu` |
| **Địa chỉ IP** | `192.168.250.3/24` |
| **Default Gateway** | `192.168.250.1` |
| **Card mạng vật lý** | `eno1` |
| **User quản trị** | `mini` (có quyền `sudo`) |
| **SSH Host Alias** | `ssh mini-ubuntu` (đã cấu hình sẵn SSH Key) |
| **Hệ điều hành** | Ubuntu 26.04 / 24.04 LTS (x86_64) |
| **Phần cứng** | AMD Ryzen 7 8745HS (16 vCPU), 64GB RAM, NVMe SSD |
| **Hỗ trợ ảo hóa** | AMD-V KVM (`/dev/kvm` sẵn sàng) |

### 1.2. Mô hình Single Node (SNO)
Trong mô hình SNO, chiếc Mini PC sẽ đảm nhiệm **cả 2 vai trò cùng lúc**:
1. **OpenNebula Frontend**:
   - `oned` (OpenNebula Core Daemon).
   - Cơ sở dữ liệu: **MariaDB**.
   - Giao diện Web quản trị: **FireEdge Sunstone** (kèm Nginx reverse proxy SSL).
   - Các dịch vụ bổ trợ: OneGate, OneFlow, OneForm, OneKS (Kubernetes Service).
2. **OpenNebula KVM Node (Hypervisor)**:
      - Mạng ảo Public/LAN gắn kết qua Linux Bridge `br0` (nối trực tiếp ra mạng LAN `192.168.250.0/24` qua card `eno1`).
    - Mạng ảo Private cho Kubernetes (OneKS) qua Linux Bridge `br-priv` (`172.20.0.0/24`) với NAT Masquerade ra internet.

```
 +-------------------------------------------------------------------------+
 |                      MINI PC (mini-ubuntu: 192.168.250.3)               |
 |                                                                         |
 |  [ OpenNebula Frontend ]                 [ OpenNebula KVM Node ]        |
 |   - oned Core                            - Libvirt / QEMU-KVM           |
 |   - MariaDB Database                     - Datastores (0, 1, 2)         |
 |   - FireEdge Sunstone GUI (:2616 / :443) - VMs / K8s Cluster Nodes      |
 |   - OneGate / OneFlow / OneForm / OneKS                                 |
 |          |                     \         /                      |       |
 |          |                      \       /                       |       |
 |          |                   [ Bridge: br0 ]                    |       |
 |          |                 (192.168.250.3/24)                   |       |
 |          |                          |                           |       |
 |          |                 [ Card vật lý: eno1 ]                |       |
 |          |                          |                           |       |
 |          |       [ LAN Router / Gateway: 192.168.250.1 ]        |       |
 |          |                                                      |       |
 |          +------------------- [ Bridge: br-priv ] --------------+       |
 |                             (172.20.0.1/24 - NAT br0)                   |
 |                        Dành riêng cho cụm OneKS K8s                     |
 +-------------------------------------------------------------------------+
```

---

## 2. Chuẩn Bị Tiên Quyết (Prerequisites)

### 2.1. Cấu hình trên Mini PC (`mini-ubuntu`)

#### Bước 2.1.1: Cấu hình Sudo không cần mật khẩu (NOPASSWD)
Để Ansible có thể tự động cài đặt gói, cấu hình mạng và khởi động dịch vụ mà không bị gián đoạn do yêu cầu mật khẩu tương tác:

Đăng nhập vào Mini PC qua SSH:
```bash
ssh mini-ubuntu
```

Chạy lệnh cấp quyền `NOPASSWD` cho user `mini`:
```bash
echo "mini ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/mini
sudo chmod 0440 /etc/sudoers.d/mini
```

Kiểm tra lại:
```bash
sudo -n true && echo ">>> Cấu hình sudo thành công!"
exit
```

> [!IMPORTANT]
> Nếu không cấu hình `NOPASSWD`, bạn bắt buộc phải truyền cờ `-K` (hoặc `--ask-become-pass`) trong tất cả các lệnh `ansible` và `ansible-playbook` để nhập mật khẩu `sudo` bằng tay.

#### Bước 2.1.2: Kiểm tra KVM Virtualization
Đảm bảo KVM đã được kích hoạt trong BIOS:
```bash
ssh mini-ubuntu "ls -l /dev/kvm && egrep -c '(vmx|svm)' /proc/cpuinfo"
```
Kết quả hiển thị `/dev/kvm` và số CPU core (>= 1) nghĩa là ảo hóa phần cứng đã sẵn sàng.

---

### 2.2. Chuẩn bị môi trường trên máy điều khiển (Macbook)

Tại thư mục mã nguồn `one-deploy`:
```bash
cd /Users/timi/lab/lab-opennebula/one-deploy
```

#### Bước 2.2.1: Tạo Python Virtualenv và cài đặt dependencies
Sử dụng `uv` (hoặc `python3 -m venv`):
```bash
# Tạo môi trường ảo .venv
uv venv .venv

# Kích hoạt virtualenv
source .venv/bin/activate

# Cài đặt ansible-core và các thư viện cần thiết
uv pip install -r requirements.txt ansible-core
```

#### Bước 2.2.2: Cài đặt các Ansible Galaxy Collections
Dự án yêu cầu các collection của Ansible để quản trị KVM, mã hóa và hệ thống:
```bash
ansible-galaxy collection install -r requirements.yml -p ansible_collections
```

#### Bước 2.2.3: (Khuyến nghị) Thêm Static Route trên macOS để truy cập mạng Private VM
Mạng Private `oneks_priv` (`172.20.0.0/24`) nằm sau Mini PC. Để máy Mac có thể ping và SSH trực tiếp vào các máy ảo thuộc dải này:
```bash
sudo route -n add -net 172.20.0.0/24 192.168.250.3
```
*(Chi tiết cách kiểm tra, xóa hoặc lưu vĩnh viễn xem tại [Bước 6.5](#bước-65-định-tuyến-từ-macos-vào-mạng-private-172200024)).*

---

## 3. Cấu Hình Inventory Cho SNO (`inventory/sno.yml`)

File cấu hình inventory đã được tạo sẵn tại `inventory/sno.yml`:

```yaml
---
all:
  vars:
    # --- SSH & User Settings ---
    ansible_user: mini
    ansible_python_interpreter: /usr/bin/python3
    ensure_keys_for: [mini, root]

    # --- OpenNebula Core Settings ---
    one_version: '7.4'
    # Mật khẩu tài khoản quản trị oneadmin (Web & CLI)
    one_pass: 'OpenNebula@2026'

    # OpenNebula Token:
    # - Để trống cho bản Community Edition (miễn phí từ downloads.opennebula.io)
    # - Điền '<user>:<token>' nếu bạn có license OpenNebula Enterprise
    # one_token: ''

    # --- Database ---
    db_backend: MariaDB
    db_name: opennebula
    db_owner: oneadmin
    db_password: 'OneDbPassword@2026'

    # --- Storage Datastores (Local Storage) ---
    # Chế độ 'ssh': Lưu trữ trực tiếp trên thư mục /var/lib/one/datastores cục bộ
    ds:
      mode: ssh

    # --- Mạng Ảo & Linux Bridge (br0 & br-priv) ---
    # - lan_net (Public/LAN): Gắn với card eno1 qua bridge br0 (Dải LAN 192.168.250.0/24)
    # - oneks_priv (Private): Dành cho cụm Kubernetes OneKS qua bridge br-priv (172.20.0.0/24)
    vn:
      lan_net:
        managed: true
        move_ip: true
        template:
          VN_MAD: bridge
          PHYDEV: eno1
          BRIDGE: br0
          # Dải IP cấp cho các Virtual Machine (192.168.250.200 - 192.168.250.229)
          AR:
            TYPE: IP4
            IP: 192.168.250.200
            SIZE: 30
          NETWORK_ADDRESS: 192.168.250.0
          NETWORK_MASK: 255.255.255.0
          GATEWAY: 192.168.250.1
          DNS: 192.168.250.1 1.1.1.1

      oneks_priv:
        managed: true
        move_ip: false
        template:
          VN_MAD: bridge
          BRIDGE: br-priv
          # Dải IP cấp nội bộ cho cụm Kubernetes (Control Plane, Worker, Pods)
          AR:
            TYPE: IP4
            IP: 172.20.0.10
            SIZE: 240
          NETWORK_ADDRESS: 172.20.0.0
          NETWORK_MASK: 255.255.255.0
          GATEWAY: 172.20.0.1
          DNS: 192.168.250.1 1.1.1.1

    # --- Các tính năng ---
    features:
      gui: true          # Web UI FireEdge Sunstone
      gate: true         # OneGate
      flow: true         # OneFlow
      form: true         # OneForm (hỗ trợ từ OpenNebula 7.2+)
      ks: true           # OneKS Kubernetes Service (hỗ trợ từ OpenNebula 7.2+)
      prometheus: false  # Tắt mặc định, bật nếu cần monitor metrics
      ceph: false
      evpn: false

    # --- Cấu hình TProxy (Bắt buộc cho OneKS & OneGate) ---
    gate_tproxy:
      - :service_port: 5030
        :remote_addr: 192.168.250.3
        :remote_port: 5030
      - :service_port: 2633
        :remote_addr: 192.168.250.3
        :remote_port: 2633

    # --- Reverse Proxy SSL (Nginx) ---
    # Tự động tạo chứng chỉ SSL tự ký và cấu hình Nginx lắng nghe HTTPS cổng 443
    ssl:
      web_server: nginx
      generate_cert: true
      key: /etc/one/fireedge-pki/key/server.key
      certchain: /etc/one/fireedge-pki/crt/certchain.crt

    # Tùy biến tên hiển thị chứng chỉ PKI (Common Name hiển thị trong Keychain / Trình duyệt)
    pki:
      ca:
        common_name: 'OpenNebula Lab Root CA'
      server:
        common_name: 'OpenNebula Lab Server'

# Khai báo SNO: Cả frontend và node đều trỏ vào mini-ubuntu
frontend:
  hosts:
    mini-ubuntu:
      ansible_host: 192.168.250.3

node:
  hosts:
    mini-ubuntu:
      ansible_host: 192.168.250.3
```

> [!TIP]
> **Về cấu hình mạng Bridge (`br0`)**:
> - Khi `move_ip: true`, `one-deploy` sẽ tự động cập nhật cấu hình **Netplan** trên Ubuntu, gán card `eno1` làm port của bridge `br0` và di chuyển địa chỉ IP `192.168.250.3` sang interface `br0`.
> - Nếu bạn muốn tự tạo bridge `br0` thủ công trên máy Ubuntu trước khi chạy Ansible, hãy đặt `move_ip: false`.

---

## 4. Tiến Hành Cài Đặt (Deployment Steps)

Thực hiện toàn bộ các lệnh sau từ máy điều khiển (Macbook) bên trong virtualenv (`source .venv/bin/activate`):

### Bước 4.1: Kiểm tra kết nối Ansible
Kiểm tra khả năng kết nối và quyền sudo:
```bash
ansible -i inventory/sno.yml all -m ping
```
*Kết quả trả về `"ping": "pong"` và `SUCCESS` là đạt.*

---

### Bước 4.2: Chạy Playbook chuẩn bị hệ thống (`pre.yml`)
Playbook này sẽ cấu hình kho lưu trữ (repository), cài đặt Python dependencies, thiết lập `/etc/hosts`, cấu hình thời gian NTP và chuẩn bị kernel:
```bash
ansible-playbook -i inventory/sno.yml playbooks/pre.yml
```

> [!NOTE]
> Trong quá trình chạy `pre.yml`, nếu hệ thống cần cập nhật kernel hoặc module mạng, server có thể tự reboot nếu được yêu cầu. Chờ server khởi động lại rồi tiếp tục bước tiếp theo.

---

### Bước 4.3: Chạy Playbook triển khai chính (`site.yml`)
Playbook này sẽ cài đặt toàn bộ:
1. **MariaDB Database**: Tạo database `opennebula` và user `oneadmin`.
2. **OpenNebula 7.4 Server**: Cài `opennebula`, `opennebula-gate`, `opennebula-flow`, `opennebula-form`.
3. **FireEdge Sunstone**: Giao diện Web GUI và cấu hình Nginx SSL reverse proxy.
4. **KVM Node**: Cài đặt `opennebula-node-kvm`, `libvirt`, cấu hình ssh key cho `oneadmin`.
5. **Datastores & Networks**: Tự động cấu hình Datastores cục bộ (0, 1, 2) và Virtual Network `lan_net` gắn với `br0`.
6. **Đăng ký Node**: Tự động thêm `mini-ubuntu` vào danh sách Host của OpenNebula (`onehost create`).

Chạy lệnh:
```bash
ansible-playbook -i inventory/sno.yml playbooks/site.yml
```

> [!TIP]
> Bạn cũng có thể chạy toàn bộ quy trình (`pre` + `site`) trong 1 lệnh duy nhất:
> ```bash
> ansible-playbook -i inventory/sno.yml playbooks/main.yml
> ```

---

## 5. Nghiệm Thu & Kiểm Tra Sau Cài Đặt

### 5.1. Kiểm tra trạng thái dịch vụ trên Mini PC
SSH vào Mini PC:
```bash
ssh mini-ubuntu
```

Kiểm tra trạng thái các dịch vụ quan trọng:
```bash
sudo systemctl status opennebula opennebula-ks opennebula-fireedge opennebula-gate opennebula-flow mariadb libvirtd nginx
```
Tất cả các dịch vụ phải ở trạng thái `active (running)`.

---

### 5.2. Kiểm tra OpenNebula qua CLI (bằng user `oneadmin`)
Đổi sang user `oneadmin`:
```bash
sudo su - oneadmin
```

Kiểm tra dịch vụ OneKS Kubernetes:
```bash
oneks list clusters
```
*Lệnh thực thi thành công trả về bảng danh sách cụm Kubernetes (hiện tại chưa có cụm nào).*

Kiểm tra danh sách Hypervisor Nodes:
```bash
onehost list
```
*Kết quả mẫu:*
```
  ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT
   0 mini-ubuntu     default     0       0 / 1600 (0%)        0K / 59G (0%)   on
```
> Trạng thái **`STAT`** phải là **`on`**. Nếu là `init` hoặc `update`, đợi khoảng 30 giây để OpenNebula hoàn thành polling metric.

Kiểm tra danh sách Datastores:
```bash
onedatastore list
```
*Kết quả mẫu:*
```
  ID NAME                SIZE AVAIL CLUSTERS  IMAGES TYPE DS      TM      STAT
   0 system                 - -     0              0 sys  -       ssh     on
   1 default            82.5G 88%   0              0 img  fs      ssh     on
   2 files              82.5G 88%   0              0 fil  fs      ssh     on
```

Kiểm tra danh sách Virtual Networks:
```bash
onevnet list
```
*Kết quả mẫu:*
```
  ID USER     GROUP    NAME            CLUSTERS   BRIDGE   STATE    LEASES
   0 oneadmin oneadmin lan_net         0          br0      rdy           0
```

---

### 5.3. Đăng nhập Giao diện Web FireEdge Sunstone

Mở trình duyệt trên máy tính cùng mạng LAN:
- **Địa chỉ truy cập**:
  - Qua HTTPS (Nginx SSL): `https://192.168.250.3` *(Khuyến nghị)*
  - Qua cổng trực tiếp của FireEdge: `http://192.168.250.3:2616`
- **Thông tin đăng nhập**:
  - **Username**: `oneadmin`
  - **Password**: Mật khẩu bạn đã thiết lập tại biến `one_pass` trong file `inventory/sno.yml` (mặc định trong hướng dẫn là `OpenNebula@2026`).

> [!NOTE]
> Bạn cũng có thể tra cứu token/mật khẩu gốc bất cứ lúc nào trên Mini PC bằng lệnh:
> ```bash
> sudo cat /var/lib/one/.one/one_auth
> ```

> [!TIP]
> **Xử lý cảnh báo chứng chỉ SSL tự ký (NET::ERR_CERT_AUTHORITY_INVALID):**
> 1. **Cách nhanh (Chrome)**: Khi gặp trang cảnh báo đỏ, click chuột vào nền trang và gõ thẳng phím `thisisunsafe` để truy cập ngay.
> 2. **Cách triệt để (macOS Keychain)**: Tải file CA về máy Mac:
>    ```bash
>    sudo cp /etc/one/fireedge-pki/crt/ca.crt /tmp/opennebula-ca.crt && sudo chmod 644 /tmp/opennebula-ca.crt
>    scp mini@192.168.250.3:/tmp/opennebula-ca.crt ~/Downloads/
>    sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/Downloads/opennebula-ca.crt
>    ```
>    Khởi động lại trình duyệt (`Cmd + Q`) để có ổ khóa xanh an toàn.

> [!IMPORTANT]
> **Thời hạn và Hướng dẫn gia hạn (Renew) chứng chỉ SSL:**
> * **Thời hạn**:
>   * Root CA (`OpenNebula Lab Root CA`): Hạn dùng **10 năm** (đến năm 2036). Do đó bạn chỉ cần cài đặt và tin cậy trên máy Mac **một lần duy nhất**.
>   * Server Certificate: Hạn dùng **1 năm (365 ngày)** để tuân thủ quy chuẩn bảo mật bắt buộc của Apple macOS/iOS và Google Chrome (tối đa 398 ngày).
> * **Khi chứng chỉ hết hạn (sau 1 năm)**:
>   * Các dịch vụ OpenNebula và toàn bộ máy ảo VM, Kubernetes OneKS bên trong **hoàn toàn không bị gián đoạn hay ảnh hưởng**.
>   * Trình duyệt web khi truy cập sẽ hiển thị cảnh báo `NET::ERR_CERT_DATE_INVALID`.
> * **Cách gia hạn lại thêm 1 năm (chỉ mất 10 giây)**:
>   * **Cách 1 (Qua Ansible - Khuyên dùng)**:
>     1. Trên Mini PC, xóa cert server cũ (vẫn giữ nguyên Root CA):
>        ```bash
>        sudo rm -f /etc/one/fireedge-pki/crt/server.crt /etc/one/fireedge-pki/csr/server.csr /etc/one/fireedge-pki/crt/certchain.crt
>        ```
>     2. Trên máy Mac, chạy lại playbook với tag `gui`:
>        ```bash
>        .venv/bin/ansible-playbook -i inventory/sno.yml playbooks/site.yml --tags gui
>        ```
>        *Ansible sẽ tự động lấy Root CA cũ ký ra một server certificate mới có hạn thêm 365 ngày. Do Root CA trên Mac không đổi, trình duyệt trên Mac sẽ lập tức xanh lại mà không cần import lại gì cả.*
>   * **Cách 2 (Bằng 1 lệnh OpenSSL trực tiếp trên Mini PC)**:
>     ```bash
>     sudo openssl x509 -req -in /etc/one/fireedge-pki/csr/server.csr \
>       -CA /etc/one/fireedge-pki/crt/ca.crt \
>       -CAkey /etc/one/fireedge-pki/key/ca.key \
>       -CAcreateserial -out /etc/one/fireedge-pki/crt/server.crt \
>       -days 365 -sha256 \
>       -extfile <(echo -e "subjectAltName=IP:192.168.250.3\nextendedKeyUsage=serverAuth,clientAuth\nbasicConstraints=CA:FALSE")
>     sudo cat /etc/one/fireedge-pki/crt/server.crt /etc/one/fireedge-pki/crt/ca.crt | sudo tee /etc/one/fireedge-pki/crt/certchain.crt > /dev/null
>     sudo systemctl reload nginx
>     ```

---

## 6. Hướng Dẫn Khởi Tạo Virtual Machine Đầu Tiên

Sau khi đăng nhập vào FireEdge Sunstone:

### Bước 6.1: Tải Appliance OS từ Public Marketplace
1. Vào mục **Storage** $\rightarrow$ **Apps** (hoặc **Marketplaces** $\rightarrow$ **OpenNebula Public**).
2. Tìm kiếm image OS bạn muốn sử dụng, ví dụ:
   - `Ubuntu 24.04 - KVM`
   - `Alpine Linux 3.20 - KVM` (siêu nhẹ, tải trong 10 giây).
3. Nhấp chọn Appliance $\rightarrow$ Bấm nút **Download** (biểu tượng đám mây tải xuống).
4. Chọn:
   - **Datastore**: `1: default` (Image Datastore).
   - Đặt tên cho Image và Template $\rightarrow$ Bấm **Download**.
5. Vào **Storage** $\rightarrow$ **Images**, đợi trạng thái của Image chuyển từ `LOCKED` sang `READY`.

---

### Bước 6.2: Khởi chạy (Instantiate) Virtual Machine
1. Vào mục **Templates** $\rightarrow$ **VM Templates**.
2. Chọn Template vừa tải về (ví dụ `Ubuntu 24.04`).
3. Bấm **Instantiate** (biểu tượng Play / Chạy):
   - **Name**: Đặt tên cho máy ảo (ví dụ `test-vm-01`).
   - **Capacity**: Chọn số CPU, vCPU (ví dụ 2 vCPU) và dung lượng RAM (ví dụ 2048 MB).
   - **Network**: Chọn Network `lan_net` (máy ảo sẽ tự động được cấp 1 IP trong dải `192.168.250.200 - 229`).
   - **User Input / Credentials**: Điền mật khẩu `root` cho máy ảo hoặc dán SSH Public Key của bạn.
4. Bấm **Instantiate**.

---

### Bước 6.3: Quản lý và truy cập Console của VM
1. Vào mục **Instances** $\rightarrow$ **VMs**.
2. Khi trạng thái VM chuyển sang `RUNNING`, nhấp vào VM.
3. Bấm vào biểu tượng **VNC / SPICE Console** ở góc trên bên phải để mở màn hình ảo tương tác trực tiếp với máy ảo.
4. Hoặc SSH trực tiếp từ máy Mac vào IP mà OpenNebula đã cấp cho VM:
   ```bash
   ssh root@192.168.250.200
   ```

---

### Bước 6.4: Quản trị Kubernetes Cluster (OneKS)
OpenNebula 7.4 tích hợp sẵn dịch vụ **OneKS (OpenNebula Kubernetes Service)** cho phép khởi tạo cụm Kubernetes (RKE2) tự động:
1. Vào mục **Kubernetes** $\rightarrow$ **K8S Clusters** trên thanh menu bên trái của FireEdge Sunstone.
2. Bấm nút **+ Create** để mở trình khởi tạo cụm Kubernetes:
   - **General**: Đặt tên cụm k8s (ví dụ `my-k8s-cluster`).
   - **Family**: Chọn `General Purpose` (workload thông thường).
   - **Flavour**: Chọn `Single-Node Control Plane` (phù hợp với SNO) hoặc cấu hình tài nguyên CPU, RAM, Disk cho node.
   - **Networks**:
     - **Public Network**: Chọn mạng `lan_net` (ID: `0`, dải `192.168.250.200 - 229`) để Virtual Router / Ingress nhận IP Public và mở cổng ra ngoài.
     - **Private Network**: Chọn mạng `oneks_priv` (ID: `1`, dải `172.20.0.10 - 249`) để các node Kubernetes Control Plane và Worker giao tiếp nội bộ tốc độ cao và bảo mật.
3. Bấm **Create** $\rightarrow$ OneKS sẽ tự động tải appliance, thiết lập router ảo, cài đặt RKE2 Kubernetes và cung cấp file `kubeconfig` trực tiếp trên web console.

---

### Bước 6.5: Định tuyến từ macOS vào mạng Private (`172.20.0.0/24`)

Máy ảo Kubernetes (Control Plane / Worker) hoặc máy ảo nội bộ được cấp IP thuộc dải `172.20.0.0/24` (thông qua bridge `br-priv` trên Mini PC). Do dải này không nằm cùng mạng LAN vật lý với máy Mac, bạn cần trỏ static route trên macOS qua Mini PC (`192.168.250.3`) để có thể ping, SSH hoặc gọi API trực tiếp vào máy ảo.

#### 1. Thêm static route tạm thời (có hiệu lực ngay lập tức):
```bash
sudo route -n add -net 172.20.0.0/24 192.168.250.3
```

#### 2. Kiểm tra kết nối:
- Ping kiểm tra gateway của bridge `br-priv` trên Mini PC (luôn phản hồi):
  ```bash
  ping -c 2 172.20.0.1
  ```
- Kiểm tra bảng định tuyến trên macOS:
  ```bash
  netstat -nr -f inet | grep 172.20.0
  ```
- Khi máy ảo trong mạng private đã bật (ví dụ có IP `172.20.0.13`), bạn có thể ping và SSH thẳng từ Terminal máy Mac:
  ```bash
  ssh root@172.20.0.13
  ```

#### 3. Xóa static route (khi không sử dụng):
```bash
sudo route -n delete -net 172.20.0.0/24 192.168.250.3
```

#### 4. Cấu hình tự động lưu vĩnh viễn trên macOS (Persistent across reboots):
Lệnh `route add` trên macOS sẽ mất hiệu lực sau khi restart máy. Để route tự động kích hoạt mỗi khi khởi động macOS:

1. Tạo file LaunchDaemon:
   ```bash
   sudo nano /Library/LaunchDaemons/local.route.oneks.plist
   ```
2. Dán nội dung sau:
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
       <key>Label</key>
       <string>local.route.oneks</string>
       <key>ProgramArguments</key>
       <array>
           <string>/sbin/route</string>
           <string>-n</string>
           <string>add</string>
           <string>-net</string>
           <string>172.20.0.0/24</string>
           <string>192.168.250.3</string>
       </array>
       <key>RunAtLoad</key>
       <true/>
       <key>StandardErrorPath</key>
       <string>/tmp/local.route.oneks.err</string>
       <key>StandardOutPath</key>
       <string>/tmp/local.route.oneks.out</string>
   </dict>
   </plist>
   ```
3. Phân quyền và kích hoạt service:
   ```bash
   sudo chown root:wheel /Library/LaunchDaemons/local.route.oneks.plist
   sudo chmod 644 /Library/LaunchDaemons/local.route.oneks.plist
   sudo launchctl load /Library/LaunchDaemons/local.route.oneks.plist
   ```

---

## 7. Xử Lý Sự Cố & Câu Hỏi Thường Gặp (Troubleshooting)

### 7.1. Lỗi `sudo: interactive authentication is required`
- **Nguyên nhân**: User `mini` chưa được bật quyền `NOPASSWD` trong `/etc/sudoers.d/mini`.
- **Khắc phục**:
  1. Đăng nhập vào Mini PC: `ssh mini-ubuntu`.
  2. Chạy: `echo "mini ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/mini && sudo chmod 0440 /etc/sudoers.d/mini`.
  3. Hoặc chạy Ansible kèm tham số `-K` để nhập mật khẩu khi chạy:
     ```bash
     ansible-playbook -i inventory/sno.yml playbooks/main.yml -K
     ```

---

### 7.2. Node ở trạng thái `ERR` hoặc không kết nối được KVM
- **Nguyên nhân**: Thường do SSH key giữa user `oneadmin` và localhost chưa được nhận diện, hoặc libvirt chưa chạy.
- **Khắc phục**:
  1. Đăng nhập vào Mini PC:
     ```bash
     ssh mini-ubuntu
     sudo su - oneadmin
     ```
  2. Thử SSH vào chính nó:
     ```bash
     ssh mini-ubuntu
     # Hoặc:
     ssh 192.168.250.3
     ```
     Nếu có câu hỏi `Are you sure you want to continue connecting (yes/no)?`, gõ `yes`.
  3. Buộc OpenNebula đồng bộ lại driver và probe lại node:
     ```bash
     onehost sync -f
     onehost forceupdate 0
     ```
  4. Kiểm tra lại `onehost list` xem trạng thái đã chuyển sang `on` hay chưa.

---

### 7.3. Cách xem log khi gặp lỗi dịch vụ
Các file log chính của OpenNebula nằm tại thư mục `/var/log/one/`:
- Log của core daemon:
  ```bash
  sudo tail -f /var/log/one/oned.log
  ```
- Log của FireEdge Sunstone GUI:
  ```bash
  sudo tail -f /var/log/one/fireedge.log
  sudo journalctl -u opennebula-fireedge -f
  ```
- Log của scheduler:
  ```bash
  sudo tail -f /var/log/one/sched.log
  ```

---

### 7.4. Cứu hộ mạng nếu Netplan cấu hình sai bridge `br0`
Nếu quá trình chuyển IP từ `eno1` sang `br0` gặp sự cố khiến máy mất kết nối:
1. Kết nối màn hình + bàn phím trực tiếp vào Mini PC (hoặc qua cổng console).
2. Kiểm tra cấu hình Netplan:
   ```bash
   cat /etc/netplan/*.yaml
   ```
3. Khôi phục cấu hình IP trực tiếp trên `eno1` và chạy:
   ```bash
   sudo netplan apply
   ```

---

### 7.5. Lỗi phụ thuộc `nodejs` khi cài đặt FireEdge (`opennebula-fireedge Depends nodejs <= 21`)
- **Nguyên nhân**: Trên Ubuntu 26.04 (hoặc một số bản Ubuntu mới), kho universe của Ubuntu cung cấp `nodejs 22.x`, trong khi `opennebula-fireedge` yêu cầu `nodejs <= 21`. OpenNebula đã cung cấp sẵn `nodejs 20.20.x` trong repo của mình, nhưng APT mặc định ưu tiên phiên bản cao hơn (22 > 20).
- **Khắc phục**: Pin phiên bản Node.js 20 cho APT bằng cách tạo file `/etc/apt/preferences.d/opennebula-nodejs.pref`:
  ```bash
  echo -e 'Package: nodejs\nPin: version 20.*\nPin-Priority: 1001' | sudo tee /etc/apt/preferences.d/opennebula-nodejs.pref
  ```
  Sau đó chạy lại playbook `ansible-playbook -i inventory/sno.yml playbooks/site.yml`.

---

### 7.6. Lỗi `Cannot connect to OneKS server, please verify that service is running`
- **Nguyên nhân**:
  1. Gói `opennebula-ks` chưa được cài đặt hoặc service `opennebula-ks.service` chưa chạy trên cổng `10780`.
  2. OneKS phụ thuộc vào TProxy (Transparent Proxy) để các node k8s giao tiếp với OpenNebula core và OneGate. Nếu thiếu cấu hình port `5030` và `2633` trong `/var/lib/one/remotes/etc/vnm/OpenNebulaNetwork.conf`, OneKS daemon sẽ từ chối khởi động.
- **Khắc phục**:
  1. Cấu hình biến `gate_tproxy` trong file `inventory/sno.yml`:
     ```yaml
     gate_tproxy:
       - :service_port: 5030
         :remote_addr: 192.168.250.3
         :remote_port: 5030
       - :service_port: 2633
         :remote_addr: 192.168.250.3
         :remote_port: 2633
     ```
  2. Bật tính năng `ks: true` trong mục `features:`.
  3. Chạy lại playbook triển khai:
     ```bash
     ansible-playbook -i inventory/sno.yml playbooks/site.yml
     ```
  4. Kiểm tra OneKS đã hoạt động:
     ```bash
     sudo systemctl status opennebula-ks
     sudo su - oneadmin -c "oneks list clusters"
     ```

---

### 7.7. Lỗi OneKS `State changed from PROVISIONING to PROVISIONING_FAILURE`
- **Hiện tượng**: Khi tạo cụm K8s trên Sunstone, trạng thái chuyển từ `PROVISIONING` sang `PROVISIONING_FAILURE`, xem log hiển thị `Dependency ClusterRouter failed: EVENT API one.vrouter.allocate subscriber timed out after 500 seconds`.
- **Nguyên nhân**:
  1. Trong quá trình tạo cụm, OneKS tạo ra một máy ảo tạm gọi là **Seed VM** chạy KinD (Kubernetes in Docker) cùng Cluster API Provider for OpenNebula (CAPONE).
  2. CAPONE cần kết nối ngược lại OpenNebula XML-RPC endpoint qua biến `:one_xmlrpc_tproxy:` trong `/etc/one/oneks-server.conf`.
  3. Giá trị mặc định là `http://169.254.16.9:2633/RPC2`. Khi Seed VM gửi gói tin đến IP link-local `169.254.16.9`, do thiếu route link-local nên hạt nhân định tuyến qua Default Gateway LAN (`192.168.250.1`), dẫn đến bị router LAN hủy gói tin (`i/o timeout`) và không thể tự động tạo Cluster Router (`vr-controlplane`).
- **Khắc phục**:
  1. Cấu hình `:one_xmlrpc_tproxy:` trỏ trực tiếp về IP Frontend có thể định tuyến được:
     ```yaml
     # Trong /etc/one/oneks-server.conf
     :one_xmlrpc_tproxy: http://192.168.250.3:2633/RPC2
     ```
  2. Khởi động lại dịch vụ OneKS: `sudo systemctl restart opennebula-ks`.
  3. Với cấu hình này, Seed VM trên `lan_net` kết nối trực tiếp tầng L2, và các worker nodes trên `oneks_priv` kết nối thông qua gateway `172.20.0.1` của Mini PC. CAPONE sẽ kết nối tức thì và cấp phát Virtual Router thành công.

---

## 8. Tóm Tắt Quy Trình Bằng 1 Lệnh Duy Nhất

Sau khi đã hoàn tất phần `Prerequisites` (NOPASSWD sudo và galaxy collections):

```bash
# Kích hoạt virtualenv
source .venv/bin/activate

# Chạy toàn bộ tiến trình cài đặt OpenNebula 7.4 SNO
ansible-playbook -i inventory/sno.yml playbooks/main.yml
```

Chúc mừng! Bạn đã hoàn thành việc thiết lập Cloud ảo hóa OpenNebula 7.4 SNO mạnh mẽ ngay trên Mini PC của mình.
