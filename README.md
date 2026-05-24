# 23520678_lab2.2

Dự án này cung cấp mã nguồn và quy trình chi tiết để tự động hóa triển khai hạ tầng mạng và máy chủ trên Amazon Web Services (AWS) thông qua **Infrastructure as Code (IaC)** với cấu trúc **Nested CloudFormation Stacks**. Toàn bộ quy trình được quản trị tự động thông qua luồng CI/CD của **AWS CodePipeline**.

---

## 1. Tổng Quan Kiến Trúc (Architecture Overview)

Hệ thống được thiết kế theo dạng module hóa, chia nhỏ các thành phần hạ tầng thành các file độc lập để dễ dàng quản lý và tái sử dụng:

* **`network.yaml` (Module Mạng):** Khởi tạo một Virtual Private Cloud (VPC) với dải IP `10.0.0.0/16`. Phân chia hệ thống mạng thành Public Subnet (có Internet Gateway) và Private Subnet (có NAT Gateway để cấp quyền truy cập ra ngoài an toàn).
* **`compute.yaml` (Module Máy chủ):** Khởi tạo Security Group mở port 22/80 và một máy chủ EC2 (`t2.micro`) nằm sâu bên trong Private Subnet.
* **`main.yaml` (Root Stack):** Đóng vai trò là file gốc, tự động gọi và truyền các tham số (parameters/outputs) giữa module mạng và module máy chủ.

**Luồng CI/CD (AWS CodePipeline):**
1.  **Source:** Tự động bắt sự kiện khi có thay đổi mã nguồn trên nhánh chính của kho lưu trữ GitHub (`khaipd18/23520678_lab2.2`).
2.  **Build:** Môi trường AWS CodeBuild thực thi `buildspec.yml`, sử dụng lệnh `aws cloudformation package` để đóng gói các file template con và đẩy lên Amazon S3. *(Lưu ý: Các bước kiểm tra tĩnh cfn-lint và taskcat được bỏ qua để tối ưu thời gian triển khai trong môi trường lab).*
3.  **Deploy:** AWS CloudFormation nhận file `packaged-main.yaml` và tiến hành khởi tạo hạ tầng thực tế.

---

## 2. Cấu Trúc Mã Nguồn

```text
.
├── .gitignore
├── .taskcat.yml                 # Cấu hình kiểm thử (đã tắt trong buildspec)
├── buildspec.yml                # Kịch bản thực thi cho AWS CodeBuild
├── README.md                    # Tài liệu hướng dẫn
└── templates/                   # Chứa các file CloudFormation
    ├── compute.yaml
    ├── main.yaml
    └── network.yaml
```

## 3. Hướng Dẫn Cài Đặt Môi Trường

Để Pipeline có thể chạy thành công, bạn cần thiết lập trước các tài nguyên nền tảng sau trên tài khoản AWS:

### 3.1. Chuẩn Bị Amazon S3 Bucket

CloudFormation cần một nơi để lưu trữ các file template con trước khi triển khai.

1. Truy cập dịch vụ Amazon S3.
2. Tạo một bucket với tên khớp chính xác với biến môi trường trong file `buildspec.yml`: `lab2-02-khaipd18`.
3. Chọn Region phù hợp (Khuyến nghị: `ap-southeast-1` - Singapore).

### 3.2. Thiết Lập Phân Quyền (IAM Roles)

* **Role cho CloudFormation (`Lab-CloudFormation-Role`):** Tạo một Role dành cho dịch vụ CloudFormation và gắn chính sách `AdministratorAccess` để cấp quyền tạo VPC và EC2.
* **Role cho CodePipeline:** Đảm bảo Service Role của CodePipeline cũng được gắn chính sách `AdministratorAccess` (hoặc ít nhất là quyền `cloudformation:DescribeStacks`) để luồng chạy không bị báo lỗi Access Denied ở bước Deploy.

### 3.3. Xử Lý Định Dạng File Tại Máy Cục Bộ

Nếu bạn thực hiện lập trình trực tiếp trên môi trường cục bộ (đặc biệt là qua WSL trên máy ThinkPad), hãy đảm bảo các file `.yaml` được lưu dưới chuẩn kết thúc dòng của Linux (LF) thay vì Windows (CRLF) để tránh lỗi cú pháp ảo.

```bash
# Chạy trong terminal WSL
sudo apt-get install dos2unix
dos2unix templates/*.yaml buildspec.yml
```

---

## 4. Hướng Dẫn Triển Khai (Chạy Mã Nguồn)

1. Đẩy toàn bộ mã nguồn lên nhánh chính (`main`) của GitHub.
2. Truy cập **AWS CodePipeline**, tạo một Pipeline mới:
   * **Source:** Kết nối với GitHub Repository của bạn.
   * **Build:** Chọn AWS CodeBuild, tạo một dự án Build mới sử dụng file `buildspec.yml` có sẵn trong source code.
   * **Deploy:** Chọn AWS CloudFormation, Action mode là `Create or update a stack`.
     * Trỏ file template vào Artifact từ bước Build (`packaged-main.yaml`).
     * Bắt buộc chọn 2 Capabilities: `CAPABILITY_IAM` và `CAPABILITY_AUTO_EXPAND`.
     * Gắn IAM Role là `Lab-CloudFormation-Role`.
3. Lưu cấu hình và nhấn **Release change** để kích hoạt luồng triển khai tự động.

---

## 5. Hướng Dẫn Kiểm Tra Kết Quả Triển Khai

Sau khi Pipeline hiển thị trạng thái **Succeeded** ở cả 3 bước, hãy tiến hành kiểm chứng hạ tầng:

**Kiểm tra AWS CloudFormation:**
* Truy cập dịch vụ CloudFormation, kiểm tra danh sách Stacks.
* Phải xuất hiện 1 Root Stack và 2 Nested Stacks (Mạng và Máy chủ) với trạng thái màu xanh lá `CREATE_COMPLETE`.

**Kiểm tra hạ tầng Mạng (VPC):**
* Truy cập dịch vụ VPC, xác nhận có 1 VPC tên `LabPipeline-VPC`.
* Kiểm tra phần Route Tables: Đảm bảo bảng định tuyến Public có trỏ ra Internet Gateway (`igw-***`), và bảng định tuyến Private có trỏ ra NAT Gateway (`nat-***`).

**Kiểm tra hạ tầng Máy Chủ (EC2):**
* Truy cập dịch vụ EC2 > Instances.
* Xác nhận có một máy chủ tên `LabPipeline-PrivateEC2` đang ở trạng thái **Running**.
* Kiểm tra tab Networking của máy chủ này để đảm bảo nó được gắn địa chỉ IP nội bộ (`10.0.x.x`) và **không có** địa chỉ Public IPv4 (do nằm trong Private Subnet).
