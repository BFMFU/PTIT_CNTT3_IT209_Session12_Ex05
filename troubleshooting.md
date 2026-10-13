# BÁO CÁO CHẨN ĐOÁN VÀ KHẮC PHỤC SỰ CỐ PIPELINE CI/CD (BÀI 5)

## 1. Giới thiệu tổng quan
Trong quá trình triển khai luồng tự động hóa CI/CD với GitHub Actions cho dự án Spring Boot, tệp kịch bản `.github/workflows/broken-pipeline.yml` ban đầu gặp phải 3 sự cố đứt gãy nghiêm trọng khiến pipeline bị dừng giữa chừng (trạng thái Failed). 

Tài liệu này ghi lại toàn bộ quá trình phân tích log, xác định nguyên nhân gốc rễ (Root Cause Analysis) và giải pháp kỹ thuật đã áp dụng để khôi phục luồng pipeline đạt trạng thái tích xanh (Success).

---

## 2. Chi tiết chẩn đoán và khắc phục 3 sự cố

### 2.1. Lỗi 1: Permission Denied (Gradle Wrapper thiếu quyền thực thi)

#### a. Triệu chứng lỗi (Symptoms & Log)
Khi runner thực thi bước kiểm thử và đóng gói ứng dụng:
```bash
Run ./gradlew test bootJar
/home/runner/work/_temp/xxxx-xxxx.sh: line 1: ./gradlew: Permission denied
Error: Process completed with exit code 126.
```

#### b. Phân tích nguyên nhân gốc rễ (Root Cause)
- Khi mã nguồn được commit và push từ môi trường Windows lên Git, thuộc tính quyền thực thi (POSIX execute bit `+x`) của tệp `gradlew` không được bảo lưu (chỉ có quyền đọc/ghi `100644`).
- Khi GitHub Actions runner khởi chạy trên hệ điều hành Linux (`ubuntu-latest`) và tải mã nguồn về qua `actions/checkout`, tệp `./gradlew` thiếu quyền `execute`. Do đó, bash shell từ chối thực thi với mã lỗi `126 (Permission Denied)`.

#### c. Giải pháp khắc phục (Resolution)
- **Trong workflow**: Bổ sung bước cấp quyền thực thi cho Gradle Wrapper trước khi gọi lệnh `./gradlew`:
  ```yaml
  - name: Grant execute permission for gradlew
    run: chmod +x gradlew
  ```
- **Trong Git Repository**: Cập nhật file mode trên Git index để bảo lưu quyền `executable (100755)`:
  ```bash
  git update-index --chmod=+x gradlew
  ```

---

### 2.2. Lỗi 2: Authentication Failed (Lỗi đăng nhập Docker Registry do sai tên Secret)

#### a. Triệu chứng lỗi (Symptoms & Log)
Khi chạy bước `docker/login-action@v3` để đăng nhập vào GitHub Container Registry (`ghcr.io`):
```text
Error: Username and password are required
Error: Cannot perform an interactive login from a non TTY device
Error: Process completed with exit code 1.
```
hoặc khi đẩy image lên registry bị lỗi `unauthorized: authentication required`.

#### b. Phân tích nguyên nhân gốc rễ (Root Cause)
- Kịch bản ban đầu khai báo:
  ```yaml
  password: ${{ secrets.REGISTRY_TOKN }}
  ```
- Tên secret bị gõ sai chính tả (`REGISTRY_TOKN` thiếu chữ `E`), hoặc repository không hề khai báo secret mang tên này.
- Khi truy cập biến secret không tồn tại, GitHub Actions trả về chuỗi rỗng `""`. Bước login không nhận được mật khẩu hợp lệ nên thất bại ngay lập tức.
- Đối với GHCR trên GitHub Actions, GitHub cung cấp sẵn biến tự sinh `${{ secrets.GITHUB_TOKEN }}` kết hợp với permission `packages: write` mà không cần cấu hình secret thủ công.

#### c. Giải pháp khắc phục (Resolution)
- Sửa lại tham số `password` trong action đăng nhập sang biến `${{ secrets.GITHUB_TOKEN }}` hợp lệ:
  ```yaml
  - name: Login to GitHub Container Registry (GHCR)
    uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
  ```
- Đảm bảo workflow có quyền `packages: write`:
  ```yaml
  permissions:
    contents: read
    packages: write
  ```

---

### 2.3. Lỗi 3: Build Context Error (Không tìm thấy Dockerfile do sai thư mục làm việc)

#### a. Triệu chứng lỗi (Symptoms & Log)
Khi chạy bước build và push Docker image bằng `docker/build-push-action@v5`:
```text
ERROR: failed to solve: failed to read dockerfile: open /var/lib/docker/.../docker/Dockerfile: no such file or directory
# hoặc
unable to prepare context: path "./docker" not found
Error: buildx failed with: exit code 1
```

