# Ansible Infrastructure Deployment

Repo Ansible để triển khai hạ tầng Kubernetes + PostgreSQL + RabbitMQ.

## Kiến trúc
ı
- **Kubernetes**: 1 master, 2 worker nodes (sử dụng `kubeadm` + `containerd`)
- **PostgreSQL**: 1 master, 1 slave (streaming replication)
- **RabbitMQ**: 2 nodes cluster với HA policy

## Cấu trúc thư mục

```
.
├── ansible.cfg              # Cấu hình Ansible
├── inventory/
│   └── hosts.ini           # Inventory file
├── group_vars/             # Biến cho từng group
│   ├── k8s_cluster.yml
│   ├── postgres_cluster.yml
│   └── rabbitmq.yml
├── roles/
│   ├── common/             # Role chung cho tất cả node
│   ├── kubernetes/
│   │   ├── common/         # Cài đặt chung cho k8s
│   │   ├── master/         # Cấu hình master node
│   │   └── worker/         # Cấu hình worker nodes
│   ├── postgres/
│   │   ├── master/         # PostgreSQL master
│   │   └── slave/          # PostgreSQL slave
│   └── rabbitmq/
│       └── node/           # RabbitMQ node
├── site.yml                # Playbook tổng
├── k8s.yml                 # Playbook chỉ cho Kubernetes
├── postgres.yml            # Playbook chỉ cho PostgreSQL
└── rabbitmq.yml            # Playbook chỉ cho RabbitMQ
```

## Yêu cầu

- Ansible >= 2.9
- Python >= 3.6 trên control node
- SSH key-based authentication từ Ansible control node tới tất cả target nodes
- Ubuntu 22.04 (hoặc tương thích) trên tất cả target nodes
- Quyền sudo/root trên tất cả target nodes

## Cấu hình

### 1. Cập nhật Inventory

Chỉnh sửa `inventory/hosts.ini` với IP/hostname thực tế của các server:

```ini
[k8s_master]
k8s-master ansible_host=10.0.0.10

[k8s_workers]
k8s-worker1 ansible_host=10.0.0.11
k8s-worker2 ansible_host=10.0.0.12

[pg_master]
pg-master ansible_host=10.0.0.20

[pg_slave]
pg-slave ansible_host=10.0.0.21

[rabbitmq]
rabbit1 ansible_host=10.0.0.30
rabbit2 ansible_host=10.0.0.31
```

### 2. Cấu hình biến

Chỉnh sửa các file trong `group_vars/` để phù hợp với môi trường:

- `group_vars/k8s_cluster.yml`: Version Kubernetes, CIDR, network plugin
- `group_vars/postgres_cluster.yml`: Version PostgreSQL, passwords, backup config
- `group_vars/rabbitmq.yml`: Version RabbitMQ, passwords, cluster config

**QUAN TRỌNG**: Thay đổi tất cả passwords mặc định!

### 3. Thiết lập SSH key

Đảm bảo Ansible control node có thể SSH vào tất cả target nodes không cần password:

```bash
ssh-copy-id user@target-node
```

## Sử dụng

### Triển khai toàn bộ hạ tầng

```bash
ansible-playbook site.yml
```

### Triển khai từng phần

```bash
# Chỉ Kubernetes
ansible-playbook k8s.yml

# Chỉ PostgreSQL
ansible-playbook postgres.yml

# Chỉ RabbitMQ
ansible-playbook rabbitmq.yml
```

### Kiểm tra kết nối

```bash
ansible all -m ping
```

## Quy trình triển khai

Playbook `site.yml` sẽ chạy theo thứ tự:

1. **Common baseline**: Cấu hình chung cho tất cả node (timezone, locale, packages, swap, hosts)
2. **Kubernetes common**: Cài đặt containerd, kubeadm, kubelet, kubectl
3. **Kubernetes master**: Khởi tạo cluster, cài network plugin (Calico)
4. **Kubernetes workers**: Join workers vào cluster
5. **PostgreSQL master**: Cài đặt và cấu hình master với replication
6. **PostgreSQL slave**: Thiết lập streaming replication từ master
7. **RabbitMQ**: Cài đặt và cấu hình cluster (nếu có nhiều node)

## Verify

### Kubernetes

```bash
# Trên master node
kubectl get nodes
kubectl get pods -n kube-system
```

### PostgreSQL

```bash
# Trên master node
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"

# Trên slave node
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

### RabbitMQ

```bash
# Kiểm tra cluster status
sudo rabbitmqctl cluster_status

# Kiểm tra management UI (port 15672)
curl http://localhost:15672
```

## Rollback

### Kubernetes

```bash
# Reset worker node
kubeadm reset

# Reset master node (cẩn thận!)
kubeadm reset
```

### PostgreSQL

```bash
# Dừng replication trên slave
sudo systemctl stop postgresql
# Xóa data và cấu hình lại nếu cần
```

### RabbitMQ

```bash
# Remove node khỏi cluster
sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl start_app
```

## Lưu ý

- Đảm bảo các port cần thiết đã được mở trên firewall
- Kubernetes: 6443, 10250, 10259, 10257, 2379-2380, 30000-32767
- PostgreSQL: 5432
- RabbitMQ: 5672, 15672, 25672, 4369
- SSH: 22

- Backup passwords và cấu hình quan trọng trước khi triển khai
- Test trên môi trường dev/staging trước khi deploy production

## Troubleshooting

### Kubernetes worker không join được

- Kiểm tra join command trên master: `cat /tmp/kubeadm_join_cmd.sh`
- Kiểm tra network connectivity giữa master và worker
- Kiểm tra firewall rules

### PostgreSQL replication không hoạt động

- Kiểm tra `pg_hba.conf` có cho phép replication user
- Kiểm tra `postgresql.conf` có `wal_level = replica`
- Kiểm tra network connectivity giữa master và slave

### RabbitMQ cluster không join

- Kiểm tra `erlang.cookie` giống nhau trên tất cả nodes
- Kiểm tra hostname resolution
- Kiểm tra ports 4369, 25672 đã mở

## Tác giả

Triển khai theo kế hoạch: K8s-PG-Rabbit-Ansible-Infra
