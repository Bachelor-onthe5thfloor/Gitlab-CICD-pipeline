# Mô hình triển khai

<img width="1953" height="1011" alt="image" src="https://github.com/user-attachments/assets/65018278-b9d9-41ac-8d3d-86d1f42317ed" />

Mô hình của bạn được xây dựng theo nguyên tắc tách biệt môi trường và tự động hóa bảo mật, đảm bảo mọi thay đổi về mã nguồn đều được kiểm duyệt chặt chẽ trước khi lên môi trường Production.

**GitLab Server:** Đóng vai trò là Source Code Management (SCM) và CI/CD Orchestrator. Đây là nơi khởi nguồn của mọi luồng công việc, quản lý mã nguồn tập trung và điều phối các lệnh thực thi đến các Runner tương ứng thông qua file cấu hình `.gitlab-ci.yml`.

**Build & Deploy Runner (Môi trường Staging/Temporary):** * Chịu trách nhiệm thực hiện các tác vụ biên dịch (Compile) và đóng gói (Packaging).

- **Đặc điểm quan trọng:** Runner này kiêm luôn việc triển khai một bản website tạm thời (Ephemereal Environment). Mục đích của việc này là tạo ra một "bia bắn" thực tế để Test Runner có thể thực hiện quét động (DAST).

**Test Runner (Security & Quality Assurance):** * Đây là "trung tâm kiểm soát an ninh" của Pipeline. Nó thực hiện các bài thử nghiệm khắc nghiệt bao gồm:
* **SAST & SCA:** Quét mã nguồn tĩnh và thư viện bên thứ ba ngay khi code được đưa lên.
* **Container Scan:** Quét các tầng lớp (layers) của Docker Image.
* **DAST (Dynamic Analysis):** Tương tác trực tiếp với website tạm thời trên Build Runner để tìm lỗ hổng runtime.
* **Performance Test:** Đảm bảo ứng dụng chịu tải tốt trước khi phát hành.

**Harbor Private Registry:** Thay vì dùng Docker Hub công cộng, bạn sử dụng Harbor. Quản lý vòng **đời:** Harbor cho phép bạn quản lý Image qua từng giai đoạn (Dev -> Staging -> Prod).

**K3s Node (Production):** * Lựa chọn **K3s** giúp tối ưu hóa tài nguyên cho hệ thống nhưng vẫn giữ được sức mạnh điều phối của Kubernetes.

- Ứng dụng chỉ được triển khai lên đây sau khi đã vượt qua tất cả các chốt chặn bảo mật tại Test Runner và được lưu trữ an toàn trong Harbor.

**Zabbix Server:** Đóng vai trò là "mắt thần" cho toàn bộ hệ thống.

- Zabbix giám sát sức khỏe của mọi thành phần từ GitLab, các Runners, Harbor cho đến tình trạng tiêu thụ tài nguyên trên cụm K3s.
- Điều này hoàn thiện vòng lặp DevOps (Monitor), giúp đội ngũ vận hành phát hiện sớm các bất thường về hiệu năng hoặc các cuộc tấn công từ chối dịch vụ (DoS).

# Kịch bản 1: Xây dựng CI/CD pipeline tích hợp công cụ kiểm thử bảo mật mã nguồn và Docker Image

<img width="1411" height="636" alt="image 1" src="https://github.com/user-attachments/assets/1494fb14-c862-4062-8cf6-18623a1472f9" />


Mô tả luồng hoạt động của pipeline:

- **Developer - Push code (Lập trình viên):**
    - Lập trình viên hoàn thiện mã nguồn trên máy cá nhân và thực hiện lệnh `push` code lên hệ thống quản lý phiên bản.
- **Gitlab-CI (Hệ thống điều phối):**
    - Gitlab nhận diện có sự thay đổi trong code và tự động kích hoạt (trigger) pipeline. Đây là "bộ não" điều khiển tất cả các bước tiếp theo.
- **Maven - Build (Biên dịch):**
    - Sử dụng công cụ Maven để biên dịch mã nguồn, quản lý các thư viện phụ thuộc và kiểm tra xem code có lỗi logic cơ bản nào không thông qua việc chạy Unit Tests.
- **SonarQube - Code quality check (Kiểm tra chất lượng):**
    - Code sau khi build sẽ được đẩy qua SonarQube để phân tích "tĩnh". Công cụ này tìm các lỗi tiềm ẩn (bugs), các đoạn code thừa, hoặc các lỗ hổng bảo mật dựa trên quy tắc có sẵn (Static Application Security Testing - SAST).
- **Trivy - SCA (Kiểm tra thư viện bên thứ ba):**
    - Trước khi đóng gói, Trivy thực hiện quét SCA (Software Composition Analysis). Bước này cực kỳ quan trọng để kiểm tra xem các thư viện mã nguồn mở mà dự án đang dùng có chứa lỗ hổng bảo mật nào đã biết hay không.
- **Docker - Build and push to registry (Đóng gói):**
    - Nếu code sạch và an toàn, hệ thống sẽ đóng gói ứng dụng vào một Docker Image. Sau đó, Image này được đẩy (push) lên một kho lưu trữ (Registry) để chuẩn bị cho việc triển khai.
- **Trivy - Docker image Scan (Quét lỗ hổng Image):**
    - Cuối cùng, Trivy quét lại một lần nữa toàn bộ Docker Image (bao gồm cả hệ điều hành bên trong image đó) để đảm bảo không có lỗ hổng hệ thống nào trước khi mang đi chạy thực tế.

Code build and unit test:

```yaml
build:
  stage: build
  variables:
    GIT_STRATEGY: clone
  script:
    - mvn clean install 
  artifacts:
    paths:
      - target/ 
    expire_in: 1 hour
  tags:
    - $runner
  rules:
    - if: "$CI_COMMIT_TAG"
      when: always

```

- ***giải thích code***
    - GIT_STRATEGY: clone
        
        Bắt GitLab Runner clone lại toàn bộ repository, Đảm bảo code mới 100%
        
    - mvn clean install: Thực hiện build project Java bằng Maven: xóa thư mục build cũ, biên dịch mã nguồn, chạy test và tạo artifact.
    - Chỉ định đường dẫn các file/thư mục cần lưu làm artifact. Lưu toàn bộ thư mục `target/`, nơi chứa file `.jar`, `.war` và các kết quả build khác. Thiết lập thời gian tồn tại của artifact là 1 giờ trước khi bị tự động xóa.
    - chỉ định runner thực thi job là dev-server
    - Định nghĩa điều kiện khi có commit tag  thì job này được chạy:
    - Job sẽ luôn được thực thi khi điều kiện trên thỏa mãn, kể cả khi các job trước đó thất bại.
