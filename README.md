# Bài 5: Chẩn đoán và khắc phục 3 sự cố đứt gãy Pipeline CI/CD thực tế

## 1. Mục tiêu bài thực hành
Phân tích log lỗi thực tế trên GitHub Actions và thực hiện sửa lỗi trực tiếp trên tệp kịch bản `.github/workflows/broken-pipeline.yml` để khôi phục luồng CI/CD đạt trạng thái **tích xanh (Success)**.

## 2. Các sự cố được chẩn đoán và khắc phục

| Sự cố | Mã lỗi / Triệu chứng | Nguyên nhân gốc rễ | Giải pháp khắc phục |
| :--- | :--- | :--- | :--- |
| **Lỗi 1 (Permission Denied)** | `line 1: ./gradlew: Permission denied` (Exit code 126) | Gradle Wrapper thiếu quyền thực thi POSIX (`+x`) trên runner Linux | Thêm step `chmod +x gradlew` và cập nhật Git filemode (`git update-index --chmod=+x gradlew`) |
| **Lỗi 2 (Authentication Failed)** | `Username and password are required` / `unauthorized` | Khai báo sai tên biến Secret (`REGISTRY_TOKN` thay vì `GITHUB_TOKEN`) | Đổi sang `${{ secrets.GITHUB_TOKEN }}` với quyền `packages: write` |
| **Lỗi 3 (Build Context Error)** | `failed to read dockerfile: open ... ./docker/Dockerfile: no such file or directory` | Chỉ định sai build context (`./docker`) và Dockerfile path | Sửa `context: .`, `file: ./Dockerfile`, và chuẩn hóa image tag chữ thường theo chuẩn OCI |

## 3. Cấu trúc thư mục
```text
Ex05/
├── .github/
│   └── workflows/
│       └── broken-pipeline.yml   # Workflow CI/CD đã sửa và chạy thành công
├── gradle/
│   └── wrapper/                  # Gradle Wrapper
├── src/                          # Mã nguồn Spring Boot ứng dụng
├── .dockerignore                 # Danh sách loại trừ khi build Docker
├── Dockerfile                    # Containerization cho Spring Boot App
├── build.gradle                  # Cấu hình dự án Gradle
├── gradlew / gradlew.bat         # Wrapper script (đã cấp quyền thực thi +x)
├── settings.gradle               # Tên project
├── troubleshooting.md            # Báo cáo chi tiết nguyên nhân, log và giải pháp
└── README.md                     # Hướng dẫn tổng quan
```

## 4. Tài liệu nộp bài
- **Kịch bản CI/CD đã sửa**: [.github/workflows/broken-pipeline.yml](.github/workflows/broken-pipeline.yml)
- **Báo cáo phân tích sự cố**: [troubleshooting.md](troubleshooting.md)
- **GitHub Repository**: [BFMFU/PTIT_CNTT3_IT209_Session12_Ex05](https://github.com/BFMFU/PTIT_CNTT3_IT209_Session12_Ex05)
- **Kết quả Workflow Run**: Trạng thái **Success (Tích xanh)** trên tab GitHub Actions.