#### b. Phân tích nguyên nhân gốc rễ (Root Cause)
- Kịch bản ban đầu trỏ cấu hình build context vào thư mục không tồn tại:
  ```yaml
  context: ./docker
  file: ./docker/Dockerfile
  ```
- Trong khi đó, tệp `Dockerfile` của dự án nằm ngay tại thư mục gốc của repository (`./Dockerfile`).
- Do chỉ định sai ngữ cảnh build context, BuildKit/Docker daemon không thể tìm thấy tệp `Dockerfile` để phân tích cú pháp và xây dựng image.
- Ngoài ra, định danh image tag khi đẩy lên GitHub Container Registry (`ghcr.io`) yêu cầu toàn bộ ký tự tên kho lưu trữ phải là chữ thường (lowercase); nếu giữ nguyên tên repository chứa chữ hoa sẽ bị lỗi cú pháp `invalid reference format: repository name must be lowercase`.

#### c. Giải pháp khắc phục (Resolution)
- Sửa `context` về thư mục gốc `.` và `file` về `./Dockerfile`.
- Thêm bước tiền xử lý chuẩn hóa tên image về chữ thường (lowercase):
  ```yaml
  - name: Prepare lowercase image tag
    id: meta
    run: |
      REPO_LOWER=$(echo "${{ github.repository }}" | tr '[:upper:]' '[:lower:]')
      echo "tag=ghcr.io/${REPO_LOWER}:latest" >> $GITHUB_OUTPUT

  - name: Build and push Docker image
    uses: docker/build-push-action@v5
    with:
      context: .
      file: ./Dockerfile
      push: true
      tags: ${{ steps.meta.outputs.tag }}
  ```

---

## 3. Bảng so sánh cấu hình Trước và Sau khi sửa

| Thành phần | Trước khi sửa (Lỗi) | Sau khi sửa (Đã khắc phục) | Mục đích / Kết quả |
| :--- | :--- | :--- | :--- |
| **Quyền Gradle Wrapper** | Không có bước cấp quyền | `- name: Grant execute permission for gradlew`<br>`run: chmod +x gradlew` | Cấp quyền thực thi `+x`, tránh lỗi `Permission denied (126)` |
| **Lệnh build ứng dụng** | `./gradlew test` (chỉ chạy test, không đóng gói JAR) | `./gradlew test bootJar` | Kiểm thử và tạo file JAR trong `build/libs/` cho Docker build |
| **Secret đăng nhập Registry** | `password: ${{ secrets.REGISTRY_TOKN }}` (sai chính tả) | `password: ${{ secrets.GITHUB_TOKEN }}` | Xác thực thành công vào GHCR với quyền `packages: write` |
| **Build Context Docker** | `context: ./docker`<br>`file: ./docker/Dockerfile` | `context: .`<br>`file: ./Dockerfile` | Trỏ chính xác vị trí tệp `Dockerfile` tại thư mục gốc |
| **Định danh Image Tag** | `tags: ghcr.io/${{ github.repository }}:latest` (dính chữ in hoa) | `tags: ${{ steps.meta.outputs.tag }}` (đã chuyển chữ thường) | Tuân thủ chuẩn OCI/Docker registry naming convention |

---

## 4. Tệp kịch bản hoàn chỉnh sau khi khắc phục (`.github/workflows/broken-pipeline.yml`)

```yaml
name: Broken Pipeline - Ex5

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      # Khắc phục Lỗi 1: Cấp quyền thực thi cho Gradle Wrapper
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Gradle tests and package
        run: ./gradlew test bootJar

      # Khắc phục Lỗi 2: Sửa tên Secret từ REGISTRY_TOKN thành GITHUB_TOKEN hợp lệ
      - name: Login to GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Chuẩn hóa tên image về chữ thường theo chuẩn OCI/Docker
      - name: Prepare lowercase image tag
        id: meta
        run: |
          REPO_LOWER=$(echo "${{ github.repository }}" | tr '[:upper:]' '[:lower:]')
          echo "tag=ghcr.io/${REPO_LOWER}:latest" >> $GITHUB_OUTPUT

      # Khắc phục Lỗi 3: Chỉ định đúng build context (thư mục gốc .) và đường dẫn Dockerfile (./Dockerfile)
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tag }}
```

---

## 5. Kết quả kiểm thử và xác nhận
1. **Lần chạy ban đầu (Commit lỗi)**:
   - Pipeline bị gãy tại step `Run Gradle tests and package` do lỗi `Permission denied`.
   - Các bước đăng nhập Docker và Build Docker image bị bỏ qua (Skipped).
2. **Lần chạy sau khi sửa (Commit fix)**:
   - Toàn bộ các bước: Checkout -> Setup Java -> Grant permission -> Run Gradle tests & package -> Login to GHCR -> Build and push Docker image đều thực thi thành công.
   - Trạng thái Workflow Run trên tab Actions chuyển sang **Tích xanh (Success)**.
   - Package Docker image được xuất bản thành công lên GitHub Container Registry (`ghcr.io`).