- ***kết quả và giải thích***
    
    đây là kết quả test
    
    <img width="1548" height="400" alt="image 2" src="https://github.com/user-attachments/assets/9e6f90fc-ef48-48be-8979-0d11b1668ba4" />
    
    - **Tests run: 1**
        
        Có 1 test case được thực thi.
        
    - **Failures: 0**
        
        Không có test nào thất bại do assert sai (logic test đúng).
        
    - **Errors: 0**
        
        Không xảy ra lỗi runtime trong quá trình chạy test (ví dụ: NullPointerException, context load lỗi).
        
    - **Skipped: 0**
        
        Không có test nào bị bỏ qua (không dùng `@Disabled` hoặc điều kiện skip).
        
    - **Time elapsed: 13.939 s**
        
        Tổng thời gian thực thi test là khoảng 13.9 giây.
        
    
    <img width="1552" height="644" alt="image 3" src="https://github.com/user-attachments/assets/a88f4488-1508-4b73-bf0a-1e37728c9db0" />

    
    đây là kết quả build
    
    Quá trình build ứng dụng bằng Maven trong pipeline CI/CD đã được thực hiện thành công. Hệ thống ghi nhận trạng thái *BUILD SUCCESS*, cho thấy mã nguồn được biên dịch, kiểm thử và đóng gói mà không phát sinh lỗi. Tổng thời gian thực thi toàn bộ quá trình build là khoảng 21 giây, phù hợp với một ứng dụng Spring Boot có kiểm thử tự động.
    
    <img width="2249" height="1038" alt="image 4" src="https://github.com/user-attachments/assets/1b34b6f2-48ae-433d-950d-f55b5dc99943" />

    
    Sau khi build hoàn tất, GitLab Runner tiến hành thu thập và upload artifact theo cấu hình. Thư mục `target/` chứa các file kết quả build đã được phát hiện và đóng gói thành archive để lưu trữ trên GitLab. Quá trình upload artifact diễn ra thành công với mã phản hồi 201 (Created), xác nhận artifact đã được lưu trữ đầy đủ và sẵn sàng cho các bước triển khai tiếp theo. Cuối cùng, Runner thực hiện dọn dẹp thư mục làm việc và các biến môi trường tạm thời, kết thúc job với trạng thái *Job succeeded*.
    

Code kiểm thử mã nguồn tĩnh

```yaml

build-sonar:
  stage: build-sonar
  dependencies:
    - build  
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  cache:
    key: "${CI_COMMIT_REF_SLUG}-sonar"
    paths:
      - .sonar/cache
    policy: pull-push
  script:
    - echo "=== Running SonarQube Scan ==="
    - mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=$CI_PROJECT_NAME -Dsonar.projectName="$project_name" -Dsonar.host.url=http://192.168.8.80:8000 -Dsonar.token=$SONAR_TOKEN -Dsonar.java.binaries=target/classes -DskipTests=true
  allow_failure: true
  rules:
    - if: "$CI_COMMIT_TAG"
      when: always
  tags:
    - $runner

```

- ***Giải thích code***
    
    Job `build-sonar` có nhiệm vụ **thực hiện phân tích chất lượng mã nguồn (Static Code Analysis)** bằng **SonarQube** sau khi quá trình build hoàn tất. Job này sử dụng kết quả build từ job `build` để quét mã nguồn và đánh giá các tiêu chí như code smell, bug, vulnerability và technical debt.
    
    job `build-sonar` phụ thuộc vào job `build`. Điều này cho phép job này sử dụng **artifact (thư mục `target/`)** được tạo ra từ job build, đặc biệt là các file `.class` cần thiết cho phân tích SonarQube.
    
    Biến `SONAR_USER_HOME` chỉ định thư mục lưu trữ dữ liệu của SonarQube Scanner trong project, giúp thuận tiện cho việc cache và tăng tốc các lần scan tiếp theo.
    
    - Cache được sử dụng để lưu trữ dữ liệu SonarQube Scanner.
    - `key` được tạo dựa trên branch hoặc tag (`CI_COMMIT_REF_SLUG`).
    - `paths` chỉ định thư mục `.sonar/cache` cần cache.
    - `policy: pull-push` cho phép runner:
        - Tải cache trước khi chạy job
        - Cập nhật cache sau khi job kết thúc
    
    Cơ chế này giúp **giảm thời gian phân tích ở các lần chạy tiếp theo**.
    
    Thực hiện SonarQube scan bằng Maven plugin với các tham số:
    
    - `sonar.projectKey`: khóa định danh duy nhất của project trên SonarQube.
    - `sonar.projectName`: tên hiển thị của project trên giao diện SonarQube.
    - `sonar.host.url`: địa chỉ server SonarQube.
    - `sonar.token`: token xác thực để gửi kết quả phân tích lên SonarQube.
    - `sonar.java.binaries`: đường dẫn tới các file `.class` được build sẵn.
    - `DskipTests=true`: bỏ qua việc chạy test trong job này để tránh trùng lặp và tiết kiệm thời gian.
- ***Kết quả và giải thích***
    
    tiếp theo truy cập vào website của sonarqube: 192.168.8.80:8000 để xem kết quả test mã nguồn
    
    <img width="1797" height="409" alt="image 5" src="https://github.com/user-attachments/assets/f71fb6b3-3842-4db4-9be5-6c1017ef0010" />

    
    ### 1. Security – A (0 issues)
    
    Chỉ số Security đạt mức **A**, cho thấy không phát hiện lỗ hổng bảo mật nào trong mã nguồn. Điều này chứng minh dự án không chứa các pattern nguy hiểm phổ biến như hard-coded credentials, injection hoặc các rủi ro bảo mật nghiêm trọng khác.
    
    ---
    
    ### 2. Reliability – C (16 issues)
    
    Chỉ số Reliability ở mức **C**, với 16 lỗi tiềm ẩn liên quan đến độ ổn định của ứng dụng. Các vấn đề này thường là:
    
    - Khả năng phát sinh lỗi runtime
    - Xử lý exception chưa đầy đủ
    - Logic có thể gây crash trong một số tình huống biên
    
    Mức C cho thấy dự án vẫn chạy được, nhưng cần cải thiện để tăng độ tin cậy khi vận hành thực tế.
    
    ---
    
    ### 3. Maintainability – A (69 code smells)
    
    Chỉ số Maintainability đạt mức **A**, mặc dù SonarQube phát hiện 69 code smells. Đây là các vấn đề liên quan đến:
    
    - Quy ước code
    - Độ dễ đọc
    - Cấu trúc chưa tối ưu
    
    Các code smell này **không gây lỗi ngay**, nhưng nếu không xử lý sẽ làm tăng technical debt trong dài hạn.
    
    ---
    
    ### 4. Security Hotspots Reviewed – E (0.0%)
    
    Chỉ số Security Hotspots Reviewed ở mức **E (0%)**, cho thấy:
    
    - Có các vị trí code nhạy cảm về bảo mật
    - Tuy nhiên chưa được review thủ công trên SonarQube
    
    Điều này không đồng nghĩa với lỗ hổng, nhưng phản ánh rằng việc kiểm tra bảo mật thủ công chưa được thực hiện.
    
    ---
    
    ### 5. Coverage – 0.0%
    
    Coverage bằng **0%**, cho thấy dự án chưa cấu hình hoặc chưa có unit test được đo coverage.
    
    Nguyên nhân phổ biến:
    
    - Chưa tích hợp JaCoCo
    - Test không được include trong quá trình scan SonarQube
    
    Điều này là **điểm cần cải thiện** để đảm bảo chất lượng mã nguồn lâu dài.
    
    ---
    
    ### 6. Duplications – 13.9%
    
    Tỷ lệ trùng lặp mã nguồn là 13.9%, cho thấy có một số đoạn code bị lặp lại giữa các class hoặc module.
    
    Mức này chưa nghiêm trọng, nhưng có thể:
    
    - Gây khó khăn khi bảo trì
    - Làm tăng rủi ro lỗi khi sửa đổi code
    
    ---
    
    <img width="544" height="348" alt="image" src="https://github.com/user-attachments/assets/b5501a77-de0c-4290-8531-d8a8c6ab89d8" />

    
    Chuyển sang tab Issues ta thấy có 71 vấn đề trong đó có 6 vấn đề ở mức độ high, 33 vấn đề ở mức độ Medium, 32 vấn đề ở mức độ low. Đây là mức đánh giá do Sonarqube set sẵn từ trước
    
    lấy ví dụ 1 issue ở mức độ high:
    
    <img width="1415" height="1228" alt="image" src="https://github.com/user-attachments/assets/c693106c-b94f-46a7-a850-60f5436dbab5" />

    
    sonar sẽ chỉ ra issue đó cụ thể ở đâu trong mã nguồn, với ví dụ này thì nó nằm ở trong file src/main/java/com/example/demo/controller/AdminController.java
    
    <img width="1451" height="598" alt="image" src="https://github.com/user-attachments/assets/15372b1a-674a-4014-8ec0-4c0244bdbb09" />

    
    Tab why is this an issue, Sonarqube sẽ nêu ra lý do tại sao nó được xem là 1 vấn đề. Với ví dụ này, Sonarqube cho rằng “Các chuỗi ký tự chuỗi trùng lặp làm cho quá trình tái cấu trúc trở nên phức tạp và dễ xảy ra lỗi, vì bất kỳ thay đổi nào cũng cần được phổ biến trên tất cả các lần xuất hiện.  Ngoại lệ Để tránh tạo ra một số kết quả dương tính giả, các chữ có ít hơn 5 ký tự sẽ bị loại trừ.”
    
    <img width="1077" height="977" alt="image" src="https://github.com/user-attachments/assets/6a76fc29-06b2-40c6-8bc4-28f11e82a57f" />

    
    tab “How can I fix it” Sonarqube cũng đưa ra 1 số gợi ý để sửa khắc phục vấn đề
    
    - **Bước 1:** Khai báo một biến hằng số ở đầu class: `private static final String ACTION_1 = "action1";`.
    - **Bước 2:** Thay thế tất cả các vị trí đang gõ tay `"action1"` bằng biến `ACTION_1`.
    

