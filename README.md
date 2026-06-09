================================================================================
          CẨM NANG HƯỚNG DẪN KUBERNETES (K8S) TOÀN DIỆN CHO THỰC HÀNH
================================================================================

1. MÔ HÌNH TƯ DUY KHÁI NIỆM (MENTALLY MAPPING)
--------------------------------------------------------------------------------
- Để dễ hình dung về cách thức vận hành, ta sử dụng các hình ảnh ẩn dụ sau:
  + kubectl: Chiếc điều khiển từ xa (Remote Control) dùng để ra lệnh.
  + minikube / kind: Chiếc Tivi (Kubernetes Cluster) nhận và thực thi lệnh.
  + Rancher: Màn hình Dashboard trực quan quản lý Cluster bằng giao diện đồ họa.

- Luồng quản lý kiến trúc từ trên xuống dưới (Architecture Flow):
  Control Plane (Master Node) -> Worker Node -> Pod -> Container

  + Control Plane: Đầu não điều khiển, lập lịch và quản lý trạng thái Cluster.
  + Worker Node: Máy chủ vật lý hoặc máy ảo chịu trách nhiệm chạy các ứng dụng.
  + Pod: Đơn vị triển khai nhỏ nhất, chứa một hoặc một nhóm Container chung mạng.
  + Container: Môi trường chạy ứng dụng cô lập (Docker, containerd).


2. HƯỚNG DẪN CÀI ĐẶT MÔI TRƯỜNG TRÊN UBUNTU/LINUX
--------------------------------------------------------------------------------
### Bước 1: Cài đặt Kubectl CLI
sudo curl -LO "https://k8s.io(curl -L -s https://k8s.io)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

### Bước 2: Bật tính năng tự động gợi ý lệnh (Bash Auto-completion)
sudo apt-get update && sudo apt-get install bash-completion -y
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc

### Bước 3: Kiểm tra phiên bản cài đặt thành công
kubectl version --client

### Bước 4: Cài đặt và khởi chạy Minikube
curl -LO https://googleapis.com
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Khởi động Cluster sử dụng driver ảo hóa (Mặc định khuyến khích dùng Docker)
minikube start --driver=docker


3. TẤT TẦN TẬT CÁC LỆNH SỬ DỤNG PHỔ BIẾN (CHEATSHEET)
--------------------------------------------------------------------------------
### Kiểm tra trạng thái Cluster & Node
- minikube status                   : Kiểm tra trạng thái cụm minikube local.
- kubectl cluster-info              : Xem thông tin tổng quan của Cluster.
- kubectl get nodes                 : Liệt kê các Node hiện có.
- kubectl get nodes -o wide         : Xem chi tiết IP mạng, OS, Kernel của Node.

### Quản lý Vòng đời Pod
- kubectl get pods -A               : Liệt kê toàn bộ Pod ở tất cả các Namespace.
- kubectl get pods -w               : Theo dõi (watch) trạng thái Pod thời gian thực.
- kubectl run <ten_pod> --image=<url_image> : Tạo nhanh một Pod đơn lẻ từ Image.
- kubectl describe pod <ten_pod>    : Xem thông tin cấu hình, lỗi, sự kiện (Events).
- kubectl logs <ten_pod>            : In nhật ký hệ thống (logs) của Pod để debug.
- kubectl logs -f <ten_pod>         : Theo dõi luồng logs liên tục (live stream).
- kubectl exec -it <ten_pod> -- bash: Đi vào bên trong container của Pod để gõ lệnh.
- kubectl delete pod <ten_pod>      : Xóa một Pod cụ thể.

### Quản lý Deployment & Scale (Bản sao rộng)
- kubectl apply -f <ten_file>.yaml   : Khởi tạo/Cập nhật tài nguyên từ file Manifest.
- kubectl get deploy                : Xem danh sách và trạng thái các Deployment.
- kubectl get rs                    : Xem các ReplicaSet (Bộ quản lý số lượng bản sao).
- kubectl scale deployment/<ten> --replicas=<so_luong> : Tăng/giảm số Pod bằng lệnh.
- kubectl edit deployment/<ten>     : Chỉnh sửa trực tiếp file YAML đang chạy trên cụm.
- kubectl delete deploy <ten>       : Xóa một Deployment (sẽ xóa luôn các Pod của nó).


