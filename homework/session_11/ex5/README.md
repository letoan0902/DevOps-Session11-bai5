# Bài 5 - Nối job tự động Build & Push Docker Image sau khi test Gradle xanh

Học viên: Lê Phú Toàn
Repo: `letoan0902/DevOps-Session11-bai5`
Image đích: `ghcr.io/letoan0902/devops-session11-bai5`

## 1. Mục tiêu

Tách luồng CI thành 2 job riêng biệt trong cùng một file `ci.yml`:

1. Job `test`: chạy JUnit test bằng Gradle và đóng gói file JAR (`app.jar`).
2. Job `docker-build`: đóng gói Dockerfile của ứng dụng Spring Boot thành image và đẩy lên GitHub Container Registry (`ghcr.io`).

Hai job được ràng buộc thứ tự bằng từ khóa `needs: test`, nhờ đó image chỉ được build và push khi toàn bộ test đã xanh. Nếu test đỏ, job `docker-build` được bỏ qua (trạng thái Skipped) và không có image lỗi nào được xuất bản.

Yêu cầu đề bài được đáp ứng như sau:

- Job thứ 2 tên `docker-build` (`name: Build and Push Docker Image`).
- Có `needs: test`.
- Trong `docker-build` có 2 hành động chính: đăng nhập registry (`docker/login-action@v3`) và build + push (`docker/build-push-action@v5`).
- Đích push là `ghcr.io`.

## 2. Kiến trúc pipeline

```
   git push / pull_request / workflow_dispatch
                     |
                     v
   +--------------------------------------------+
   | Job 1: test  (name: Test Gradle)           |
   |   1. actions/checkout@v5                   |
   |   2. actions/setup-java@v4  (Temurin 21)   |
   |   3. gradle/actions/setup-gradle@v4 (8.10) |
   |   4. gradle test --no-daemon               |
   |   5. gradle bootJar --no-daemon            |
   |   6. upload-artifact: app-jar              |
   |   7. upload-artifact: test-report (always) |
   +--------------------------------------------+
                     |
                     |  needs: test
                     |  (chỉ đi tiếp khi job test = success)
                     |  test đỏ  -> docker-build = Skipped
                     v
   +--------------------------------------------------+
   | Job 2: docker-build                              |
   |   (name: Build and Push Docker Image)            |
   |   1. actions/checkout@v5                         |
   |   2. docker/setup-buildx-action@v3               |
   |   3. docker/login-action@v3  -> ghcr.io          |
   |   4. docker/metadata-action@v5 (tag sha, latest) |
   |   5. docker/build-push-action@v5 (push)          |
   +--------------------------------------------------+
                     |
                     v
   ghcr.io/letoan0902/devops-session11-bai5:sha-xxxxxxx
   ghcr.io/letoan0902/devops-session11-bai5:latest   (nhánh main)
```

## 3. Giải thích từng phần của `ci.yml`

Toàn bộ nội dung nằm ở `.github/workflows/ci.yml` (bản sao cùng thư mục với báo cáo này).

### 3.1. Tên và sự kiện kích hoạt

```yaml
name: CI - Test Gradle va Build/Push Docker Image

on:
  push:
    branches: ["**"]
  pull_request:
  workflow_dispatch:
```

- `push` với `branches: ["**"]` kích hoạt workflow khi push lên mọi nhánh. Điều này cần thiết để kịch bản làm đứt test trên nhánh `break-test` cũng chạy CI.
- `pull_request` chạy khi mở hoặc cập nhật PR. Khi đó image chỉ được build thử, không push.
- `workflow_dispatch` cho phép bấm "Run workflow" thủ công trên tab Actions.

### 3.2. Quyền của `GITHUB_TOKEN`

```yaml
permissions:
  contents: read
  packages: write
```

- `contents: read` đủ để `actions/checkout` tải mã nguồn.
- `packages: write` cho phép ghi package lên `ghcr.io` khi dùng `GITHUB_TOKEN`. Trong bài này việc đăng nhập dùng secret `REGISTRY_TOKEN` (xem mục 8), nhưng khai báo quyền tối thiểu này vẫn là thực hành tốt và giúp chuyển sang `GITHUB_TOKEN` dễ dàng nếu cần.

### 3.3. Biến môi trường `IMAGE_NAME`