---

---

Code SCA

```yaml
sca-scan:
  stage: test
  
  image:
    name: aquasec/trivy:latest
    entrypoint: [""] 

  variables:
    GIT_STRATEGY: clone
    SEVERITY: "CRITICAL,HIGH"
    TRIVY_NO_PROGRESS: "true" 
    TRIVY_CACHE_DIR: ".trivycache/" 
    TRIVY_TIMEOUT: "20m"

  script:
    - echo "=== Starting Trivy SCA (Filesystem) Scan ==="
    - 'apk add --no-cache curl && curl -I https://repo.maven.apache.org || echo "Network Warning: Cannot reach Maven Central"'

    - apk add --no-cache curl
    - curl -L -o html.tpl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl

    - trivy fs --timeout 20m --severity $SEVERITY --format template --template "@html.tpl" -o sca_report.html .
    
    - echo "=== Trivy SCA Scan Finished ==="

  allow_failure: true 

  cache:
    paths:
      - .trivycache/

  artifacts:
    when: always
    paths:
      - sca_report.html
    expire_in: 7 days
  
  tags:
    - tools-server
  only:
    - tags
```

Job `sca-scan` này là một mắt xích cực kỳ quan trọng trong quy trình DevSecOps mà bạn đang xây dựng. Nó thực hiện nhiệm vụ Software Composition Analysis (SCA) — tức là phân tích các thành phần phần mềm (thư viện bên thứ ba) để tìm lỗ hổng bảo mật.

trong job này, tôi sẽ sử dụng runner trên tools-server để kích hoạt pipeline line

- ***giải thích code***
    - **image: aquasec/trivy:latest**: Sử dụng công cụ Trivy, một scanner bảo mật cực kỳ mạnh mẽ và phổ biến. entrypoint: [""] giúp ghi đè thiết lập mặc định của image để GitLab có thể chạy các lệnh shell tùy biến.
    - **variables**:
        - **SEVERITY: "CRITICAL,HIGH"**: Thiết lập này rất quan trọng. Nó yêu cầu Trivy chỉ tập trung báo cáo những lỗ hổng ở mức độ Nghiêm trọng và Cao. Điều này giúp bạn tránh bị "ngộp" bởi hàng trăm lỗi nhỏ (Low/Medium) không quá nguy hiểm.
        - **TRIVY_CACHE_DIR: ".trivycache/"**: Chỉ định thư mục lưu trữ cơ sở dữ liệu lỗ hổng (Vulnerability DB). Kết hợp với phần cache bên dưới, nó giúp scanner chạy nhanh hơn ở những lần sau vì không phải tải lại toàn bộ DB từ internet.
    
    trivy fs --timeout 20m --scanners vuln,secret,config --severity $SEVERITY --format template --template "@html.tpl" -o sca_report.html .
    
    Đây là "linh hồn" của job DevSecOps này:
    
    - **fs (Filesystem):** Yêu cầu Trivy quét trực tiếp các tệp tin thô trong thư mục dự án (như pom.xml, package.json, Dockerfile).
    - **-format template --template "@html.tpl":** * Mặc định Trivy xuất ra màn hình đen trắng rất khó đọc.
        - Việc tải file html.tpl ở bước trước và dùng tham số này giúp chuyển đổi dữ liệu thô thành một báo cáo HTML chuyên nghiệp, có màu sắc cảnh báo rõ ràng.
- ***Giải thích kết quả***
    
    Sau khi job hoàn thành sẽ có artifact là 1 file SCA_report.html
    
    <img width="2243" height="434" alt="image" src="https://github.com/user-attachments/assets/8b2fbc05-0183-49ae-9f1d-d9bf24dbf922" />

    
    <img width="2767" height="1503" alt="image" src="https://github.com/user-attachments/assets/860eac73-06e2-4b4a-b661-18fd5edec071" />

    
    Trong file report này Trivy nó sẽ cung cấp cho chúng ta đầy đủ thông tin các dependencies có lỗ hổng bao gồm tên gói thư viện (Package), mã định danh lỗ hổng (Vulnerability ID), mức độ nghiệm trọng (Severity) giúp lập ra danh sách ưu tiên xử lý, phiên bản hiện tại (installed version), phiên bản vá lỗi (fixed version), liên kết tham khảo (links) danh sách các đường dẫn đến báo cáo chi tiết, bài phân tích hoặc các bản tin bảo mật của các nhà phát hành. 
    
    từ đó có căn cứ để yêu cầu team Dev thực hiện "Dependency Update" định kỳ, thay vì chỉ tập trung viết tính năng mới.
    
    các Lỗi Critical là lỗi phải sửa ngay lập tức (Hotfix). Các lỗi HIGH có thể đưa vào kế hoạch sửa trong tuần, còn các lỗi LOW/MEDIUM có thể để sau. Nếu không có report này, bạn sẽ không biết nên bắt đầu từ đâu.
    
    File report này biến những rủi ro vô hình trong code thành các con số và hành động cụ thể. Nó là công cụ giao tiếp hiệu quả giữa đội ngũ Dev (người sửa lỗi) và Ops (người kiểm soát lỗi).
    

---

---

---

trước khi có docker image thì mình phải có docker file. docker file như là bản công thức để tạo ra docker image

Dockerfile

```yaml
FROM maven:3.5.3-jdk-8-alpine AS build

WORKDIR /app
COPY . .

RUN mvn install -DskipTests=true

FROM alpine:3.22.2

RUN adduser -D shoeshop
RUN apk add --no-cache openjdk8

WORKDIR /run

COPY --from=build /app/target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar
RUN chown -R shoeshop:shoeshop /run

USER shoeshop

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar"]
```