4. CHIẾN LƯỢC CẬP NHẬT ỨNG DỤNG (ROLLOUT & ROLLBACK)
--------------------------------------------------------------------------------
### Cơ chế triển khai (Deployment Strategies)
- RollingUpdate (Mặc định): Thay thế Pod cũ bằng Pod mới một cách cuốn chiếu (từng
  cái một). Đảm bảo hệ thống luôn hoạt động liên tục, không bị gián đoạn (Zero Downtime).
- Recreate: Xóa bỏ toàn bộ các Pod cũ trước, sau đó mới tạo đồng loạt các Pod mới.
  Cách này gây ra một khoảng thời gian chết (Downtime) nhưng tránh việc 2 phiên bản
  chạy song song.

### Hệ thống lệnh Rollout & Quay xe (Rollback)
- kubectl rollout status deployment/<ten>  : Theo dõi tiến độ cập nhật bản build mới.
- kubectl rollout history deployment/<ten> : Xem lịch sử các phiên bản chỉnh sửa cấu hình.
- kubectl rollout restart deployment/<ten> : Ép buộc khởi động lại toàn bộ Pod.
- kubectl rollout undo deployment/<ten>    : Quay về phiên bản cấu hình ngay trước đó.
- kubectl rollout undo deployment/<ten> --to-revision=<ID> : Quay về bản cụ thể theo ID.

### Lưu ý đặt tên lý do cập nhật (Change Cause)
Để danh sách lịch sử (history) hiển thị lý do thay đổi thay vì chữ `<none>`, sử dụng:
kubectl annotate deployment/<ten_deploy> kubernetes.io/change-cause="Nâng cấp lên v2"


5. MẠNG TRONG K8S: SERVICES VÀ INGRESS (NETWORKING)
--------------------------------------------------------------------------------
Ứng dụng chạy trong Pod có tính chất tạm thời, IP sẽ thay đổi khi Pod bị lỗi và khởi
động lại. Service ra đời để làm đầu mối giao tiếp cố định.

### Phân loại các dạng Service
1. ClusterIP (Mặc định): Cấp một IP nội bộ. Chỉ các thành phần trong cùng một Cluster
   mới gọi được nhau. Bên ngoài Internet hoàn toàn không thể truy cập.
2. NodePort: Mở một cổng ngẫu nhiên hoặc cố định trên toàn bộ các Node (trong dải
   30000-32767). Cho phép bên ngoài truy cập thông qua địa chỉ `<IP_MÁY_NODE>:<CỔNG_NODEPORT>`.
3. LoadBalancer: Tích hợp với Cloud Provider (AWS, GCP, Azure...) cấp một IP công khai
   (External IP) để định tuyến trực tiếp vào ứng dụng.