```yaml
env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}
```

`github.repository` có dạng `letoan0902/DevOps-Session11-bai5`, chứa chữ hoa. Docker yêu cầu tên repository của image phải viết thường, nếu không sẽ báo lỗi `invalid reference format`. Workflow xử lý điều này bằng `docker/metadata-action@v5`: action này tự chuyển phần tên image về chữ thường khi sinh tag, nên kết quả là `ghcr.io/letoan0902/devops-session11-bai5`. Cách khác là dùng bước `tr '[:upper:]' '[:lower:]'` để chuyển thường rồi ghi vào `$GITHUB_ENV`; bài này chọn `metadata-action` vì gọn hơn và đồng thời sinh label OCI.

### 3.4. Job `test`

| Bước | Action / lệnh | Vai trò |
|------|---------------|---------|
| Checkout | `actions/checkout@v5` | Lấy mã nguồn về runner |
| JDK | `actions/setup-java@v4` (`temurin`, `21`) | Cài Java 21 đúng với toolchain trong `build.gradle` |
| Gradle | `gradle/actions/setup-gradle@v4` (`gradle-version: "8.10"`) | Cài Gradle 8.10 và bật cache Gradle |
| Test | `gradle test --no-daemon` | Chạy JUnit; nếu có test đỏ thì lệnh thoát khác 0 và job thất bại |
| Đóng gói | `gradle bootJar --no-daemon` | Tạo `build/libs/app.jar` |
| Artifact JAR | `actions/upload-artifact@v4` (`app-jar`) | Lưu file JAR để tải về |
| Artifact báo cáo | `actions/upload-artifact@v4` (`test-report`, `if: always()`) | Lưu báo cáo HTML của test, kể cả khi test thất bại |

Về Gradle wrapper: repo không commit `gradle-wrapper.jar` (file nhị phân) nên không có `gradlew`. Thay vào đó workflow dùng `gradle/actions/setup-gradle@v4` với `gradle-version: "8.10"` rồi gọi lệnh `gradle` trực tiếp. Cách này tránh file nhị phân trong repo mà vẫn cố định phiên bản Gradle; Dockerfile cũng dùng image `gradle:8.10-jdk21` cùng phiên bản.

`if: always()` ở bước upload báo cáo rất hữu ích: khi test đỏ, các bước phía sau mặc định bị bỏ qua, nhưng báo cáo test lại chính là thứ cần xem nhất.

### 3.5. Job `docker-build` và từ khóa `needs`

```yaml
  docker-build:
    name: Build and Push Docker Image
    needs: test
    runs-on: ubuntu-latest
```

Mặc định, các job trong một workflow chạy song song. `needs: test` thay đổi hành vi đó:

- Chạy tuần tự: `docker-build` chỉ được xếp lịch sau khi job `test` kết thúc.
- Điều kiện ngầm định là `success()` của các job được liệt kê trong `needs`. Nếu `test` thất bại, bị hủy hoặc bị bỏ qua, `docker-build` không chạy và hiển thị trạng thái Skipped (màu xám, không phải đỏ).
- Sơ đồ job trên tab Actions vẽ mũi tên từ `test` sang `docker-build`.
- Khác với `if: success()`: `if: success()` chỉ là điều kiện, không tự tạo quan hệ phụ thuộc hay thứ tự giữa các job. Nếu chỉ viết `if: success()` mà không có `needs`, hai job vẫn chạy song song và `success()` chỉ đánh giá trạng thái của chính job đó. Ngược lại `needs` vừa tạo thứ tự, vừa tự áp điều kiện thành công của job phụ thuộc, vừa cho phép đọc kết quả qua `needs.test.result` và `needs.test.outputs`.
- Muốn chạy `docker-build` ngay cả khi `test` thất bại thì phải viết rõ `if: always()` hoặc `if: ${{ !cancelled() }}`; bài này không làm vậy vì mục tiêu là chặn image khi test đỏ.

### 3.6. Các bước trong `docker-build`