- ***giải thích***
    
    ### Giai đoạn 1: Build Stage (Môi trường xây dựng)
    
    Giai đoạn này nhiệm vụ duy nhất là biên dịch mã nguồn Java thành file thực thi (.jar).
    
    - **FROM maven:3.5.3-jdk-8-alpine AS build**:
        - Sử dụng Image Maven bản 3.5.3 chạy trên Java 8 và hệ điều hành Alpine (siêu nhẹ).
        - **AS build**: Đặt tên cho giai đoạn này là "build" để giai đoạn sau có thể gọi lại.
    - **WORKDIR /app**: Tạo và di chuyển vào thư mục /app bên trong container.
    - **COPY . .**: Sao chép toàn bộ mã nguồn từ máy bạn (thư mục hiện tại) vào thư mục /app của container.
    - **RUN mvn install -DskipTests=true**:
        - Chạy lệnh Maven để biên dịch và đóng gói ứng dụng thành file .jar.
        - **DskipTests=true**: Bỏ qua việc chạy Unit Test ở bước này để quá trình build image diễn ra nhanh hơn (vì bạn thường đã chạy test ở job trước đó trong Pipeline rồi).
    
    ---
    
    ### Giai đoạn 2: Runtime Stage (Môi trường chạy ứng dụng)
    
    Sau khi có file .jar, Docker sẽ bỏ toàn bộ "đống đổ nát" (Maven, mã nguồn thô) ở giai đoạn 1 và chỉ giữ lại kết quả cuối cùng.
    
    - **FROM alpine:3.22.2**: Sử dụng một phiên bản Alpine Linux trắng tinh, cực kỳ nhỏ gọn làm nền tảng.
    - **RUN adduser -D shoeshop**:
        - **Bảo mật:** Tạo một người dùng mới tên là shoeshop.
        - Trong DevSecOps, chúng ta **không bao giờ** chạy ứng dụng bằng quyền root để tránh hacker chiếm quyền điều khiển toàn bộ hệ thống nếu ứng dụng bị tấn công.
    - **RUN apk add openjdk8**: Cài đặt môi trường thực thi Java 8 (JRE) trên nền Alpine để có thể chạy được file .jar.
    - **WORKDIR /run**: Thiết lập thư mục làm việc cho ứng dụng là /run.
    - **COPY --from=build /app/target/shoe-ShoppingCart-*.jar ...**:
        - **Phần quan trọng nhất:** Chỉ lấy duy nhất file .jar đã build từ giai đoạn build trước đó sang giai đoạn này.
    - **RUN chown -R shoeshop:shoeshop /run**: Cấp quyền sở hữu thư mục /run cho người dùng shoeshop vừa tạo.
    - **USER shoeshop**: Chuyển sang sử dụng user shoeshop để thực thi các lệnh sau đó. Đây là bước **"Hardening"** (gia cố bảo mật) cho container.
    - **EXPOSE 8080**: Thông báo rằng ứng dụng này sẽ lắng nghe ở cổng 8080.
    - **ENTRYPOINT ["java", "-jar", "/run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar"]**: Lệnh cố định để khởi động ứng dụng Spring Boot khi container được bật lên.

Job build và push docker image lên private registry

```yaml
buildAndPushDockerImage:
  stage: build-and-push-docker-image

  variables:
      GIT_STRATEGY: clone
  
  before_script:
      - echo "${REGISTRY_PASSWORD}" | docker login "${REGISTRY_URL}" -u "${REGISTRY_USER}" --password-stdin

  script:
      - docker build --network=host -t ${DOCKER_IMAGE} . 
      - docker push ${DOCKER_IMAGE}
  only:
      - tags
  tags:
      - $runner
```

- echo "${REGISTRY_PASSWORD}" | docker login "${REGISTRY_URL}" -u "${REGISTRY_USER}" --password-stdin
- **Mục đích**: Đăng nhập vào kho lưu trữ Docker (như Harbor hoặc Docker Hub) để có quyền đẩy (push) Image lên.
- **-password-stdin**: Đây là một phương pháp bảo mật quan trọng. Thay vì truyền mật khẩu trực tiếp trong câu lệnh (có thể bị lộ trong log), mật khẩu được đọc từ luồng dữ liệu chuẩn (stdin).
- **docker build --network=host -t ${DOCKER_IMAGE} .**:
    - **build**: Đọc **Dockerfile** mà bạn vừa soạn thảo để tạo ra Image.
    - **-network=host**: Cho phép quá trình build sử dụng trực tiếp cấu hình mạng của máy chủ Runner. Điều này cực kỳ hữu ích trong môi trường nội bộ (lab) để tránh các lỗi phân giải tên miền (DNS) hoặc khi cần tải các gói phụ thuộc từ các server nội bộ khác.
    - **t ${DOCKER_IMAGE}**: Gắn "nhãn" (tag) cho Image để định danh (ví dụ: 192.168.8.80:8000/shoeshop/app:v1.0).
- **docker push ${DOCKER_IMAGE}**:
    - Tải Image đã build xong từ máy Runner lên **Registry server (Harbor)**.
    - Sau bước này, Image đã sẵn sàng để K3s kéo về và triển khai (Deploy).
- DOCKER_IMAGE dạng như sau: ${REGISTRY_URL}/${REGISTRY_PROJECT}/${CI_PROJECT_NAME}:${CI_COMMIT_TAG}_${CI_COMMIT_SHORT_SHA}

<img width="1874" height="874" alt="image" src="https://github.com/user-attachments/assets/c7507de7-d7ec-4407-bd23-db54194503f4" />


kết quả khi job chạy xong

<img width="1171" height="702" alt="image" src="https://github.com/user-attachments/assets/f2192de9-1ea5-482b-9f3a-79dab023dce2" />


Docker image đã có trên private registry

```yaml
container-scanning:
  stage: container-scanning
 
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  
  variables:
    GIT_STRATEGY: none
    SEVERITY: "CRITICAL,HIGH"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_USERNAME: "$REGISTRY_USER"
    TRIVY_PASSWORD: "$REGISTRY_PASSWORD"

  script:
    - echo "=== Preparing Trivy Template ==="
    - apk add --no-cache curl
    
    - curl -L -o html.tpl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl

    - echo "=== Starting Trivy Container Image Scan ==="
    
    - trivy image --severity $SEVERITY --format template --template "@html.tpl" -o trivy_report.html ${DOCKER_IMAGE}
    - echo "=== Scan Finished ==="

  allow_failure: true

  artifacts:
    when: always
    paths:
      - trivy_report.html
    expire_in: 7 days
  only:
    - tags
  tags:
    - tools-server 
```

- ***Giải thích code***
    
    tương tự như job SCA, chỉ khác là sử dụng chức năng khác của trivy. trivy image nó sẽ tự động kết nối đến Registry dựa trên biến TRIVY_USERNAME, TRIVY_PASSWORD để pull image về thư mục tạm bên trọng container Trivy để phân tích. Trivy sẽ quét các lớp (layers) của image, ví dụ như bản thân hệ điều hành **Alpine 3.22.2** hay gói **openjdk8** mà bạn đã cài trong Dockerfile để tìm lỗ hổng trong hệ điều hành và các gói phần mềm được cài thêm.
    