### Ingress (Bộ định tuyến nâng cao)
- Hoạt động như một Reverse Proxy (giống Nginx/HAProxy) nằm trước Service. 
- Giúp gom các Service lại và định tuyến dựa trên đường dẫn URL hoặc Domain (Ví dụ: 
  ://myapp.com chuyển hướng đến api-service, ://myapp.com đến web-service).

### Lệnh thực hành mạng:
- kubectl get svc                   : Xem danh sách các service hiện tại.
- kubectl expose deployment/<ten> --type=NodePort --port=<cong_internal> : Tạo nhanh svc.
- minikube service <ten_service> --url : Lấy URL truy cập trực tiếp từ máy local.


6. CƠ CHẾ QUẢN LÝ TÀI NGUYÊN (RESOURCE MANAGEMENT)
--------------------------------------------------------------------------------
Để tránh việc một Pod bị rò rỉ bộ nhớ gây ảnh hưởng đến toàn bộ Node, bắt buộc phải
thiết lập giới hạn tài nguyên tại phần `resources` trong Manifest.

- Requests (Mức tối thiểu): Lượng CPU/RAM mà Pod chắc chắn được cam kết cấp phát
  ngay khi vừa khởi tạo. K8s sẽ dùng thông số này để tính toán xem Node nào còn đủ
  chỗ để đặt Pod vào (gọi là Schedule).
- Limits (Mức tối đa): Lượng CPU/RAM cao nhất mà Pod được phép chạm ngưỡng sử dụng.
  + Nếu vượt quá Limit CPU   -> Ứng dụng bị bóp hiệu năng, chạy chậm đi (Throttled).
  + Nếu vượt quá Limit Memory-> Pod lập tức bị hệ thống giết (Lỗi OOMKilled).

* Đơn vị tính toán:
  + CPU: Tính theo mili-cores. Ví dụ: `500m` = 0.5 Core CPU.
  + Memory: Tính theo Byte. Ví dụ: `256Mi` (Mebibyte), `1Gi` (Gibibyte).


7. KHÔNG GIAN ẢO (NAMESPACES)
--------------------------------------------------------------------------------
- Khái niệm: Namespace (ns) là cơ chế phân chia một cụm Cluster vật lý thành các vùng
  không gian ảo độc lập về mặt logic (Ví dụ: `ns-dev`, `ns-staging`, `ns-production`).
- Tác dụng: Hỗ trợ cô lập tài nguyên, phân quyền bảo mật nhóm (RBAC) và kiểm soát
  hạn mức chi phí cho từng phòng ban.
- Giao tiếp xuyên Namespace: Các Pod ở hai Namespace khác nhau vẫn có thể kết nối với
  nhau qua DNS hệ thống theo cú pháp: `<ten-service>.<ten-namespace>.svc.cluster.local`.

### Lệnh thực hành Namespace:
- kubectl get ns                    : Xem danh sách tất cả Namespace trong cụm.
- kubectl create ns <ten_ns>        : Tạo một không gian ảo mới.
- kubectl get pods -n <ten_ns>      : Xem danh sách các Pod thuộc riêng Namespace đó.


8. LƯU TRỮ DỮ LIỆU BỀN VỮNG (STORAGE VOLUMES)
--------------------------------------------------------------------------------
Mặc định khi Pod bị xóa, mọi dữ liệu phát sinh bên trong container sẽ biến mất. Để lưu
dữ liệu vĩnh viễn (như Database, file upload), K8s dùng cơ chế:

- PersistentVolume (PV): Phần ổ đĩa cứng thực tế do Quản trị viên hệ thống (Admin)
  khởi tạo trước hoặc được cấp phát tự động từ Cloud.
- PersistentVolumeClaim (PVC): Lời yêu cầu thuê ổ đĩa của Lập trình viên (Developer).
  Pod sẽ gọi PVC, PVC sẽ tự tìm PV có dung lượng tương xứng để gắn vào Pod.


9. QUẢN LÝ CẤU HÌNH VÀ BẢO MẬT (CONFIGMAP & SECRET)
--------------------------------------------------------------------------------
Tách biệt mã nguồn và thông tin cấu hình môi trường theo chuẩn ứng dụng hiện đại:

- ConfigMap (CM): Dùng để lưu trữ các cấu hình dạng văn bản thuần túy không nhạy cảm
  (Ví dụ: URL của hệ thống khác, biến cấu hình ứng dụng, file cấu hình nginx.conf).
- Secret: Dùng để lưu trữ các thông tin bảo mật, nhạy cảm (Ví dụ: Mật khẩu Database,
  Token API, SSH Key). Dữ liệu lưu trong Secret mặc định được mã hóa dưới dạng chuỗi Base64.


================================================================================
10. MẪU FILE MANIFEST HOÀN CHỈNH (.YAML) ĐẦY ĐỦ TÍNH NĂNG
================================================================================
Dưới đây là một file YAML chuẩn mẫu tích hợp toàn bộ các kiến thức từ mục 1 đến mục 9,
bao gồm: Namespace, ConfigMap, Secret, Deployment với giới hạn tài nguyên và Service.

---
apiVersion: v1
kind: Namespace
metadata:
  name: huong-dan-k8s
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: huong-dan-k8s
data:
  APP_MODE: "production"
  DATABASE_HOST: "mysql-service"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: huong-dan-k8s
type: Opaque
data:
  # Chuỗi gốc là 'mysecretpassword' được mã hóa sang base64
