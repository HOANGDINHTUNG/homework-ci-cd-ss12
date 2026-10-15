# Báo Cáo Chẩn Đoán và Khắc Phục Sự Cố Pipeline CI/CD (Bài 5)

## 1. Tổng quan
Trong quá trình triển khai tự động hoá CI/CD cho dự án Spring Boot với GitHub Actions, tệp cấu hình pipeline `.github/workflows/broken-pipeline.yml` ban đầu gặp phải 3 lỗi nghiêm trọng khiến pipeline bị đứt gãy. Tài liệu này ghi nhận chi tiết nguyên nhân gốc rễ (Root Cause) và giải pháp khắc phục cho từng lỗi.

---

## 2. Chi tiết 3 sự cố và giải pháp khắc phục

### Sự cố 1: Permission Denied trên Gradle Wrapper (`./gradlew`)
- **Triệu chứng / Log lỗi**:
  ```text
  /home/runner/work/_temp/...: line 1: ./gradlew: Permission denied
  Error: Process completed with exit code 126.
  ```
- **Nguyên nhân**:
  Tệp kịch bản `gradlew` được phát triển trên môi trường Windows hoặc khi commit vào Git không được gán quyền thực thi (`executable bit` / `chmod +x`). Khi chạy trên Linux Runner (`ubuntu-latest`), hệ điều hành từ chối thực thi tệp này.
- **Giải pháp**:
  Bổ sung bước cấp quyền thực thi cho `gradlew` ngay trước bước chạy test:
  ```yaml
  - name: Grant execute permission for gradlew
    run: chmod +x gradlew
  ```

---

### Sự cố 2: Authentication Failed khi đăng nhập Docker Registry
- **Triệu chứng / Log lỗi**:
  ```text
  Error: Username and password are required
  hoặc:
  Error: Cannot perform an interactive login from a non TTY device
  unauthorized: authentication required
  ```
- **Nguyên nhân**:
  Cấu hình bước `docker/login-action` tham chiếu sai tên biến Secret (ví dụ: gõ nhầm `${{ secrets.REGISTRY_TOKEN }}` hoặc `${{ secrets.DOCKER_PASSWORD }}` trong khi Secret này chưa được khai báo hoặc bị sai chính tả).
- **Giải pháp**:
  - Đối với GitHub Container Registry (`ghcr.io`), sử dụng biến Secret có sẵn do GitHub Actions tự động cấp phát: `${{ secrets.GITHUB_TOKEN }}`.
  - Cung cấp quyền ghi package: `permissions: packages: write`.
  - Cấu hình chính xác:
  ```yaml
  - name: Log in to GitHub Container Registry
    uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
  ```

---

### Sự cố 3: Build Context Error (Không tìm thấy Dockerfile)
- **Triệu chứng / Log lỗi**:
  ```text
  failed to solve: failed to read dockerfile: open /var/lib/docker/.../Dockerfile: no such file or directory
  Error: buildx failed with: exit code 1
  ```
- **Nguyên nhân**:
  Action `docker/build-push-action` bị cấu hình sai thư mục ngữ cảnh (`context`) hoặc đường dẫn tệp Dockerfile (`file`). Mặc định nếu không chỉ định rõ ngữ cảnh hoặc thư mục làm việc bị lệch, Docker daemon sẽ không tìm thấy `Dockerfile` ở thư mục gốc.
- **Giải pháp**:
  Khai báo tường minh tham số `context: .` (thư mục gốc dự án) và `file: ./Dockerfile`:
  ```yaml
  - name: Build and push Docker image
    uses: docker/build-push-action@v5
    with:
      context: .
      file: ./Dockerfile
      push: true
      tags: ${{ steps.meta.outputs.tags }}
      labels: ${{ steps.meta.outputs.labels }}
  ```

---

## 3. Nội dung tệp `.github/workflows/broken-pipeline.yml` sau khi khắc phục hoàn chỉnh

```yaml
name: CI/CD Pipeline - Fixed

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Set up JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      # FIX LỖI 1: Cấp quyền thực thi cho Gradle Wrapper
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Tests
        run: ./gradlew test

  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      - name: Lowercase repository name
        run: |
          echo "REPO_LC=${GITHUB_REPOSITORY,,}" >> $GITHUB_ENV

      # FIX LỖI 2: Đăng nhập Docker Registry bằng secrets.GITHUB_TOKEN chuẩn
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ env.REPO_LC }}
          tags: |
            type=raw,value=latest

      # FIX LỖI 3: Khai báo context: . và file: ./Dockerfile
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## 4. Kết luận
Sau khi áp dụng đầy đủ 3 giải pháp trên, toàn bộ pipeline đã được phục hồi hoàn toàn:
- Job **Run Unit Tests**: Tích xanh (Tests passed).
- Job **Build & Push Docker Image**: Tích xanh (Image đóng gói và đẩy lên GHCR thành công).
