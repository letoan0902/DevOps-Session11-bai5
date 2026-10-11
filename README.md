# DevOps Session 11 - Bài 5: Nối job Build & Push Docker Image sau khi test Gradle xanh

Dự án Spring Boot (Gradle, Java 21) tối giản dùng để minh họa workflow GitHub Actions gồm 2 job liên kết bằng `needs`:

- `test`: chạy JUnit test bằng Gradle và đóng gói file JAR.
- `docker-build`: chỉ chạy khi `test` thành công; đăng nhập `ghcr.io`, build và push Docker image.

Trạng thái CI: (cập nhật sau khi push)

## Chạy thử trên máy local

```bash
# Build image
docker build -t devops-session11-bai5:local .

# Chạy container
docker run -d --rm -p 8086:8080 --name s11b5 devops-session11-bai5:local

# Kiểm tra
curl -s http://localhost:8086/
curl -s http://localhost:8086/api/version
curl -s http://localhost:8086/actuator/health

# Dọn dẹp
docker rm -f s11b5
docker rmi devops-session11-bai5:local
```

## Báo cáo và workflow

- Báo cáo chi tiết: [homework/session_11/ex5/README.md](homework/session_11/ex5/README.md)
- Workflow chạy thật: [.github/workflows/ci.yml](.github/workflows/ci.yml)
- Bản sao để nộp bài: [homework/session_11/ex5/ci.yml](homework/session_11/ex5/ci.yml)
- Patch làm đứt test (để kiểm tra trạng thái Skipped): [homework/session_11/ex5/break_test.patch](homework/session_11/ex5/break_test.patch)

## Danh sách file

```
.
|-- .github/workflows/ci.yml          # workflow 2 job: test -> docker-build
|-- src/main/java/vn/ptit/devops/     # DevopsApplication, HelloController
|-- src/main/resources/application.properties
|-- src/test/java/vn/ptit/devops/HelloControllerTest.java
|-- build.gradle, settings.gradle     # cấu hình Gradle (không commit wrapper jar)
|-- Dockerfile                        # multi-stage: gradle build -> temurin JRE alpine
|-- .dockerignore, .gitignore, .gitattributes
|-- homework/session_11/ex5/
    |-- README.md                     # báo cáo bài tập
    |-- ci.yml                        # bản sao workflow để nộp
    |-- break_test.patch              # patch làm đứt JUnit test
```

## Lưu ý về secret

Job `docker-build` đăng nhập `ghcr.io` bằng secret `REGISTRY_TOKEN` (Personal Access Token, tạo theo bài 4). Khi chưa thêm secret, job `test` vẫn xanh nhưng `docker-build` sẽ thất bại ở bước đăng nhập. Xem hướng dẫn trong báo cáo.
