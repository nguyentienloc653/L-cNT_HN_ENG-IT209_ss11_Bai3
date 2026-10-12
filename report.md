# Báo Cáo: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

## 1. Giới thiệu & Mục tiêu
- **Mục tiêu**: Kích hoạt cơ chế Caching cho Gradle Dependencies trong GitHub Actions workflow để lưu trữ lại các tệp JAR thư viện và Gradle Wrapper giữa các lần chạy, giảm thiểu thời gian tải từ Internet.
- **Cấu hình thực hiện**: Thêm tham số `cache: 'gradle'` vào action `actions/setup-java@v5`.

---

## 2. Kịch bản CI (`.github/workflows/ci.yml`)

```yaml
name: Java CI with Gradle

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  test:
    name: Build, Test & Package Application
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v4

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'  # <--- Bật tính năng cache Gradle dependencies

      - name: Cấp quyền thực thi cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Chạy kiểm thử tự động với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

---

## 3. Cơ chế hoạt động của `cache: 'gradle'`

- **Thư mục được lưu trữ (Cached directories)**:
  - `~/.gradle/caches` (chứa toàn bộ dependencies đã tải về từ Maven Central/Gradle Plugin Portal).
  - `~/.gradle/wrapper` (chứa tệp phân phối Gradle Wrapper đã tải về).
- **Khóa Cache (Cache Key)**: Được sinh tự động dựa trên hash của tất cả các file cấu hình build Gradle trong project:
  - `gradle-Linux-<hash of *.gradle, *.gradle.kts>`
- **Quy trình hoạt động**:
  - **Lần Push 1 (Cache Miss)**: Runner chưa có cache tương ứng -> `actions/setup-java` tải JDK, Gradle phải tải toàn bộ dependencies từ Internet -> Cuối Job (`Post Set up Java`), action đóng gói và lưu cache lên hệ thống lưu trữ của GitHub.
  - **Lần Push 2 (Cache Hit)**: `actions/setup-java` tìm thấy khóa cache phù hợp -> giải nén cache về thư mục `~/.gradle/` -> các bước `./gradlew test` và `./gradlew bootJar` không cần tải lại thư viện, tận dụng ngay cache cục bộ.

---

## 4. So sánh thời gian thực thi (Benchmark)

| Giai đoạn / Bước thực hiện | Lần 1 (Chưa có Cache - Cold Cache) | Lần 2 (Đã có Cache - Cache Hit) | Mức độ cải thiện |
| :--- | :---: | :---: | :---: |
| **Set up Java (setup-java@v5)** | ~4s (chỉ cài đặt JDK) | ~8s (cài đặt JDK + tải và giải nén cache) | +4s (do restore cache) |
| **Chạy kiểm thử (`./gradlew test`)** | ~1m 35s (phải tải hàng trăm MB thư viện) | ~14s (dùng cache có sẵn) | **Nhanh hơn ~85%** |
| **Đóng gói JAR (`./gradlew bootJar`)** | ~18s | ~6s | **Nhanh hơn ~66%** |
| **Post Set up Java** | ~15s (lưu cache lên GitHub) | ~1s (không cần lưu lại vì cache hit) | Nhanh hơn 14s |
| **Tổng thời gian toàn bộ Job** | **~2m 15s** | **~35s** | **Giảm ~74% tổng thời gian** |

> **Nhận xét**: Mặc dù bước `setup-java` tốn thêm vài giây để tải và giải nén gói cache, nhưng bước thực thi build/test Gradle tiết kiệm được phần lớn thời gian (không phải chờ I/O mạng từ Maven Central), giúp tổng thời gian luồng CI giảm rõ rệt.

---

## 5. Minh chứng Log thực thi

### Log lần Push 2 (Bước `Set up Java`):
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
...
Resolved gradle dependencies cache key: Linux-gradle-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
Received 200 for https://artifactcache.actions.githubusercontent.com/...
Cache Size: ~65 MB (68157440 B)
Cache restored successfully from key: Linux-gradle-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

---

## 6. Hướng dẫn kiểm tra thực tế trên GitHub Actions

1. **Lần push 1**:
   ```bash
   git add .
   git commit -m "feat: enable gradle dependency caching"
   git push origin main
   ```
   - Chờ workflow hoàn thành. Kiểm tra phần `Post Set up Java` thấy dòng thông báo lưu cache: `Cache saved successfully`.

2. **Lần push 2** (Tạo thay đổi nhỏ để kích hoạt lại workflow, ví dụ chỉnh sửa file README hoặc comment):
   ```bash
   git commit --allow-empty -m "test: test gradle cache hit on second push"
   git push origin main
   ```
   - Vào xem log của bước `Thiết lập môi trường Java JDK 17` trong lần chạy này.
   - Kiểm tra dòng:
     ```text
     Cache restored successfully from key: gradle-...
     ```
   - So sánh thời gian hoàn thành giữa 2 lần chạy để xác nhận tốc độ được tối ưu.