1. `actions/checkout@v5`: lấy mã nguồn (cần có Dockerfile và `src`).
2. `docker/setup-buildx-action@v3`: bật Buildx, cần cho cache `type=gha`.
3. `docker/login-action@v3`: đăng nhập `ghcr.io` với `username: ${{ github.actor }}` và `password: ${{ secrets.REGISTRY_TOKEN }}`. Bước này có `if: github.event_name != 'pull_request'` vì PR không push image, và PR từ fork không đọc được secret.
4. `docker/metadata-action@v5` (`id: meta`): sinh danh sách tag và label.
   - `type=sha`: tag dạng `sha-<7 ký tự đầu của commit>`, mỗi commit một tag, giúp truy vết.
   - `type=raw,value=latest,enable={{is_default_branch}}`: chỉ gắn tag `latest` khi push lên nhánh mặc định (`main`).
5. `docker/build-push-action@v5`:
   - `context: .` dùng Dockerfile ở gốc repo.
   - `push: ${{ github.event_name != 'pull_request' }}`: push ở mọi sự kiện trừ pull request.
   - `tags` và `labels` lấy từ đầu ra của bước `meta`.
   - `cache-from: type=gha` và `cache-to: type=gha,mode=max`: dùng GitHub Actions cache để tăng tốc các lần build sau.

## 4. Dự án Spring Boot và Dockerfile

### 4.1. Ứng dụng

- `build.gradle`: plugin `java`, `org.springframework.boot` 3.3.4, `io.spring.dependency-management` 1.1.6, Java toolchain 21; phụ thuộc `spring-boot-starter-web`, `spring-boot-starter-actuator` và `spring-boot-starter-test` (test); `bootJar` đặt tên cố định `app.jar`.
- `HelloController`:
  - `GET /` trả chuỗi `Hello from PTIT DevOps Session 11 - Le Phu Toan`.
  - `GET /api/version` trả JSON `{"app":"devops-session11-bai5","version":"1.0.0"}`.
- `application.properties`: `server.port=8080` và chỉ mở endpoint `health` của actuator.
- `HelloControllerTest`: dùng `@SpringBootTest` và `@AutoConfigureMockMvc` với 2 test: `rootReturnsGreeting` kiểm tra `GET /` chứa chuỗi chào; `versionReturnsExpectedJson` kiểm tra `GET /api/version` có `app` và `version` bằng `1.0.0`.

### 4.2. Dockerfile multi-stage

```dockerfile
# Stage 1: build file JAR bằng Gradle
FROM gradle:8.10-jdk21 AS build
WORKDIR /home/gradle/project
COPY settings.gradle build.gradle ./
COPY src ./src
RUN gradle bootJar --no-daemon

# Stage 2: image chạy tối giản, không dùng quyền root
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN adduser -D app
COPY --from=build /home/gradle/project/build/libs/app.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

- Stage `build` có đầy đủ JDK và Gradle nhưng không đi vào image cuối, nên image runtime nhỏ gọn (chỉ JRE Alpine và `app.jar`).
- Stage runtime tạo user `app` không phải root và chạy tiến trình bằng user này (`USER app`), giảm rủi ro nếu ứng dụng bị khai thác.
- `.dockerignore` loại `build/`, `.gradle/`, `.git`, `.github/`, `homework/` để context nhỏ và tránh làm mất cache khi chỉ sửa tài liệu.

## 5. Kết quả kiểm chứng trên máy local

Máy local không có Gradle nên dùng Docker để chạy đúng phiên bản Gradle 8.10 và JDK 21 mà workflow dùng.

### 5.1. Gradle test và bootJar

Lệnh:

```bash
MSYS_NO_PATHCONV=1 docker run --rm -v "$PWD:/home/gradle/project" -w /home/gradle/project \
  gradle:8.10-jdk21 gradle test bootJar --no-daemon --console=plain
```

Output quan trọng:

```
> Task :compileJava
> Task :processResources
> Task :classes
> Task :resolveMainClassName
> Task :bootJar
> Task :compileTestJava
> Task :processTestResources NO-SOURCE
> Task :testClasses
> Task :test

BUILD SUCCESSFUL in 2m 11s
6 actionable tasks: 6 executed
```

Kết quả test đọc từ `build/test-results/test/TEST-vn.ptit.devops.HelloControllerTest.xml`:

```
<testsuite name="vn.ptit.devops.HelloControllerTest" tests="2" skipped="0" failures="0" errors="0" ...>
<testcase name="versionReturnsExpectedJson()" classname="vn.ptit.devops.HelloControllerTest" .../>
<testcase name="rootReturnsGreeting()" classname="vn.ptit.devops.HelloControllerTest" .../>
```

2 test, 0 thất bại, 0 lỗi. File `build/libs/app.jar` được tạo với dung lượng khoảng 22 MB.

### 5.2. Kiểm chứng patch làm đứt test

Áp `break_test.patch` rồi chạy lại `gradle test` bằng cùng lệnh Docker:

```
HelloControllerTest > versionReturnsExpectedJson() FAILED
    java.lang.AssertionError at HelloControllerTest.java:34