- ***Giải thích kết quả***
    
    <img width="1602" height="1204" alt="image" src="https://github.com/user-attachments/assets/ba4a96a7-d4b8-4e29-9f1f-58c55f3bf37e" />

    
    bản report này về cấu trúc cũng tương tự như report của job SCA. Nó sẽ chia ra 2 lớp là lớp hệ điều hành và lớp ứng dụng. ở tầng hệ điều hành có 3 lỗ hổng của Gói libpng (thư viện xử lý hình ảnh) trong bản Alpine 3.22.2 của bạn có 3 lỗi HIGH (CVE-2025-64720, CVE-2025-65018, CVE-2025-66293). Báo cáo xác nhận "No Misconfigurations found", nghĩa là cấu hình Dockerfile của bạn (chạy user non-root, không để lộ port thừa...) đã đạt chuẩn an toàn. ở tầng ứng dụng,  Các thư viện phổ biến như jackson-databind và logback đều có nhiều lỗi mức HIGH do sử dụng phiên bản quá cũ (2.10.2 và 1.2.3). Mối đe dọa lớn nhất: org.apache.tomcat.embed:tomcat-embed-core dính mã CVE-2025-24813 mức độ CRITICAL. Đây là lỗi cực kỳ nguy hiểm, có thể cho phép tấn công từ xa.
    

# Kịch bản 2: Kiểm thử động dự án và kiểm tra hiệu năng sau khi triển khai

 

<img width="1815" height="1129" alt="image" src="https://github.com/user-attachments/assets/553312ef-d213-4744-8d16-d3370a6af1bd" />


Sau khi pipeline ở kịch bản 1 sẽ tiếp tục bổ sung thêm bước DAST, trước khi thực hiện được thì phải chạy DAST thì phải deploy dự án bằng cách đơn giản là run Docker container. 

code run container

```yaml
deploy-Docker-container:
  stage: deploy

  variables:
    GIT_STRATEGY: none
  before_script:
    - echo "${REGISTRY_PASSWORD}" | docker login "${REGISTRY_URL}" -u "${REGISTRY_USER}" --password-stdin
  script:
    - docker pull $DOCKER_IMAGE
    - docker rm -f $DOCKER_CONTAINER
    - docker run --name $DOCKER_CONTAINER -dp 8001:8080 $DOCKER_IMAGE
  tags:
    - $runner
  only:
    - tags
```

`DOCKER_CONTAINER: ${CI_PROJECT_NAME}`

các biến registry password, registry url, registry user đã được định nghĩa từ trước trong variable của dự án

<img width="1334" height="1020" alt="image" src="https://github.com/user-attachments/assets/954b63f4-d4e5-43da-b5d2-252969c39b27" />


dự án chạy trên cổng 8001 của build&deploy runner

code run DAST sử dụng công cụ OWASP ZAP - Zed Attack Proxy 

```yaml
ZAP-scan:
  stage: ZAP-scan 
  
  image:
    name: zaproxy/zap-stable
    entrypoint: [""] 
  

  variables:
    TARGET_URL: "http://192.168.8.110:8001"
    GIT_STRATEGY: none
  script:
    - echo "=== Starting OWASP ZAP Baseline Scan ==="

    - mkdir -p /zap/wrk
    - cp -r /zap/wrk/* . || true 
    - zap-baseline.py -t $TARGET_URL -r zap_report.html || true
    
    - cp /zap/wrk/zap_report.html . || true 
    
    - echo "=== ZAP Scan Finished ==="

  allow_failure: true

  artifacts:
    when: always
    paths:
      - zap_report.html
    expire_in: 7 days
  only:
    - tags
  tags:
    - tools-server 
```

