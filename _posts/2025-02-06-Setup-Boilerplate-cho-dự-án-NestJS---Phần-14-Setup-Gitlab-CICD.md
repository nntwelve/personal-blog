Đây là bài viết nằm trong Series NestJS thực chiến, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

![](https://images.viblo.asia/ccd78c6a-8886-4271-bd8c-485c47659a31.png)

# Đặt vấn đề 📜
Continuous Integration và Continuous Delivery/Deployment hay CI/CD, một khái niệm mà đi đâu chúng ta cũng nghe thấy trong thời đại tự động hoá ngày nay. Vậy CI/CD là gì? cùng mình tìm hiểu trong bài viết hôm nay nhé.

Nói ngắn gọn và đơn giản nhất có thể thì CI/CD giúp chúng ta rút ngắn thời gian phát triển và triển khai ứng dụng đến người dùng. Nó giúp giải quyết được các vấn đề sau:
- Khi phát triển xong tính năng và cần triển khai lên server test, chúng ta không cần phải tốn công truy cập vào server, pull code/image docker (trường hợp build image ở local và push lên registry), rồi sau đó start lại.
- Team tester cũng không phải tốn nhiều công sức trong việc pull code về rồi chạy các test để đảm bảo tính năng mới không gây ra lỗi.
- ...và rất nhiều lợi ích khác.

Không làm mất thời gian của mọi người nữa, chúng ta hãy bắt đầu luôn nhé.
# Thông tin package 📦️
- Gitlab CI/CD version 17.9
# Thực hành CI 👮
Mình sẽ đi qua từng bước 1 từ version cơ bản nhất đến version tối ưu nhất (theo như kiến thức của mình 😁). Trong trường hợp các bạn cần gấp có thể kéo đến mục "**Nội dung hoàn chỉnh**" bên dưới để tham khảo
## 1. Chi tiết từng bước
### 1.1. Phiên bản 1: Sơ khai
Đây là version cơ bản nhất của chúng ta để đáp ứng được các nhu cầu về test, build và deploy. Ở phiên bản này, chúng ta đơn giản chỉ cần đảm bảo code đã pass các *eslint rule* và *unit test* lẫn *integration test* đối với các branch mà các bạn trong team push lên. Bên cạnh đó, khi cần deploy vào server staging thì chúng ta tiến hành build image docker và đẩy lên registry.
```yml:.gitlab-ci.yml
stages: [test, build, deploy]

test lint: # <=== Kiểm tra các eslint rules có pass hết hay không
  stage: test
  image: node:20.9.0
  script:
    - npm install
    - npm run lint
    - npm run build

unit test: # <=== Kiểm tra các unit test có pass hết hay không
  stage: test
  image: node:20.9.0
  script:
    - npm install
    - npm test auth # <=== Chỉ chạy test cho module auth do các module khác chưa có test

e2e test: # <=== Kiểm tra với e2e test xem có pass hết hay không, ⚠️ở dự án này mình chưa viết e2e test nên tạm thời chỉ in ra
  stage: test
  image: node:20.9.0
  script:
    - echo "E2E test running"

build docker: # <=== Jobs này dùng để build docker image
  stage: build
  image: docker:git
  services: # <=== Để build được docker image chúng ta cần khai báo keyword services với giá trị là docker:dind
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo $CI_COMMIT_BRANCH # <=== Dùng để in ra tên branch, có thể xoá nếu muốn
    - echo $CI_DEFAULT_BRANCH # <=== Dùng để in ra tên branch mặc định, có thể xoá nếu muốn
    - echo $CI_COMMIT_REF_SLUG # <=== Đây là phiên bản đã qua xử lý của tên nhánh, thường dùng để tạo task, VD: deploy/staging => deploy-staging
    - docker build --pull -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG"
  only:
    - /^deploy\/.*$/

# Jobs này dùng để deploy lên server staging
deploy staging: # <=== Tạm thời dùng mock cho in ra với lệnh echo, mình sẽ hướng dẫn việc deploy lên server sau
  stage: deploy
  script:
    - echo "Deploy to staging server"
  only:
    - deploy/staging
    
# Đây là job deploy staging thật
# deploy staging:
#   image: docker:latest
#   stage: deploy
#   timeout: 2h
#   services:
#     - docker:dind
#   before_script:
#     - "which ssh-agent || ( apt install openssh-client -y )"
#     - mkdir -p ~/.ssh
#     - touch ~/.ssh/known_hosts
#     - ssh-keyscan -v -H $STAGING_SERVER_IP >> ~/.ssh/known_hosts
#     - eval $(ssh-agent -s)
#     - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
#     - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
#       ~/.ssh/config'
#     - chmod 700 ~/.ssh
#     - chmod 600 ~/.ssh/known_hosts
#   script:
#     - >-
#       $STAGING_SSH_CMD "cd $STAGING_WORK_DIR && docker compose pull && docker compose up -d &&
#       exit"
```
> ❇️ Vì đây là mục đầu tiên nên mình sẽ giải tích chi tiết nhất có thể để mọi người nắm được cách hoạt động, các mục sau sẽ ngắn gọn hơn.

Giải thích: 
- Các mục như `test lint`, `unit test`, `integration test`,... được gọi là các **Job**, là các công việc chúng ta mong muốn thực hiện.
- Các job sẽ được thực thi theo thứ tự khai báo ở keyword `stages`, như trên thì thứ tự sẽ từ **test => build => deploy**. Mặc định nếu chúng ta không khai báo thì sẽ lấy thứ tự sau **.pre => build => test => deploy => .post**, tuỳ theo nhu cầu mà chúng ta có thể kiểm soát thứ tự với stage.
- Vì hầu hết các lệnh của chúng ta đều cần NodeJS để chạy nên sẽ phải dùng keyword `image` để chỉ định image cần dùng. Với khai báo `image: node:20.9.0` thì khi Job được khởi chạy Gitlab sẽ dùng image đó để khởi tạo môi trường thực thi code cho chúng ta. Quá trình này tương tự khi chúng ta Dockerizing ở phần trước. 
> Nếu để ý các bạn sẽ thấy ở job `e2e test` chỉ dùng in ra dòng text  "E2E test running" nên việc dùng keyword `image` là dư thừa. Khi nào chúng ta viết e2e test và cần chạy lệnh để test thì mới dùng đến `image` có NodeJS
- Keyword `script` sẽ là nơi chúng ta thực thi các lệnh theo nhu cầu. Ví dụ ở job `test lint`, mình sẽ lần lượt thực thi việc cài đặt dependencies với lệnh `npm install`, sau đó kiểm tra code có vi phạm eslint rules hay không bằng lệnh `npm lint`, sau cùng là build thử xem có lỗi build gì không với lệnh `npm build`. 
> Việc chạy các lệnh trên y như cách chúng ta làm trước khi review code của các thành viên trong team, nhưng với CI thay vì làm thủ công thì chúng ta đã có thể tự động hoá và bước trực tiếp đến quá trình review logic code và các yêu cầu liên quan

- Ở job `build docker` chúng ta có keyword `before_script`, giống như tên của nó, các lệnh bên trong sẽ được chạy trước khi chạy các lệnh ở `script`. Trong job trên thì mình dùng để login vào registry của gitlab.
> Registry Gitlab là nơi lưu trữ image của chúng ta tương tự với Docker hub

- Cuối cùng là keyword `only`, chỉ định job này chỉ chạy trên các nhánh được chỉ định. Trong ví dụ của chúng ta thì job `deploy staging` chỉ chạy khi commit branch là `deploy/staging` nếu trên các nhánh khác thì job này sẽ không chạy.
> Chúng ta cũng có thể dùng wildcard config để chỉ định job được chạy, ví dụ ở trên `deploy/*` thì các branch bắt đầu bằng `deploy/` để sẽ trigger job `build docker`
> 
> ⚠️ **Tuy nhiên ở các version Gitlab sau này thì keyword `only` và `except` đã deprecated, Gitlab khuyến khích chúng ta dùng `rules`** ⚠️. 
> 
> Phần tiếp theo chúng ta sẽ tiến hành improve lại.

- Job `deploy staging` cũng hơi dài nên chúng ta sẽ bàn đến sau tạm thời dùng mock

Chúng ta sẽ tiến hành chạy thử ở nhánh bất kỳ bằng cách commit file vừa viết và push lên gitlab là xong, việc còn lại gitlab sẽ tự lo. Chúng ta sẽ vào gitlab và vào mục **Build=>Pipelines**:

![](https://images.viblo.asia/51176e67-0db5-44ea-b8dd-91be11e6ced6.png)

Nội dung hiển thị sẽ có **3 jobs** thuộc Stages **test**, đúng với những gì chúng ta đã quy định ban đầu nếu branh không có chứa prefix **deploy/** hoặc **deploy/staging**

![](https://images.viblo.asia/c568a988-0bf0-47c1-8b3a-579817582ba5.png)

Mọi người có thể xem chi tiết quá trình chạy bằng cách bấm vào tên các job:

![](https://images.viblo.asia/6c1c4b6d-901e-44a0-b483-354303209efe.png)

> Có thể thấy khi các job chạy thì các lệnh ở trong đa số giống với khi chúng ta chạy ở máy local từ việc chuẩn bị môi trường, pull source code từ git và thực thi các lệnh.

Do code chúng ta đang trong trạng thái lý tưởng nên sẽ success dễ dàng, như hình bên dưới là 3 job đều đang thành công:

![](https://images.viblo.asia/66854786-275c-453a-9fba-532207a8a1cd.png)

Trong trường hợp có lỗi thì các bạn sẽ thấy như bên dưới:

![](https://images.viblo.asia/84159d95-1d80-41c2-b25e-4b0db77497cc.png)

Lúc này thì chúng ta sẽ bấm vào job để có thể xem được vấn đề đang gặp là gì sau đó có thể thông báo cho team member❗️ (vì đôi lúc job có thể failed do vấn đề từ các runner chạy job của Gitlab, lát nữa chúng ta sẽ đề cập đến vấn đề đó). Hình bên dưới cho chúng ta thấy lỗi ở bước chạy unit test ở file **modules/users/users.service.spec.ts**

![](https://images.viblo.asia/37e776fb-8a01-4402-9d45-456779bb7eb6.png)


Giờ mình sẽ thử merge nhánh vừa rồi vào **deploy/staging** xem chuyện gì sẽ xảy ra:

![](https://images.viblo.asia/f693b5dd-f098-4ee3-b699-3a2608bfe8b5.png)

Các jobs ở cả 3 stage đều đã chạy, để xác thực xem job `build docker` có chạy đúng yêu cầu hay không chúng ta có thể kiểm tra các lệnh trong job chạy và vào mục **Deploy=>Container Registry** để xem image vừa được tạo:

![](https://images.viblo.asia/894d27ab-35c9-4865-a053-c6afb718e042.png)

> Từ stage `build` trở đi thì đã đến giai đoạn của CD nên nếu có vấn đề thì người xử lý thường là team devops hoặc team leader (nếu team size nhỏ)

### 1.2 Phiên bản 2: Tinh gọn
Ở phần 1 chúng ta chỉ có 1 vài job nên có thể không cần phải để ý nhiều khi viết file **.gitlab-ci.yml**, tuy nhiên đối với các dự án lớn với số lượng jobs khá nhiều như hình dưới thì vấn đề clean code là điều không thể thiếu. 

![image.png](https://images.viblo.asia/5419afea-3c8d-4674-a62d-f9cb202e45f4.png)

Giả sử chúng ta phát triển theo hướng deploy các image production theo tag trên git và deploy thêm server UAT (trường hợp khác IP với server staging) thì chúng ta phải **bổ sung thêm các job sau**:
```yml:.gitlab-ci.yml
...
# Job này dùng build image docker theo git tag
build tag:
  image: docker:git
  stage: build
  variables: # <=== Thay vì lặp lại tên image như ở job build docker, chúng ta có thể tạo biến
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker build --pull -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - tags

# Job này deploy code lên server UAT nên dùng config khác
deploy uat:
  image: docker:latest
  stage: deploy
  timeout: 2h
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $UAT_SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$UAT_SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
  script:
    - >-
      $UAT_SSH_CMD "cd $UAT_WORK_DIR && docker compose pull && docker compose up -d &&
      exit"
```
Chúng ta sẽ cùng điểm qua toàn bộ các vấn đề của file trên:
- Lệnh `npm install` bị lặp lại 3 lần ở các job build, tuy nhiên vấn đề ở đây không phải là duplicate code mà là các lệnh `npm install` sẽ tốn khá nhiều thời gian đặc biệt với các dự án lớn.
- Về bản chất job `build tag` không khác gì so với job `build docker` nhưng bị lặp lại 2 lần. Chưa kể việc sau này nếu cần thêm các version cho các nhánh khác thì chúng ta lại phải viết thêm.
- Job `deploy uat` cũng tương tự như trên, có logic giống với `deploy staging` nhưng chỉ khác về config

Để xử lý các vấn đề trên chúng ta sẽ tinh gọn từng bước như sau:
#### Yêu cầu 1:
Bản chất việc chạy `npm install` là để cài đặt dependency nên chúng ta không thể loại bỏ trực tiếp trong bất cứ job nào. Thay vào đó chúng ta có thể tận dụng việc pipeline có thể chia sẽ resource với nhau thông qua **cache** và sử dụng stage **.pre** để tiến hành tạo cache:
```yml:.gitlab-ci.yml
stages: [test, build, deploy] # Stage .pre và .post mặc định sẽ luôn tồn tại nên chúng ta không cần khai báo ở đây

prepare dependencies:
  stage: .pre
  image: node:20.9.0
  script:
    - npm install
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

test lint:
  stage: test
  image: node:20.9.0
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  script:
    - npm run lint
    - npm run build

unit test:
  stage: test
  image: node:20.9.0
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  script:
    - npm test auth
 ...
```
Giải thích: 
- Như tên gọi của nó, job `prepare dependencies` nhằm mục đích phục vụ cho việc cài đặt dependencies của project và cần được chạy đầu tiên nên sẽ đặt ở stage `.pre`
- Keyword `cache` dùng với mục đích cache lại folder/file được khai báo ở `paths` là `node_modules`.  Các job khác muốn dùng chỉ cần khai báo keyword để tiến hành tải về cache này chứ không cần phải cài đặt lại.
> Theo như tài liệu thì cache sẽ được tải về từ cache server khi job bắt đầu và được upload lên cache server sau khi job kết thúc.

- Để sử dụng cache chúng ta sẽ khai báo tương tự như khi tạo.

![](https://images.viblo.asia/4b9a8fc2-1bb2-4995-83ab-28669d9803c5.png)

Chú ý ở dòng 39 thì chúng ta thấy có dòng **"Uploading cache.zip to https://storage.googleapis.com/gitlab-com-runners-cache/project/65860948/part-13-gitlab-cicd-non_protected"**. Đó chính là bước runner push cache lên cache server.

![](https://images.viblo.asia/d0cdb4c7-470c-48bd-9352-9eb48e2fc0ce.png)

 Với job `test lint` dòng 18 biểu thị cho chúng ta thấy cache được tải về với message **"Downloading cache from https://storage.googleapis.com/gitlab-com-runners-cache/project/65860948/part-13-gitlab-cicd-non_protected  ETag="72de7e178d5edff0a2a0a0c0b878c2eb"**
 
 #### Yêu cầu 2:
 Chúng ta sẽ tìm cách gôm gọn việc build các job docker lại thành 1 job duy nhất mà bao quát được hết các trường hợp, chúng ta có thể dùng keyword `if` để khác phục vấn đề này
 ```yml:.gitlab-ci.yml
 ...
 # Chúng ta sẽ gôm tất cả lại chung 1 job build docker duy nhất này
 build docker:
  stage: build
  image: docker:git
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo $CI_COMMIT_REF_SLUG 
    - echo $CI_COMMIT_BRANCH
    - echo $CI_DEFAULT_BRANCH 
    - | # <=== Dùng cho các script xuống hàng
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  only:
    - /^deploy\/.*$/
    - main
 ...
 ```
 Giải thích: 
 - Trường hợp cần chạy script dài như `if` ở trên hoặc install hàng loạt package, chúng ta có thể dùng `|`. Bên cạnh đó chúng ta dùng biến tag để đặt tên cho tag docker, giá trị của biến này sẽ phụ thuộc:
     - `$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH`: điều kiện này biểu thị nếu commit ở nhánh default (thường là main) sẽ build ra tag **latest**
     - Các trường hợp còn lại thì sẽ build ra tag dựa theo tên nhánh hoặc tag và được xử lý thay thế một số ký tự đặc biệt có sẵn ở `$CI_COMMIT_REF_SLUG`
 - Chúng ta cũng sẽ bổ sung nhánh **main** vào keyword `only` để cho trường hợp build tag latest như đã đề cập

Thử push code lên để kiểm tra xem nhánh không nằm trong `only` có chạy job `build docker` hay không, sau đó tạo thêm tag để kiểm chúng trường hợp nếu là tag tạo từ branch không nằm trong `only` thì job có chạy hay không:

![](https://images.viblo.asia/f4f07f31-ad5d-4cd5-9281-370c76d1ead8.png)

Tương tự thử cho nhánh **deploy/staging** và **main** cũng cho kết quả như chúng ta mong đợi

![](https://images.viblo.asia/37188922-e6b9-44ab-bd67-f9e255fcf86a.png)

#### Yêu cầu 3
Job `deploy uat` cũng tương tự như trên, có logic giống với `deploy staging` nhưng chỉ khác về config, do đó bước đầu tiên mình sẽ dùng keyword `variables` để cố gắng đưa 2 job đó về 1 thể thống nhất như sau:
```yml:.gitlab-ci.yml
...
deploy staging:
  image: docker:latest
  stage: deploy
  timeout: 2h
  variables: # <==== Đổi về dùng chung các biến bên dưới
    SERVER_IP: $STAGING_SERVER_IP
    SSH_KEY: $STAGING_SSH_KEY
    SSH_CMD: $STAGING_SSH_CMD
    WORK_DIR: $STAGING_WORK_DIR
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
  script:
    - >-
      $SSH_CMD "cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"

deploy uat:
  image: docker:latest
  stage: deploy
  timeout: 2h
  variables:
    SERVER_IP: $UAT_SERVER_IP
    SSH_KEY: $UAT_SSH_KEY
    SSH_CMD: $UAT_SSH_CMD
    WORK_DIR: $UAT_WORK_DIR
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
  script:
    - >-
      $SSH_CMD "cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"
```
Đến đây thì chúng ta thấy có thể gôm nhóm phần `beforce_script` và `script` lại được do chúng không khác gì nhau. Lần này đến keyword `extends` ra tay:
```yml:.gitlab-ci.yml
.deploy:
  image: docker:latest
  stage: deploy
  timeout: 2h
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
  script:
    - >-
      $SSH_CMD "cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"

deploy staging:
  extends: .deploy
  variables:
    SERVER_IP: $STAGING_SERVER_IP
    SSH_KEY: $STAGING_SSH_KEY
    SSH_CMD: $STAGING_SSH_CMD
    WORK_DIR: $STAGING_WORK_DIR
  

deploy uat:
  extends: .deploy
  variables:
    SERVER_IP: $UAT_SERVER_IP
    SSH_KEY: $UAT_SSH_KEY
    SSH_CMD: $UAT_SSH_CMD
    WORK_DIR: $UAT_WORK_DIR
```
Giải thích:
- Khi tên job bắt đầu bằng dấu `.` (chấm) thì nó được gọi là **Hidden job**, job này sẽ không được chạy khi pipeline run. Ở đây chúng ta dùng nó để khai báo các nội dung chung
- Ở 2 job `deploy_staging` và `deploy_uat` chúng ta dùng `extends` để kế thừa lại toàn bộ nội dung được khai báo ở `.deploy`.

### 1.3. Phiên bản 3: Cải thiện
Khi xong phần 2 thì file của chúng ta đã đủ dùng cho các nhu cầu cơ bản 🫡, nhưng nếu muốn trông xịn xò hơn thì mình giới thiệu với các bạn thêm 3 keyword là `include`, `reference` và `rules` 
#### include
Hiện tại file **.gitlab-ci.yml** đang có 7 job và độ dài khoảng 100 line, bằng việc áp dụng `include` chúng ta có thể tách file này ra thành các file riêng rẻ để phục vụ cho các mục đích khác nhau. 

Ví dụ ở đây từ file chính mình sẽ tách ra thêm 1 file là **pipeline-template.yml** phục vụ cho việc deploy như sau:
```yml:.gitlab/.pipeline-template.yml
.deploy:
  image: docker:latest
  stage: deploy
  timeout: 2h
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
  script:
    - >-
      $SSH_CMD "cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"
```
Sau khi tách xong, ở file **.gitlab-ci.yml** chúng ta chỉ cần dùng `include` để import vào:
```yml:.gitlab-ci.yml
include:
  - local: .gitlab/.pipeline-template.yml
...
```
Vậy là xong, file của chúng ta đã gọn đi khoảng 20 dòng so với trước đó. Trên thực tế mọi người khi dùng sẽ có nhiều phần có thể đem sang file khác để hạn chế được việc file **.gitlab-ci.yml** quá dài.
#### reference
Với `reference` chúng ta có thể tối ưu hoá việc kế thừa nội dung từ job khác so với `extends`. Lần này mình sẽ lại chia ra thêm 1 file nữa để phục vụ cho việc lưu các nội dung liên quan tới các môi trường staging, uat, prod.
```yml:.gitlab/.pipeline-environment.yml
.staging:
  variables:
    SERVER_IP: STAGING_SERVER_IP
    SSH_KEY: STAGING_SSH_KEY
    SSH_CMD: STAGING_SSH_CMD
    WORK_DIR: STAGING_WORK_DIR
  script:
    - echo "Running default staging script"

.uat:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
  script:
    - echo "Running default uat script"

.prod:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
  script:
    - echo "Running default prod script"
```
Giả sử mỗi môi trường chúng ta sẽ có các biến riêng và cách chạy script mặc định, và ở  **.gitlab-ci.yml** tạm thời chúng ta không muốn chạy các script mặc định đó mà chỉ muốn lấy cấu hình biến môi trường thôi thì sẽ chỉnh lại như sau:
```yml:.gitlab-ci.yml
include:
  - local: .gitlab/.pipeline-template.yml
  - local: .gitlab/.pipeline-environment.yml
  ...
deploy staging:
  extends: .deploy
  variables: !reference [.staging, variables]

deploy uat:
  extends: .deploy
  variables: !reference [.uat, variables]
```
File của chúng ta không chỉ rõ ràng hơn mà còn ngắn hơn thêm 1 chút nữa so với trước đó. `reference` cũng có thể áp dụng với các keyword khác để tận dụng các nội dung có sẵn, giúp chúng ta tiện lợi hơn rất nhiều trong quá trình phát triển.

#### rules
Như đã nói ở trên, 2 keyword `only` và `except` đã deprecate nên chúng ta cần dùng `rules` để thay thế tránh trường hợp sau này Gitlab không còn support nữa. Giải thích đơn giản thì rules là 1 tập các điều kiện, nếu pass thì pipeline sẽ chạy job đó còn không thì sẽ bỏ qua.

Quay lại với job `build docker`, mong muốn của chúng ta là muốn job này chỉ chạy khi nằm ở các branch được chỉ định:
```yml:.gitlab-ci.yml
...
build docker:
  stage: build
  image: docker:git
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - | # <=== Dùng cho các script xuống hàng
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  only:
    - /^deploy\/.*$/
    - main
    - tags
```
Với `rules` chúng ta sẽ đổi lại như sau:
```yml:.gitlab-ci.yml
...
build docker:
  stage: build
  image: docker:git
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - | # <=== Dùng cho các script xuống hàng
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  rules: # <=== Thay `only` với `rules`
    - if: $CI_COMMIT_BRANCH =~ /^deploy\/.*$/
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

deploy staging:
  extends: .deploy
  variables: !reference [.staging, variables]
  rules:
    - !reference [.staging, rules] # <=== Đây là cách dùng cho trường hợp chúng ta
    # muốn thêm các rules khác nữa
deploy uat:
  extends: .deploy
  variables: !reference [.uat, variables]
  rules:
    - !reference [.staging, rules]
```
Nhớ thêm `rules` vào file **pipeline-enviroment.yml** nữa nhé mọi người, trong thực tế thì chúng ta thường chỉ chạy trên những branch đã quy định trước:
```yml:.gitlab/pipeline-enviroment.yml
.staging:
  variables:
    SERVER_IP: STAGING_SERVER_IP
    SSH_KEY: STAGING_SSH_KEY
    SSH_CMD: STAGING_SSH_CMD
    WORK_DIR: STAGING_WORK_DIR
    BRANCH_NAME: deploy/staging
  script:
    - echo "Running default staging script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME

.uat:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
    BRANCH_NAME: deploy/uat
  script:
    - echo "Running default uat script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME
...
```

Thử kiểm chứng bằng cách push code lên với branch nảy giờ chúng ta dev thì thấy pipeline không kích hoạt job `build docker`:
![](https://images.viblo.asia/c6fb5a05-649e-4662-b572-bfacbdadc41c.png)

Giờ thì thử với các branch  `deploy/staging`, `main` và tạo tag:
![](https://images.viblo.asia/f027302c-4e01-4548-af34-e4b5e3165353.png)

Mọi thứ vẫn hoạt động đúng như những gì chúng ta dự tính. Keyword `rules` vẫn còn rất nhiều option hay mọi người có thể tham khảo và tận dụng [ở đây](https://archives.docs.gitlab.com/16.11/ee/ci/yaml/index.html#rules) 
### 1.4. Phiên bản 4: Tối ưu
Đến đây theo mình thì mọi thứ đã khá clean rồi, vấn đề cuối cùng chúng ta cần quan tâm là tối ưu thời gian chạy của runner cũng như hạn chế chạy những job không liên quan. 
#### Dùng `cache:policy` để tối ưu quá trình push/pull cache
![](https://images.viblo.asia/0c18529c-0144-4228-bc0d-9452bff3ab95.png)
![](https://images.viblo.asia/8631e1f0-54a2-4ec7-915c-732b28d52c40.png)

Chúng ta cùng nhìn vào 2 hình trên:
- Ở job `prepare dependencies` trước khi chạy nó sẽ tiến hành tải cache về (chỗ mình highlight) và thời gian hoàn thành là 54 giây
- Ở job `test lint` sau khi chạy xong nó tiến hành upload cache lên cache server (Line 31) và thời gian hoàn thành là 1 phút 6 giây

Vấn đề ở đây mình muốn nói đến là việc chúng ta dùng cache chưa hiệu quả vì ở job `prepare` thì chúng ta luôn install lại để tạo ra `node_modules` nên việc pull cache về là vô nghĩa. Tương tự ở job `test lint` chúng ta cũng chỉ dùng không chỉnh sửa gì `node_modules` nên việc push cache lên cũng không mang lại giá trị gì mà chỉ gây mất thêm thời gian.

Để khắc phục vấn đề đó, chúng ta có thể tận dụng `cache:policy` với 2 giá trị là `push` và `pull` như sau:
```yml:.gitlab-ci.yml
...
prepare dependencies:
  stage: .pre
  image: node:20.9.0
  script:
    - npm install
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: push # <=== Thêm vào đây 🟢

test lint:
  stage: test
  image: node:20.9.0
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull # <=== Thêm vào đây 🟢
  script:
    - npm run lint
    - npm run build

unit test:
  stage: test
  image: node:20.9.0
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull # <=== Thêm vào đây 🟢
  script:
    - npm test auth 
...
```
Kết quả sau khi run lại thì job `test lint` chỉ còn 54 giây đã nhanh hơn so với ban đầu 12 giây (tính ra thì tầm 20%), nhìn vào nội dung job các bạn cũng thấy Gitlab sẽ báo là **Not uploading cache due to policy**
![](https://images.viblo.asia/29d96747-b9a2-4d37-9083-4843006c6177.png)

Tuy nhiên đối với job `prepare dependency` thì ngược lại tăng lên 1 phút 9 giây 😱, mặc dù Gitlab đã bỏ qua quá trình pull cache về:

![](https://images.viblo.asia/86abe169-bafd-4d6b-a9a1-b9be4beefafb.png)

> Nguyên nhân là do khi chúng ta chạy lệnh `npm install` nếu như có sẵn `node_modules` và dependency nào đã tồn tại trùng khớp với file **package.json** thì quá trình tải các dependency đó sẽ được skip. Do chúng ta không tải cache về nên buộc npm phải tải lại toàn bộ, đó là lý do tại sao pipeline bị chậm hơn.
>
> Do đó chúng ta cần cẩn trọng khi dùng `push policy` vào job các job install dependencies các bạn nhé ⚠️. Mình sẽ chỉnh lại job `prepare dependencies` như ban đầu.

#### Dùng `workflow:auto_cancel:on_new_commit` để tối ưu khi có commit mới
Có một vấn đề mà đôi lúc chúng ta sẽ gặp trong quá trình quản lý pipeline, ví dụ các bạn push code lên và pipeline run, tuy nhiên bạn nhớ ra là bạn bị lỗi chỗ nào đó nên lại commit và push code thêm lần nữa. Lúc này sẽ có 2 pipeline chạy đồng thời dẫn đến lãng phí tài nguyên không cần thiết.

![](https://images.viblo.asia/a67d2ba6-33ef-407d-b50d-528bbbdc3c1d.png)

Pipeline trước đó đáng ra nên stop và chỉ cần chạy pipeline sau khi chúng ta đã push code chỉnh sửa lên. Để khắc phục tình trạng này, `workflow` là keyword chúng ta cần. Cách dùng như sau:
```yml:.gitlab-ci.yml
...
workflow:
  auto_cancel:
    on_new_commit: interruptible

prepare dependencies:
  stage: .pre
  image: node:20.9.0
  interruptible: true
  ...

test lint:
  stage: test
  image: node:20.9.0
  interruptible: true
  ...

unit test:
  stage: test
  image: node:20.9.0
  interruptible: true
  ...

e2e test:
  stage: test
  image: node:20.9.0
  interruptible: true
  ...

build docker:
  stage: build
  image: docker:git
  interruptible: true
  ...
```
Giải thích:
- Chúng ta sẽ khai báo keyword `workflow` ở global và dùng `on_new_commit` để biểu thị khi có commit mới thì sẽ dừng job các job có gán keyword `interruptible`
- Các job nào cần được dừng thì chúng ta sẽ để keyword `interruptible` với giá trị là `true`

Thử commit code và lặp lại quá trình trên mọi người sẽ thấy sự khác biệt. Các job trước đó ngay lập tức bị cancelling và các job chưa chạy thì sẽ bị cancel
![](https://images.viblo.asia/294e44cd-8dbd-4afc-af0b-699b918ff31a.png)

> ⚠️ **Tuy nhiên mọi người cần cẩn trọng trọng việc dùng keyword `interruptible` đối với các branch liên quan tới deploy, đặc biệt là quá trình deploy production vì có thể dẫn đến dữ liệu bị deploy 1 phần.**
#### Dùng `rules:changes` để tối ưu chỉ chạy khi có những thay đổi nhất định trong các file
Mục tối ưu cuối cùng mình muốn gửi tới mọi người sẽ lại liên quan đến keyword `rules` và lần này là option `changes`. Trong quá trình dev, đôi khi có một số thay đổi ở một số file mà chúng ta không cần phải chạy job. 

Ví dụ trong source của chúng ta nếu chỉ chỉnh sửa nội dung file **README.md** hoặc `LICENSE` thì chúng ta đâu nhất thiết phải chạy job `build docker`. Do đó mình sẽ dùng `rules:changes` để tối ưu vấn đề này:
```yml:.gitlab-ci.yml
...
build docker:
  stage: build
  image: docker:git
  interruptible: true
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo $CI_COMMIT_REF_SLUG # <=== Dùng để in
    - echo $CI_COMMIT_BRANCH # <=== Dùng để in ra tên branch
    - echo $CI_DEFAULT_BRANCH # <=== Dùng để in ra tên branch mặc định
    - | # <=== Dùng cho các script xuống hàng
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  rules:
    # Gôm 3 điều kiện lại với nhau để tránh lặp code
    - if: $CI_COMMIT_BRANCH =~ /^deploy\/.*$/ || $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH || $CI_COMMIT_TAG
      changes: # <=== Thêm vào đây 🟢
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json
 ...
```
```yml:.gitlab-ci.yml
.staging:
  variables:
    SERVER_IP: STAGING_SERVER_IP
    SSH_KEY: STAGING_SSH_KEY
    SSH_CMD: STAGING_SSH_CMD
    WORK_DIR: STAGING_WORK_DIR
    BRANCH_NAME: deploy/staging
  script:
    - echo "Running default staging script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME
      changes: # <=== Thêm vào đây luôn nhé 🟢
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json

.uat:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
    BRANCH_NAME: deploy/uat
  script:
    - echo "Running default uat script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME
      changes: # <=== Thêm vào đây luôn nhé 🟢
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json
 ...
```
Giải thích: chỉ khi nào có sự thay đổi ở các file **Dockerfile, package-lock.json, package.json, .env, nest-cli.json** và toàn bộ các file bên trong folder **src** thì chúng ta mới chạy job này.

![](https://images.viblo.asia/07b5a3f3-63bc-4f5c-ada3-d703584a0af9.png)

Có thể thấy được khi chúng ta cập nhật nội dung file **README.md** thì pipeline sẽ bỏ qua job `build docker` mặc dù đang ở branch `deploy/staging`. Còn khi update nội dung file nằm trong folder **src** thì các job đ

Đến đây thì chúng ta đã hoàn thành xong quá trình tối ưu, mọi người cùng xem lại toàn bộ code ở phần Nội dung hoàn chỉnh nhé
## 2. Nội dung hoàn chỉnh
```.gitlab/.pipeline-environment.yml
.staging:
  variables:
    SERVER_IP: STAGING_SERVER_IP
    SSH_KEY: STAGING_SSH_KEY
    SSH_CMD: STAGING_SSH_CMD
    WORK_DIR: STAGING_WORK_DIR
    BRANCH_NAME: deploy/staging
  script:
    - echo "Running default staging script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME
      changes:
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json

.uat:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
    BRANCH_NAME: deploy/uat
  script:
    - echo "Running default uat script"
  rules:
    - if: $CI_COMMIT_BRANCH == $BRANCH_NAME
      changes:
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json

.prod:
  variables:
    SERVER_IP: UAT_SERVER_IP
    SSH_KEY: UAT_SSH_KEY
    SSH_CMD: UAT_SSH_CMD
    WORK_DIR: UAT_WORK_DIR
  script:
    - echo "Running default prod script"
```

```.gitlab/.pipeline-template.yml
.deploy:
  image: docker:latest
  stage: deploy
  timeout: 2h
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )"
    - mkdir -p ~/.ssh
    - touch ~/.ssh/known_hosts
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add -
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" >
      ~/.ssh/config'
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts
    - echo $CI_REGISTRY_USER $CI_JOB_TOKEN
  script:
    - >-
      $SSH_CMD "docker login registry.gitlab.com -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN && cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"
```

```.gitlab-ci.yml
include:
  - local: .gitlab/.pipeline-template.yml
  - local: .gitlab/.pipeline-environment.yml

stages: [test, build, deploy]

workflow:
  auto_cancel:
    on_new_commit: interruptible

prepare dependencies:
  stage: .pre
  image: node:20.9.0
  interruptible: true
  script:
    - npm install
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

test lint:
  stage: test
  image: node:20.9.0
  interruptible: true
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull
  script:
    - npm run lint
    - npm run build

unit test:
  stage: test
  image: node:20.9.0
  interruptible: true
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
    policy: pull
  script:
    - npm test auth # <=== Chỉ chạy test cho module auth do các module khác chưa có test

e2e test:
  stage: test
  image: node:20.9.0
  interruptible: true
  script:
    - echo "E2E test running"

build docker:
  stage: build
  image: docker:git
  interruptible: true
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - echo $CI_COMMIT_REF_SLUG # <=== Dùng để in
    - echo $CI_COMMIT_BRANCH # <=== Dùng để in ra tên branch
    - echo $CI_DEFAULT_BRANCH # <=== Dùng để in ra tên branch mặc định
    - | # <=== Dùng cho các script xuống hàng
      if [[ "$CI_COMMIT_BRANCH" == "$CI_DEFAULT_BRANCH" ]]; then
        tag=""
        echo "Running on default branch '$CI_DEFAULT_BRANCH': tag = 'latest'"
      else
        tag=":$CI_COMMIT_REF_SLUG"
        echo "Running on branch '$CI_COMMIT_BRANCH': tag = $tag"
      fi
    - docker build --pull -t "$CI_REGISTRY_IMAGE${tag}" .
    - docker push "$CI_REGISTRY_IMAGE${tag}"
  rules:
    - if: $CI_COMMIT_TAG || $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH || $CI_COMMIT_BRANCH =~ /^deploy\/.*$/
      changes:
        - Dockerfile
        - package-lock.json
        - package.json
        - src/**/*
        - .env
        - nest-cli.json

deploy staging:
  extends: .deploy
  variables: !reference [.staging, variables]
  rules:
    - !reference [.staging, rules]
deploy uat:
  extends: .deploy
  variables: !reference [.uat, variables]
  rules:
    - !reference [.uat, rules]
```
# Thực hành CD 🚀
Quay lại với file `.pipeline-template.yml`, đây sẽ là nơi chứa logic deploy code lên service của chúng ta:
```yml:.gitlab/.pipeline-template.yml
.deploy:
  image: docker:latest
  stage: deploy
  timeout: 2h
  services:
    - docker:dind
  before_script:
    - "which ssh-agent || ( apt install openssh-client -y )" # Kiểm tra ssh-agent có chưa, nếu chưa thì tải về
    - mkdir -p ~/.ssh # Đảm bảo sự tồn tại của folder .ssh
    - touch ~/.ssh/known_hosts # Tương tự ở trên tạo file known_hosts nếu chưa tồn tại, dùng để lưu trữ SSH key
    - ssh-keyscan -v -H $SERVER_IP >> ~/.ssh/known_hosts # Quét và trả về public key của server, dùng tránh SSH warning unknown host
    - eval $(ssh-agent -s)
    - echo "$SSH_KEY" | tr -d '\r' | ssh-add - # Lấy SSH key từ config và truyền vào ssh-agent
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > 
      ~/.ssh/config' # Do chúng ta CICD với docker nên cần thêm tStrictHostKeyChecking vào file config tránh SSH thực hiện host verification
    - chmod 700 ~/.ssh
    - chmod 600 ~/.ssh/known_hosts 
  script:
    - >-
      $SSH_CMD "cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit" # Thực hiện lệnh SSH vào server -> pull docker image -> restart lại docker compose -> thoát 
```
Giải thích:
- Ở đây chúng ta sẽ dùng service `docker:dind` làm môi trường cho việc deploy
- Đoạn code bên trong `before_script` đơn giản giúp chúng ta kiểm tra xem SSH client có available hay không, sau đó config kết nối để có thể kết nối đến server.
- Sau khi chuẩn bị môi trường và ssh xong thì chúng ta chỉ cần chạy lệnh đã cấu hình sẵn để truy cập server, pull image, restart server. Các lệnh ở bước này giống y như chúng ta thao tác thủ công, bình thường mọi người làm sau thì cứ truyền vào đây như vậy 😁.

Vậy các biến `$WORK_DIR`, `$SERVER_IP`, `$SSH_KEY` sẽ lấy ở đâu và truyền vào những giá trị gì, chúng ta sẽ vào **Settings => CI/CD**
![](https://images.viblo.asia/07dbee5a-88a4-48d6-b02c-32c32d15eba8.png)

Sau đó chọn expand mục **Variables** và bấm vào **Add variable**

![](https://images.viblo.asia/4bee6935-1234-4fc3-be4a-640343a62160.png)

Chúng ta chỉ cần điền tên các biến ở **Key** và giá trị ở **Value**, ví dụ các biến của mình sẽ setup:
- `$SERVER_IP`: nhập IP của server các bạn vào, ví dụ: `142.250.198.238`
- `$WORK_DIR`: thư mục chứa file docker compose trên server, ví dụ: `home/demo/nestjs-project`
- `$SSH_KEY`: Private key của SSH, cái này các bạn tạo SSH key xong copy nội dung Private key bỏ vào ô Value là được
- `$SSH_CMD`: Lệnh để ssh vào server, ví dụ: `ssh demo@142.250.198.238`

Sau khi đã cấu hình ở Gitlab xong chúng ta sẽ vào server để cấu hình service. Truy cập vào server và folder bạn muốn chứa service (lưu ý phải trùng với cấu hình ở `$WORK_DIR` nhé), vào tạo các cấu hình cơ bản để chạy service như file **docker-compose** và **.env**. Ví dụ:
```docker-compose.yml
services:
  flash_cards_api_staging:
    container_name: flash_cards_api_staging
    image: registry.gitlab.com/nestjs-side-projects/cicd-tutorial:deploy-staging
    env_file:
      - ./.env
    ports:
      - ${PORT}:${PORT}
    depends_on:
      - flash_cards_mongodb
    command: npm run start:prod
    restart: unless-stopped

  flash_cards_mongodb:
    container_name: ${DATABASE_HOST}
    image: mongo:latest
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${DATABASE_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${DATABASE_PASSWORD}
    expose:
      - ${DATABASE_PORT}
    volumes:
      - ./mongo-data:/data/db
    restart: unless-stopped
```

Việc cuối cùng là copy Public key của SSH key chúng ta vừa tạo vào folder `.ssh` trên server. Mục đích để khi Gitlab CICD thực hiện job deploy có thể kết nối đến server.
Sau khi đã copy xong giờ chúng ta thử chạy lại CICD ở nhánh deploy/staging nhé.

![](https://images.viblo.asia/191276f8-cffa-4e9d-b0a4-1fb9cf5896d4.png)

Nhưng đến đây có thể một số bạn sẽ gặp lỗi dẫn đến Pipeline bị fail, một trong những nguyên nhân có thể do khi tạo các biến chúng ta tích vào option `Protect variable`, với option đó thì các bạn cần làm thêm một bước là truy cập vào **Setting=>Repository=>Projected branches**. Nếu chưa có thì các bạn tiến hành bấm nút `Add protected branch` và thêm vào tên branch.
> Có thể dùng wildcard để áp dụng cho nhiều branch như: `*-stable` hoặc `production/*`.

![](https://images.viblo.asia/077bc9fb-fa2c-47a5-9d69-20e243e061d0.png)

Tuy nhiên nếu repository của các bạn là private thì khi run pipeline sẽ gặp lỗi như bên dưới, nguyên nhân là do ở server chạy code không có quyền pull private image vừa build.

![](https://images.viblo.asia/735c1d9f-289a-4b98-974b-dcb267440775.png)

Để khắc phục chúng ta cần thực hiện thêm 4 bước: 
- Bước 1: Tạo Access Token, các bạn bấm vào **Avatar => Edit Profile => Access tokens**.  

![](https://images.viblo.asia/42ac20a7-4020-49d5-a23d-33d96d61c363.png)

Bấm vào nút **Add new token**, nhập vào Token name và check vào `read_registry` để biểu thị rằng token này có thể pull private image từ gitlab registry. Bấm nút C**reate personal access token** vào copy lại token.

![](https://images.viblo.asia/776f68f8-ee56-4733-95b9-983ac49f40b3.png)

- Bước 2: Dùng Base64 để Encoding , để có thể dùng được chúng ta cần encoding Access token khi nảy, các bạn vào server gõ lệnh sau là được.
```shell
# Replace 'my_service_username' with your GitLab username and 'my-gitlab-token' with your PAT.
printf "%s:%s" "username-gitlab-của-bạn" "access-token-vừa-tạo" | base64 -w0
```
![](https://images.viblo.asia/180d92de-e0f6-43cf-b7de-cbc90f01b72f.png)

- Bước 3: Thêm biến `DOCKER_AUTH_CONFIG` vào CICD Variables. Các bạn vào CICD Variables như trước đó đã làm vào tạo thêm biến DOCKER_AUTH_CONFIG, thêm giá trị bên dưới vào:
```json
{
  "auths": {
    "registry.gitlab.com": {
      "auth": "bXlfc2VydmljZV91c2VybmFtZTpteS1naXRsYWItdG9rZW4="
    }
  }
}
```
![](https://images.viblo.asia/3cc7bc81-5609-4131-82fd-8d8e1b8eebbf.png)

- Bước 4: Thêm login vào script deploy. Thêm lệnh sau vào trong script `docker login registry.gitlab.com -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN`
```.gitlab/.pipeline-template.yml
.deploy:
  ...
  script:
    - >-
      $SSH_CMD "docker login registry.gitlab.com -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN && cd $WORK_DIR && docker compose pull && docker compose up -d &&
      exit"
```

Vậy là xong, các bạn thử  push code lên và run pipeline lại nhé.
![](https://images.viblo.asia/ccd78c6a-8886-4271-bd8c-485c47659a31.png)
Có thể thấy job của chúng ta đã thành công như mong đợi:

![](https://images.viblo.asia/8333a14c-7991-4ad8-a060-9fa76f2e3eec.png)


Kiểm tra lại kết quả ở phía server:
![](https://images.viblo.asia/92a1b944-6b58-4d0f-93e0-217edb6954fe.png)


Đến đây chúng ta đã hoàn thiện xong quá trình set up CICD cho nhánh staging, các nhanh còn lại tuỳ theo nhu cầu mà các bạn có thể tự config cho dự án, từ giờ các bạn chỉ cần push code lên nhánh đã setup, mọi thứ còn lại cứ để Gitlab CI/CD lo 😁. 
# Kết luận 📝
Vậy là chúng ta đã đi qua quá trình cài đặt CI/CD với Gitlab CI/CD để hỗ trợ quá trình quản lý dự án cũng như tìm hiểu về cách viết 1 file CI/CD từ đơn giản đến tối ưu. Hy vọng bài viết này sẽ giúp ít cho các bạn trong quá trình phát triển dự án 😄.

Nếu các bạn góp ý hoặc ở bài viết có phần nào không hợp lý có thể comment để mọi người cùng nhau chỉnh sửa nhé. Cảm ơn mọi người đã giành thời gian đọc bài viết.

# Tài liệu tham khảo 🔍
- CI/CD YAML syntax reference (no date a) GitLab. Available at: https://docs.gitlab.com/ee/ci/yaml/index.html (Accessed: 18 January 2025). 
- Tutorial: Create and run your first gitlab CI/CD pipeline (no date) GitLab. Available at: https://docs.gitlab.com/ee/ci/quick_start/ (Accessed: 18 January 2025). 
- Ngoan98tv - Overview (no date) GitHub. Available at: https://github.com/ngoan98tv (Accessed: 10 February 2025). 
- Linuxbeast (2024) How to pull Docker images from private GitLab registry with Gitlab CI/CD, Linuxbeast. Available at: https://linuxbeast.com/blog/how-to-pull-docker-images-from-private-gitlab-registry-with-gitlab-ci-cd/ (Accessed: 10 February 2025). 
# Change log 📓
- January, 25, 2025 - Init document
- February, 06, 2025 - Prepare to release 😁