2 tests completed, 1 failed

> Task :test FAILED
...
BUILD FAILED
```

Sau đó hoàn tác patch (`git apply -R`) và test xanh trở lại. Đây chính là tình huống làm job `test` đỏ trên CI, khi đó `docker-build` bị Skipped.

### 5.3. Docker build và chạy container

```bash
docker build -t devops-session11-bai5:local .
```

Stage `build` chạy `gradle bootJar` và kết thúc bằng `BUILD SUCCESSFUL in 26s`, sau đó stage runtime sao chép `app.jar`:

```
#11 27.24 > Task :bootJar
#11 27.24 BUILD SUCCESSFUL in 26s
#11 DONE 27.5s
#14 [stage-1 4/4] COPY --from=build /home/gradle/project/build/libs/app.jar app.jar
#14 DONE 0.1s
#15 naming to docker.io/library/devops-session11-bai5:local done
```

Chạy container và gọi thử:

```bash
docker run -d --rm -p 8086:8080 --name s11b5 devops-session11-bai5:local
curl -s http://localhost:8086/
curl -s http://localhost:8086/api/version
curl -s http://localhost:8086/actuator/health
docker exec s11b5 whoami
```

```
Hello from PTIT DevOps Session 11 - Le Phu Toan
{"app":"devops-session11-bai5","version":"1.0.0"}
{"status":"UP"}
app
```

Log khởi động: `Tomcat started on port 8080 (http) with context path '/'` và `Started DevopsApplication in 2.151 seconds`. Lệnh `whoami` trả `app` xác nhận container chạy bằng user không phải root.

Dọn dẹp: `docker rm -f s11b5` và `docker rmi devops-session11-bai5:local`; không còn container hoặc image nào do bài này tạo.

### 5.4. Lint workflow bằng actionlint

```bash
MSYS_NO_PATHCONV=1 docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest \
  -no-color .github/workflows/ci.yml homework/session_11/ex5/ci.yml
```

Kết quả: không có cảnh báo hay lỗi nào, mã thoát `0` (actionlint 1.7.12). Hai file `ci.yml` giống hệt nhau (đã so sánh bằng `cmp`).

## 6. Hướng dẫn kiểm tra trên tab Actions

### 6.1. Sơ đồ 2 job có mũi tên

1. Vào repo trên GitHub, mở tab Actions, chọn workflow `CI - Test Gradle va Build/Push Docker Image`.
2. Mở một lần chạy (run) bất kỳ. Trang Summary hiển thị sơ đồ ở phần Jobs.
3. Sơ đồ gồm 2 khối: `Test Gradle` ở bên trái, `Build and Push Docker Image` ở bên phải, nối với nhau bằng một mũi tên từ `Test Gradle` sang `Build and Push Docker Image`. Đây là hình ảnh của quan hệ `needs: test`.
4. Chụp màn hình sơ đồ này làm ảnh nộp bài (ví dụ lưu thành `homework/session_11/ex5/workflow-graph.png`).

### 6.2. Cách đọc Summary

- Khối bên trái là `Test Gradle`: dấu tích xanh nghĩa là JUnit test đạt và đã đóng gói JAR.
- Khối bên phải là `Build and Push Docker Image`: chỉ bắt đầu chạy sau khi khối bên trái xanh.
- Phần Artifacts ở cuối trang liệt kê `app-jar` và `test-report`.
- Bấm vào từng job để xem log chi tiết từng bước; ở job `docker-build`, bước "Build và push image" in ra digest của image sau khi push thành công.

## 7. Kịch bản làm đứt test để thấy docker-build bị Skipped

Mục đích: chứng minh `needs: test` thật sự chặn việc build image khi test đỏ.

### 7.1. Các bước

```bash
git checkout -b break-test
git apply homework/session_11/ex5/break_test.patch
git commit -am "Làm đứt test: kỳ vọng version 9.9.9"
git push -u origin break-test
```

Nếu không muốn dùng patch, chỉ cần sửa một dòng trong `src/test/java/vn/ptit/devops/HelloControllerTest.java`: đổi `.value("1.0.0")` thành `.value("9.9.9")`.

### 7.2. Nội dung patch

```diff
--- a/src/test/java/vn/ptit/devops/HelloControllerTest.java
+++ b/src/test/java/vn/ptit/devops/HelloControllerTest.java
@@ -31,6 +31,6 @@ class HelloControllerTest {
         mockMvc.perform(get("/api/version"))
                 .andExpect(status().isOk())
                 .andExpect(jsonPath("$.app").value("devops-session11-bai5"))
-                .andExpect(jsonPath("$.version").value("1.0.0"));
+                .andExpect(jsonPath("$.version").value("9.9.9"));
     }
 }