- ***Giải thích code***
    
    
    Job `ZAP-scan` dùng OWASP ZAP Baseline Scan để thực hiện Dynamic Application Security Testing (DAST) đối với một ứng dụng web đang chạy tại: http://192.168.8.110:8001
    
    image:
    name: zaproxy/zap-stable
    entrypoint: [""]
    
    - Sử dụng Docker image chính thức của OWASP ZAP
    - `entrypoint: [""]`:
        - Override entrypoint mặc định của image
        - Cho phép GitLab CI thực thi trực tiếp các lệnh trong `script`
        - Nếu không override, container có thể không chạy đúng lệnh `zap-baseline.py`
    
    mkdir -p /zap/wrk
    
    - `/zap/wrk` là working directory mặc định của ZAP
    - ZAP sẽ:
        - Lưu session
        - Lưu report
        - Lưu dữ liệu scan vào thư mục này
    
    cp -r /zap/wrk/* . || true
    
    - Copy nội dung từ `/zap/wrk` ra thư mục làm việc hiện tại
    - `|| true` để:
        - Tránh job fail nếu thư mục trống
    
    `zap-baseline.py -t $TARGET_URL -r zap_report.html || true`
    
    - `zap-baseline.py`:
        - Chạy Passive Scan
        - Không fuzz, không tấn công mạnh
    - `t $TARGET_URL`: URL target
    - `r zap_report.html`: xuất report HTML
    - `|| true`:
        - Không làm job fail dù phát hiện lỗ hổng
        - Phù hợp cho môi trường:
            - CI
            - Dev / Staging
            - Chưa enforce security gate
    
    cp /zap/wrk/zap_report.html . || true
    
    - Đảm bảo `zap_report.html` nằm ở working directory
    - Để GitLab CI collect artifact
    
    `when: always`:
    
    - Dù job fail hay success → vẫn lưu report
    
    `paths`: file report HTML
    
    `expire_in: 7 days`: tự động xóa sau 7 ngày
    
- ***Giải thích kết quả***
    
    <img width="1334" height="1020" alt="image" src="https://github.com/user-attachments/assets/d5360e9a-631a-4724-9f50-2339684c0923" />

    
    qua bản report ta có thể thấy có tổng cộng 10 cảnh bảo trong đó 0 cảnh báo High (lỗ hổng chiếm quyền điều khiển trực tiếp ngay lập tức), 3 cảnh báo medium (ảnh hưởng đến tính toàn vẹn dữ liệu và người dùng), 3 cảnh bảo mức low (các thiếu sót về cáu hình gia cố bảo mật), 4 cảnh bảo mức information (thông tin về kiến trúc ứng dụng)
    
    ### Phân tích các lỗ hổng mức MEDIUM (Ưu tiên xử lý)
    
    Đây là những điểm bạn cần đưa vào báo cáo thực nghiệm để chứng minh giá trị của việc quét DAST:
    
    ### A. Thiếu mã thông báo chống CSRF (Absence of Anti-CSRF Tokens)
    
    - **Vị trí:** Trang /admin/login.
    - **Rủi ro:** Kẻ tấn công có thể lừa người dùng (đã đăng nhập) thực hiện các hành động ngoài ý muốn (như đổi mật khẩu, xóa dữ liệu) thông qua một liên kết độc hại.
    - **Insight cho DevOps:** Đây là lỗi thuộc về mã nguồn hoặc cấu hình Framework (Spring Security). ZAP phát hiện ra rằng các thẻ <form> của bạn không có input chứa token bảo mật.
    
    ### B. Content Security Policy (CSP) Header Not Set
    
    - **Vị trí:** Toàn bộ hệ thống (Systemic).
    - **Rủi ro:** Ứng dụng dễ bị tấn công **XSS (Cross-Site Scripting)**. Nếu hacker chèn được một đoạn mã script độc hại vào trang web, trình duyệt sẽ thực thi nó vì không có chính sách CSP để ngăn chặn.
    - **Insight cho DevOps:** Lỗi này thường được sửa bằng cách cấu hình Web Server (Nginx) hoặc Spring Security để thêm Header Content-Security-Policy vào mọi phản hồi HTTP.
    
    ### C. Thiếu thuộc tính Integrity (Sub Resource Integrity - SRI)
    
    - **Vị trí:** Các liên kết trỏ đến Google Fonts (fonts.googleapis.com).
    - **Rủi ro:** Nếu máy chủ của Google Fonts bị hack và file CSS bị thay đổi bằng mã độc, ứng dụng của bạn sẽ tải mã độc đó về máy người dùng.
    - **Insight cho DevOps:** Bạn cần thêm thuộc tính integrity (mã băm file) vào các thẻ <link> hoặc <script> khi tải tài nguyên từ bên thứ ba.

## triển khai dự án lên K3s trên môi trường production

trong thực nghiệm này tôi triển khai k3s cluster trên 1 server duy nhất vừa là control plane, vừa là worker, application host, ingress endpoint

<img width="402" height="516" alt="image" src="https://github.com/user-attachments/assets/ca895b7e-199d-407c-8c2b-460f3a6c7b88" />


- **Ingress Controller:** Đóng vai trò là cổng giao tiếp duy nhất (Gateway) của hệ thống với mạng bên ngoài. Thành phần này chịu trách nhiệm:
    - Tiếp nhận traffic từ người dùng.
    - Định tuyến (Routing) dựa trên tên miền (Host) hoặc đường dẫn (Path).
- **Service:** Là thành phần trừu tượng hóa giúp đảm bảo tính ổn định của ứng dụng. Service hoạt động như một bộ cân bằng tải nội bộ (Internal Load Balancer), phân phối traffic đến các Pods phù hợp, đảm bảo người dùng không bị gián đoạn dịch vụ ngay cả khi các Pods thay đổi địa chỉ IP.
- **Pods:** Đơn vị triển khai nhỏ nhất trong Kubernetes. Mỗi Pod chứa một container của ứng dụng. Hệ thống được cấu hình chạy nhiều bản sao (Replicas) của Pod để đảm bảo tính sẵn sàng cao (High Availability).

file deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shoeshop-deployment
  labels:
    app: shoeshop
spec:
  replicas: 2 
  selector:
    matchLabels:
      app: shoeshop
  template:
    metadata:
      labels:
        app: shoeshop
    spec:
      imagePullSecrets:
        - name: my-registry-key
      containers:
      - name: shoeshop-container
        image: ${DOCKER_IMAGE}
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

Đây là loại resource deployment, chịu trách nhiệm quản lý vòng đời của ứng dụng. Nó được gán nhãn là shoeshop, nhãn này để các thành phần khác có thể tìm và quản lý. Duy trì song song 2 bản sao ứng dụng để dự phòng, đảm bảo hệ thống không bị gián đoạn nếu 1 container gặp lỗi. imagePullSecrets là khóa xác thực để tải Docker image từ private registry. Định nghĩa tối thiểu 256Mebibyte Ram và 0.2 CPU để chạy và ngưỡng tối đa là 512Mi RAM và 0.5 CPU, ngăn chặn ứng dụng chiếm dụng tài nguyên gây treo toàn bộ hệ thống

file service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shoeshop-service
spec:
  selector:
    app: shoeshop
  ports:
    - protocol: TCP
      port: 80         
      targetPort: 8080 
  type: ClusterIP  
```

- Service đóng vai trò là điểm truy cập cố định (Internal Load Balancer) cho ứng dụng, giúp các thành phần khác không cần quan tâm đến sự thay đổi địa chỉ IP của các Pods.
- **Cơ chế liên kết (`selector`):** Service tự động định tuyến lưu lượng đến các Pods mang nhãn `app: shoeshop`, đảm bảo kết nối thông suốt ngay cả khi Pods được tạo mới hoặc bị xóa bỏ.
- **Ánh xạ cổng (`ports`):**
    - **Port 80:** Cổng tiếp nhận yêu cầu từ nội bộ Cluster (ví dụ từ Ingress).
    - **TargetPort 8080:** Cổng đích thực tế của ứng dụng đang chạy trong Container.
- **Bảo mật (`type: ClusterIP`):** Sử dụng kiểu `ClusterIP` để giới hạn phạm vi truy cập chỉ trong mạng nội bộ Kubernetes, ngăn chặn việc truy cập trực tiếp trái phép từ Internet (bắt buộc traffic phải đi qua lớp bảo vệ Ingress).

file ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shoeshop-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: shoeshop.ducbahpb.io.vn 
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shoeshop-service
            port:
              number: 80

```

- **Vai trò Gateway (Cổng nối):** Đóng vai trò là bộ cân bằng tải lớp ứng dụng (Layer 7 Load Balancer), tiếp nhận và quản lý toàn bộ lưu lượng truy cập HTTP từ Internet vào cụm K3s.
- **Tích hợp Traefik:** Sử dụng annotation `traefik.ingress.kubernetes.io/router.entrypoints: web` để chỉ định Ingress Controller mặc định của K3s (Traefik) lắng nghe và xử lý các yêu cầu tại cổng HTTP (port 80).
- **Định tuyến theo tên miền (Host-based Routing):** Cấu hình quy tắc chỉ cho phép các yêu cầu có `Host: shoeshop.ducbahpb.io.vn` được đi qua, đảm bảo tính chính xác khi chạy nhiều ứng dụng trên cùng một cụm.
- **Điều hướng Backend:** Ánh xạ đường dẫn gốc `/` đến dịch vụ nội bộ `shoeshop-service` tại cổng 80, hoàn tất chuỗi kết nối từ người dùng đến ứng dụng.

---

code job deploy to k3s

```yaml
deploy-to-k3s:
  stage: deploy
  variables:
    GIT_STRATEGY: fetch 
  before_script:
    - export KUBECONFIG=$KUBECONFIG_CONTENT
    - apt-get update && apt-get install -y gettext-base
    - echo "${REGISTRY_PASSWORD}" | docker login "${REGISTRY_URL}" -u "${REGISTRY_USER}" --password-stdin

  script:
    - envsubst '${DOCKER_IMAGE}' < deployment.yaml | kubectl apply -f -
    
    - kubectl apply -f service.yaml
    - kubectl apply -f ingress.yaml

    - kubectl rollout status deployment/shoeshop-deployment --timeout=120s

  tags:
    - $runner
  
  only:
    - tags
  
  when: manual
```

- ***Giải thích code***
    
    Job này chịu trách nhiệm tự động hóa việc triển khai ứng dụng lên cụm K3s. Nó thực hiện các bước: chuẩn bị môi trường, thay thế biến cấu hình động, áp dụng các tài nguyên Kubernetes và kiểm tra trạng thái hoạt động sau khi triển khai.
    
    - **Giai đoạn Chuẩn bị (before_script):**
        - export KUBECONFIG=...: Nạp file cấu hình kết nối (được lưu trong biến môi trường KUBECONFIG_CONTENT) để cấp quyền cho kubectl giao tiếp với cụm K3s.
        - apt-get install gettext-base: Cài đặt công cụ envsubst. Đây là thành phần quan trọng để thực hiện kỹ thuật Dynamic Configuration (thay thế biến trong file YAML).
        - docker login ...: Đăng nhập vào Private Registry để đảm bảo phiên làm việc có quyền truy cập vào kho chứa Image (bước này thường để xác thực quyền trước khi thực hiện các thao tác sâu hơn).
    - **Giai đoạn Triển khai (script):**
        - envsubst '${DOCKER_IMAGE}' < deployment.yaml | kubectl apply -f -:
            - **Cơ chế:** Đọc file deployment.yaml, tìm chuỗi ${DOCKER_IMAGE} và thay thế bằng tên Image thực tế (kèm version/tag) từ biến môi trường.
            - **Thực thi:** Kết quả sau khi thay thế được truyền trực tiếp (pipe) cho K3s để cập nhật Deployment mà không cần tạo file trung gian.
        - kubectl apply -f service.yaml & ingress.yaml: Cập nhật cấu hình mạng nội bộ (Service) và cổng giao tiếp Internet (Ingress) để đảm bảo định tuyến chính xác.
    - **Giai đoạn Kiểm tra Chất lượng (Verification):**
        - kubectl rollout status ...: Đây là bước Quality Gate. Pipeline sẽ không báo "Thành công" ngay lập tức mà sẽ đợi (tối đa 120 giây) để xem các Pods có khởi động và chạy ổn định hay không. Nếu ứng dụng bị lỗi (CrashLoopBackOff), bước này sẽ fail và báo lỗi cho người quản trị.
        
        `when: manual`: Yêu cầu xác nhận thủ công (nút bấm "Play") từ người quản trị. Đây là chốt chặn an toàn cuối cùng để tránh việc vô tình deploy lên môi trường Production.
        

<img width="402" height="516" alt="image" src="https://github.com/user-attachments/assets/0590ee1b-0dfd-4565-b175-5182bdbc0c27" />


đây là hình ảnh shop chạy với tên miền là shoeshop.ducbahpb.io.vn

---

file Stress-test.js

```yaml
import http from 'k6/http';
import { check, sleep } from 'k6';
import { htmlReport } from 'https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '2m', target: 100 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.05'], // cho phép tới 5% lỗi
  },
};

export default function () {
  const res = http.get(__ENV.TARGET_URL);
  check(res, {
    'not 5xx': (r) => r.status < 500,
  });
  sleep(0.5);
};

export function handleSummary(data) {
  return {
    'k6-stress-report.html': htmlReport(data),
  };
}

```

**1. Mục đích tổng quát**
Kịch bản này thực hiện Stress Testing (Kiểm thử sức chịu tải) nhằm đánh giá khả năng chịu đựng của hệ thống khi lượng truy cập tăng cao đột biến. Mục tiêu là tìm ra giới hạn mà tại đó server bắt đầu trả về lỗi hoặc bị sập.

**2. Chiến lược tải (Load Profile - Stages)**
Kịch bản được cấu hình mô phỏng lưu lượng truy cập theo hình thang (Ramp-up/Hold/Ramp-down) với 3 giai đoạn:

- **Giai đoạn 1 (Warm-up):** Trong 1 phút đầu, tăng dần lượng người dùng ảo (Virtual Users - VUs) từ 0 lên 50.
- **Giai đoạn 2 (Stress):** Trong 2 phút tiếp theo, tăng và duy trì áp lực lên tới **100 người dùng ảo** đồng thời truy cập liên tục.
- **Giai đoạn 3 (Cool-down):** Trong 1 phút cuối, giảm dần lượng người dùng về 0 để kết thúc bài test nhẹ nhàng.

**3. Tiêu chí chất lượng (Quality Gate/Thresholds)**
Hệ thống thiết lập ngưỡng chấp nhận lỗi (Thresholds) để quyết định bài test Đạt (Pass) hay Trượt (Fail):

- **`http_req_failed`**: Tỷ lệ yêu cầu thất bại phải nhỏ hơn 5% (`rate<0.05`).
- **Ý nghĩa:** Nếu trong quá trình test, server trả về lỗi quá 5% tổng số request, pipeline CI/CD sẽ đánh dấu bước này là thất bại.

**4. Logic thực thi (Test Execution)**
Trong mỗi lần lặp (iteration) của một người dùng ảo:

- **Target:** Gửi yêu cầu HTTP GET tới địa chỉ URL được quy định trong biến môi trường `__ENV.TARGET_URL` (Giúp linh hoạt thay đổi môi trường test Dev/Staging/Prod mà không cần sửa code).
- **Validation:** Kiểm tra phản hồi (`check`), đảm bảo mã trạng thái (Status Code) nhỏ hơn 500 (tức là không chấp nhận lỗi Server Internal Error 5xx).
- **Pacing:** Nghỉ 0.5 giây (`sleep(0.5)`) giữa các lần gửi request để mô phỏng hành vi thực tế của con người, tránh spam request phi thực tế.

**5. Kết quả đầu ra (Reporting)**

- Kịch bản tích hợp module `k6-reporter` để tự động xuất kết quả dưới dạng file HTML (`k6-stress-report.html`).
- File này cung cấp biểu đồ trực quan về độ trễ (latency), số lượng request/giây (RPS) và tỷ lệ lỗi, thuận tiện cho việc báo cáo và phân tích sau sự cố.

code job stresstest

```yaml
stress-test:
  stage: performance-test
  image:
    name: grafana/k6:latest
    entrypoint: [""]
  variables:
    TARGET_URL: "http://shoeshop.ducbahpb.io.vn"
  script:
    - k6 run scripts/stress.js
  artifacts:
    when: always
    paths:
      - k6-stress-report.html
    expire_in: 7 days
  tags:
    - tools-server
  only:
    - tags
```

đây là kq của job

<img width="1317" height="969" alt="image" src="https://github.com/user-attachments/assets/5dcab6f6-b97e-4967-a1a4-2234ae1b1341" />


### Phân tích kết quả kiểm thử tải (Stress Test Report)

**1. Tổng quan kết quả (Overview)**

- **Tổng số yêu cầu (Total Requests):** 17,290 requests được xử lý trong suốt quá trình test.
- **Trạng thái:** **ĐẠT (PASSED)**.
    - Số yêu cầu thất bại (Failed Requests): **0**.
    - Vi phạm ngưỡng (Breached Thresholds): **0**.
    - Kiểm tra thất bại (Failed Checks): **0**.

**2. Phân tích độ trễ (Latency Analysis - `http_req_duration`)**
Chỉ số này đo lường tổng thời gian từ khi gửi yêu cầu đến khi nhận được phản hồi hoàn chỉnh.

- **Trung bình (AVG):** ~280ms. Đây là tốc độ phản hồi khá tốt cho một ứng dụng web.
- **Độ ổn định (P95):** ~611ms. Nghĩa là 95% số người dùng nhận được phản hồi dưới 0.6 giây. Đây là mức chấp nhận được.
- **Điểm chịu tải cực đại (MAX):** ~4165ms (hơn 4 giây).
    - *Nhận xét:* Có xuất hiện hiện tượng "nghẽn cổ chai" (spike) tại một số thời điểm khi tải lên cao nhất, khiến một vài request bị chậm đi đáng kể (từ 280ms vọt lên 4s). Tuy nhiên, hệ thống **không bị sập** (không có lỗi 500).

**3. Phân tích khả năng xử lý (Throughput & Network)**

- **Thời gian chờ (http_req_waiting):** Trung bình ~238ms. Chỉ số này chiếm phần lớn trong tổng thời gian duration (~280ms), cho thấy thời gian chủ yếu dành cho việc Server xử lý logic (CPU/RAM/Database), chứ không bị nghẽn ở mạng hay quá trình truyền tải dữ liệu (`receiving` chỉ mất ~41ms).
- **Tỷ lệ lỗi (Error Rate):** 0.00% (`http_req_failed`). Hệ thống đảm bảo tính toàn vẹn dữ liệu và tính sẵn sàng 100% trong suốt bài test.

Hệ thống đã vượt qua bài kiểm tra Stress Test với tổng cộng 17,290 yêu cầu mà không phát sinh bất kỳ lỗi nào (Zero Downtime). Thời gian phản hồi trung bình duy trì ở mức tốt (~280ms). Mặc dù có hiện tượng độ trễ tăng cao (lên tới 4s) tại thời điểm tải cực đại, hệ thống vẫn duy trì được kết nối và tự phục hồi mà không gây gián đoạn dịch vụ.

---

---

# Kịch bản 3: Sử dụng zabbix để giám sát hệ thống

Trong mô hình DevSecOps, việc triển khai ứng dụng (Deployment) mới chỉ là bước khởi đầu. Để đảm bảo ứng dụng hoạt động bền bỉ và an toàn, quá trình **Vận hành & Giám sát (Operations & Monitoring)** đóng vai trò then chốt. Kịch bản này tập trung vào việc triển khai giải pháp giám sát tập trung sử dụng **Zabbix** – một công cụ mã nguồn mở mạnh mẽ. Mục tiêu không chỉ dừng lại ở việc theo dõi tài nguyên hệ thống (CPU, RAM) mà còn thiết lập thông báo về telegram cho người quản trị

<img width="2658" height="495" alt="image" src="https://github.com/user-attachments/assets/98086590-59aa-4d6b-a3a1-190c3bfbb8ef" />


hình ảnh các host đã được add vào trong zabbix thông qua công cụ zabbix agent trên mỗi host. Zabbix Agent là một chương trình nhẹ (daemon) được cài đặt trên các máy chủ đích (Target Hosts). Vai trò của nó là thu thập dữ liệu nội tại của hệ điều hành (CPU, RAM, Disk, Network) và gửi về Zabbix Server để xử lý.

Trong kiến trúc giám sát của Zabbix, Item và Trigger là hai thành phần cốt lõi hoạt động tuần tự để chuyển đổi dữ liệu thô thành thông tin quản trị hữu ích. Item đóng vai trò là đơn vị thu thập dữ liệu đầu vào, chịu trách nhiệm định kỳ truy xuất các thông số kỹ thuật cụ thể từ đối tượng giám sát (như % CPU, dung lượng RAM, lưu lượng mạng) thông qua Zabbix Agent. Dữ liệu này sau đó được chuyển đến Trigger – bộ phận đánh giá logic. Trigger chứa các biểu thức điều kiện để so sánh dữ liệu thu được từ Item với các ngưỡng giới hạn (thresholds) đã thiết lập trước, từ đó xác định trạng thái của hệ thống là "Bình thường" (OK) hay "Gặp sự cố" (Problem).

Mối liên hệ với cơ chế cảnh báo:
Hai thành phần này đóng vai trò là điều kiện tiên quyết để cơ chế cảnh báo (Alerting Mechanism) hoạt động. Trong Zabbix, hệ thống cảnh báo không tự động nhìn vào dữ liệu của Item để gửi thông báo; nó chỉ phản ứng dựa trên sự thay đổi trạng thái của Trigger. Cụ thể:

1. Item cung cấp bằng chứng số liệu thực tế.
2. Trigger phân tích bằng chứng đó. Nếu giá trị vượt ngưỡng an toàn, Trigger chuyển trạng thái từ OK sang PROBLEM.
3. Chính sự chuyển đổi trạng thái này là "công tắc" kích hoạt Action (hành động), khiến Zabbix gửi email hoặc tin nhắn Telegram đến người quản trị.

Thực hiện kiểm tra khả năng cảnh báo của zabbix thông qua bài kiểm tra website có hoạt động trên cổng 8001 của dev-server (build&deploy runner) hay không.

Bước 1: thiết lập item cho host dev-server_192.168.8.110

<img width="1634" height="816" alt="image" src="https://github.com/user-attachments/assets/5015b1ce-8675-408c-a4da-726552956c20" />


- thiết lập một Item sử dụng Zabbix Agent với key net.tcp.listen[8001]. Item này sẽ định kỳ 10 giây một lần kiểm tra xem Port 8001 trên máy chủ đích có đang mở hay không. Giá trị trả về là **1** (Dịch vụ hoạt động) hoặc **0** (Dịch vụ ngừng hoạt động). Đây là cơ sở dữ liệu để Trigger kích hoạt cảnh báo khi giá trị này chuyển về 0.

Bước 2: thiết lập trigger ứng với item vừa mới thiết lập

<img width="1590" height="695" alt="image" src="https://github.com/user-attachments/assets/7aba8a2a-5e2d-4ae3-8e5e-b80ced4805ed" />


Để hoàn thiện quy trình giám sát, chúng tôi thiết lập một Trigger với mức độ nghiêm trọng **High**. Trigger này sử dụng hàm last() để liên tục đánh giá dữ liệu từ Item. Ngay khi giá trị trả về bằng **0** (tức là Port 8001 không còn lắng nghe kết nối), Trigger sẽ lập tức kích hoạt trạng thái sự cố và gửi thông báo khẩn cấp đến đội ngũ vận hành, giúp giảm thiểu tối đa thời gian gián đoạn dịch vụ (Downtime).

Bước 3: Dừng chạy container shoeshop trên dev-server và chờ thông báo từ zabbix

```yaml
docker stop shoeshop
```

<img width="1590" height="695" alt="image" src="https://github.com/user-attachments/assets/52058d17-4b43-454c-8f41-2e022927fd0e" />


hình ảnh dịch vụ không chạy trên cổng 8001 của dev-server

<img width="951" height="410" alt="image" src="https://github.com/user-attachments/assets/f45738d7-ce6b-4b21-b9cf-d0354ce61c3b" />


thông báo của zabbix về telegram

<img width="2664" height="245" alt="image" src="https://github.com/user-attachments/assets/93ffdad4-5c6e-4390-93b4-064712dd7744" />


thông báo của zabbix trên tab problems của trang chủ zabbix

Bước 4: mở lại dịch vụ và xem thông báo khôi phục từ zabbix

```yaml
docker start shoeshop
```

<img width="1631" height="1008" alt="image" src="https://github.com/user-attachments/assets/7a095349-4c2e-435c-ba20-27b7148cc3df" />


hình ảnh dịch vụ đã hoạt động trở lại

<img width="925" height="353" alt="image" src="https://github.com/user-attachments/assets/26484f5f-cbed-4900-bd9a-a5573095c4e9" />


hình ảnh thông báo của zabbix về telegram

<img width="2650" height="194" alt="image" src="https://github.com/user-attachments/assets/a0d56393-24a9-4d35-9a9e-d2450a42ca81" />


hình ảnh thông báo của zabbix trên tab problems của trang chủ zabbix

Để tối ưu hóa thời gian triển khai và đảm bảo các chỉ số giám sát tuân thủ theo các tiêu chuẩn vận hành tốt nhất (Best Practices), hệ thống được áp dụng Template "Linux by Zabbix agent" có sẵn. Thay vì cấu hình thủ công từng tham số, Template này cung cấp một bộ quy tắc giám sát toàn diện bao quát các khía cạnh sinh tồn của máy chủ. Cụ thể, về mặt tài nguyên, Zabbix tự động thu thập dữ liệu hiệu năng của CPU (system.cpu.util), dung lượng RAM khả dụng (vm.memory.size) và mức độ tải trung bình (Load Average). Các Trigger tương ứng được thiết lập sẵn để cảnh báo ngay lập tức khi hệ thống rơi vào trạng thái căng thẳng, ví dụ như khi hiệu suất CPU vượt quá 90% trong thời gian dài hoặc khi bộ nhớ RAM trống giảm xuống dưới ngưỡng an toàn (20MB), giúp ngăn chặn kịp thời các sự cố treo dịch vụ (Crash) do thiếu tài nguyên.

<img width="915" height="636" alt="image" src="https://github.com/user-attachments/assets/1b590aa9-ab88-4dfe-b638-94699393594c" />


hình ảnh thông báo vấn đề và khôi phục về mức sử dụng bộ nhớ trên gitlab-server