```

### 7.3. Kết quả mong đợi

| Job | Trạng thái mong đợi | Giải thích |
|-----|---------------------|------------|
| `Test Gradle` | Failed (đỏ) | Test `versionReturnsExpectedJson` thất bại vì `1.0.0` khác `9.9.9` |
| `Build and Push Docker Image` | Skipped (xám) | `needs: test` không thỏa vì `test` không thành công, job không được khởi chạy |

Trên sơ đồ, khối `docker-build` có biểu tượng bỏ qua (Skipped). Không có image nào được đẩy lên `ghcr.io` cho commit này. Báo cáo test vẫn được tải lên nhờ `if: always()`, có thể tải artifact `test-report` để xem chi tiết.

Sau khi kiểm tra có thể xóa nhánh `break-test` hoặc để nguyên làm bằng chứng nộp bài.

## 8. Secret `REGISTRY_TOKEN` và trạng thái trước/sau

Bước đăng nhập dùng `password: ${{ secrets.REGISTRY_TOKEN }}`. Đây là Personal Access Token do học viên tự tạo theo hướng dẫn của bài 4 (quyền `write:packages`, kèm `read:packages`), sau đó lưu vào secret của repo. Giá trị token không bao giờ được ghi vào mã nguồn hay tài liệu.

### 8.1. Cách thêm secret

1. Tạo token theo bài 4 (GitHub -> Settings -> Developer settings -> Personal access tokens), cấp quyền `write:packages`.
2. Vào repo `letoan0902/DevOps-Session11-bai5` -> Settings -> Secrets and variables -> Actions -> New repository secret.
3. Đặt tên `REGISTRY_TOKEN`, dán giá trị token, lưu lại.
4. Vào tab Actions, mở lần chạy bị lỗi và bấm "Re-run failed jobs" (hoặc push một commit mới).

### 8.2. Trạng thái trước và sau khi có secret

| Tình huống | Job `Test Gradle` | Job `Build and Push Docker Image` |
|------------|-------------------|------------------------------------|
| Chưa có secret `REGISTRY_TOKEN` | Thành công | Chạy tới bước đăng nhập rồi thất bại (thiếu mật khẩu) |
| Đã có secret hợp lệ, test xanh | Thành công | Thành công; image xuất hiện ở mục Packages của tài khoản |
| Đã có secret, test đỏ (nhánh `break-test`) | Thất bại | Skipped |
| Pull request | Thành công | Build thử, bỏ qua đăng nhập và không push |

Điểm cần nhớ: việc `docker-build` đỏ khi chưa có secret là hành vi bình thường và không phải lỗi mã nguồn; chỉ cần thêm secret rồi chạy lại. Lần đầu push, package trên `ghcr.io` có thể ở chế độ private; muốn kéo công khai cần đổi visibility trong Package settings.

## 9. Kết quả chạy thật trên GitHub Actions

(người điều phối cập nhật sau khi push)

## 10. Xử lý sự cố

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|-------------|------------------------|------------|
| Job `test` đỏ | Có JUnit test thất bại hoặc lỗi biên dịch | Mở log bước "Chạy JUnit test", tải artifact `test-report` để xem test nào đỏ, sửa code rồi push lại |
| `invalid reference format` khi build/push | Tên image chứa chữ hoa (ví dụ `DevOps-Session11-bai5`) | Dùng `docker/metadata-action` để sinh tag (tự chuyển chữ thường) hoặc chuyển thường bằng `tr` trước khi truyền vào `tags` |
| `denied: permission_denied` hoặc `write_package` khi push | Thiếu `packages: write`, token không có quyền `write:packages`, hoặc package đã tồn tại và không gắn với repo | Kiểm tra khối `permissions`, tạo lại token đúng quyền, liên kết package với repo trong Package settings |
| `denied` hoặc `unauthorized` ngay bước login | Chưa có secret `REGISTRY_TOKEN`, sai tên secret, hoặc username không khớp chủ token | Thêm secret đúng tên, dùng `username: ${{ github.actor }}` đúng tài khoản tạo token |
| Login lỗi sau một thời gian chạy ổn | Token hết hạn hoặc bị thu hồi | Tạo token mới và cập nhật lại secret `REGISTRY_TOKEN` |
| `docker-build` Skipped dù test xanh | `needs` trỏ sai tên job, hoặc job `test` bị hủy | Đối chiếu `needs: test` với khóa job `test`; chạy lại workflow |
| Build chậm hoặc cache không trúng | Cache `type=gha` chưa có (lần đầu) hoặc đã bị dọn do vượt giới hạn | Chấp nhận lần đầu chậm, các lần sau trúng cache; có thể bỏ `cache-from`/`cache-to` nếu nghi ngờ cache hỏng |
| Không tìm thấy `app.jar` khi build image | `bootJar` không tạo đúng tên | Kiểm tra `archiveFileName = 'app.jar'` trong `build.gradle` và đường dẫn `COPY --from=build` trong Dockerfile |
| `gradlew: command not found` | Repo không có Gradle wrapper | Dùng `gradle/actions/setup-gradle@v4` và gọi `gradle` như trong workflow này |

## 11. Bảng đối chiếu yêu cầu đề bài

| Yêu cầu | Đáp ứng ở đâu | Trạng thái |
|---------|---------------|------------|
| Tách luồng CI thành 2 job `test` và `docker-build` | `jobs.test` và `jobs.docker-build` trong `ci.yml` | Đạt |
| Job `test`: Gradle test và đóng gói JAR | Bước `gradle test` và `gradle bootJar` | Đạt |
| Job thứ 2 tên `docker-build` | Khóa `docker-build`, `name: Build and Push Docker Image` | Đạt |
| Dùng `needs: test` | Dòng `needs: test` trong job `docker-build` | Đạt |
| Đăng nhập Registry | `docker/login-action@v3` vào `ghcr.io` | Đạt |
| Chạy `docker/build-push-action@v5` đẩy lên `ghcr.io` | Bước "Build và push image", tag từ `metadata-action` | Đạt |
| Sơ đồ job có mũi tên từ `test` sang `docker-build` | Mục 6.1; ảnh chụp lấy từ tab Actions | Chờ ảnh chụp sau khi chạy thật |
| Sửa code làm đứt JUnit test thì `docker-build` Skipped | Patch `break_test.patch`, kịch bản mục 7; đã kiểm chứng test đỏ ở local | Đã kiểm chứng local, chờ kết quả trên Actions |
| Nộp `.github/workflows/ci.yml` | Có ở gốc repo và bản sao `homework/session_11/ex5/ci.yml` | Đạt |
| Nộp ảnh chụp sơ đồ workflow | Chụp từ tab Actions theo mục 6.1 | Chờ ảnh chụp |

## 12. Kết luận

Workflow chia CI thành 2 job rõ ràng: `test` xác nhận chất lượng mã nguồn bằng JUnit và tạo JAR, `docker-build` đóng gói và xuất bản image lên `ghcr.io`. Từ khóa `needs: test` là chốt chặn cần thiết để đảm bảo thứ tự: test xanh mới build image, test đỏ thì image không được tạo và job bị Skipped. Dự án mẫu đã được kiểm chứng ở local bằng Docker (Gradle `BUILD SUCCESSFUL`, 2 test đạt, image chạy được, `/actuator/health` trả `UP`), workflow qua actionlint không có cảnh báo. Việc còn lại phụ thuộc môi trường GitHub: thêm secret `REGISTRY_TOKEN` để bước đăng nhập và push thành công, rồi chụp sơ đồ 2 job trên tab Actions để nộp.
