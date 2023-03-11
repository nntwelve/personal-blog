---
layout: post
title: 'Cài đặt LTI cho Canvas LMS với NestJS Framework'
date: 2023-02-01 17:27:12 -0000
categories: Canvas Learning Management System
---

# Dockerizing LTI NestJS trong Canvas LMS với sự hỗ trợ của ChatGPT

Xin chào các bạn, dạo gần đây **ChatGPT** đang nổi lên như một thế lực trong giang hồ, vừa hay hôm qua có bác chia sẻ cách đăng ký tài khoản khá đơn giản. Mình có đăng ký và dùng thử thì cũng thấy thật sự kinh ngạc vì sự tiện lợi mà nó mang lại.

Hôm nay mình sẽ **Dockerizing** **[LTI Absense Request](https://viblo.asia/p/viet-1-lti-don-gian-dung-nestjs-va-reactjs-trong-canvas-lms-5pPLkP76VRZ)** mà lần trước chúng ta đã tạo và kết hợp với sự trợ giúp từ **ChatGPT**.

Thời điểm hiện tại mình đang dùng cho bài viết này:

- Ubuntu 20.04
- Docker version 20.10.7, build f0df350
- docker-compose version 1.26.2, build eefe0d31
- ChatGPT Jan 30 Version

# Tạo Dockerfile và docker-compose file với ChatGPT

Sau nhiều lần thử nghiệm với các câu hỏi khác nhau, khá bất ngờ là mỗi lần đều cho mình một kết quả gần chính xác với yêu cầu, cuối cùng mình thấy kết quả cho câu hỏi này khá là thích hợp:

> Please help me create a scenario to Dockerizing a NestJS project and Mongodb with docker-compose. Please use node:18.12.1-bullseye-slim for Dockerfile. NestJS run on port 3333, Mongodb has basic auth. Both of these services must use the same network. They all read .env file from current directory and have unless-stopped restart option.

![image.png](https://images.viblo.asia/c53aac1e-1406-4e84-b72d-b9807e289b6a.png)
![image.png](https://images.viblo.asia/419b907b-609c-440f-9802-687ddcbad2a4.png)

- File Dockerfile thì khá ổn, còn docker-compose thì mình thấy thiếu tên container và **Nestjs** service vẫn chưa depend service **Mongodb** khởi chạy nên mình sẽ yêu cầu **ChatGPT** cập nhật lại:

> Add container name to services in docker-compose. NestJS must depend on mongodb. Mongodb service read .env file from current directory to use in docker-compose file.

![image.png](https://images.viblo.asia/81e4b78a-8351-4a56-ba6d-1d38f982f7c0.png)

- Trông cũng khá là ổn, mình sẽ copy qua đây để các bạn có thể dễ thao tác:

**Dockerfile**

```Dockerfile:Dockerfile
# Use node:18.12.1-bullseye-slim as the base image
FROM node:18.12.1-bullseye-slim

# Set the working directory
WORKDIR /app

# Copy the package.json and package-lock.json files
COPY package*.json ./

# Install the dependencies
RUN npm ci

# Copy the rest of the project files
COPY . .

# Expose port 3333
EXPOSE 3333

# Set the command to run the project
CMD [ "npm", "start" ]
```

**docker-compose.yml**

```yaml:docker-compose.yml
version: '3'
services:
  nestjs:
    build: .
    container_name: nestjs
    environment:
      - NODE_ENV=production
    ports:
      - "3333:3333"
    restart: unless-stopped
    networks:
      - app-network
    volumes:
      - .:/app
    depends_on:
      - mongodb
  mongodb:
    image: mongo
    container_name: mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    ports:
      - "27017:27017"
    restart: unless-stopped
    networks:
      - app-network
    volumes:
      - .:/app
      - ./mongo-data:/data/db
networks:
  app-network:
    driver: bridge

```

- Trước đây khi chưa có **ChatGPT** mình thường tốn khoảng chừng 10-15p để viết các file này, hiện tại mặc dù mới sử dụng nhưng mình thấy với vài dòng text đã giúp tiết kiệm đc khoảng 50% thời gian. Tuy nhiên chúng ta cũng cần phải chỉnh sửa lại để phù hợp với nhu cầu của mình hơn.

- Đầu tiên mình sẽ chỉnh sửa lại một tí để thử xem file **ChatGPT** đưa chúng ta có chạy được không. Ở **Dockerfile** mình sẽ thay đổi lệnh run về development. **Docker-compose** file sẽ đọc file và .env và chỉnh sửa biến để xác thực cho database và port trùng với trong file **.env**. Chúng ta cũng không cần mount volume của source vào trong mongodb container.

**Dockerfile**

```Dockerfile:Dockerfile
# Use node:18.12.1-bullseye-slim as the base image
FROM node:18.12.1-bullseye-slim

# Set the working directory
WORKDIR /app

# node:18.12.1-bullseye-slim throw `ps: not found` if using nestjs watch mode, so install this stuff will help us resolve this error
RUN apt-get update && apt-get install -y procps && rm -rf /var/lib/apt/lists/*
...
CMD ["npm", "run", "start:dev"]
```

**docker-compose.yml**

```yaml:docker-compose.yml
version: '3'
services:
  ...
  mongodb:
    ...
    env_file:
      - .env
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${DATABASE_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${DATABASE_PASSWORD}
    ports:
      - "27022:27017"
    expose:
     - 27017
    restart: unless-stopped
    networks:
      - app-network
    volumes:
      # - .:/app
      - ./mongo-data:/data/db
    ...
```

- Vì chúng ta đã thêm và auth cho database nên cần chỉnh sửa lại biến DATABASE_URI trong file **.env**.

_Lưu ý_: chúng ta sẽ thay đổi localhost thành tên service của mongodb thì mới có thể kết nối được.

```shell:.env
...
DATABASE_URI=mongodb://admin:admin@mongodb:27017
...
```

- Chạy thử với docker-compose:

`docker-compose up`

- Nếu bạn nào gặp lỗi như hình dưới thì vào cập nhật lại phần kết nối mongoose ở **app.module.ts**.

![image.png](https://images.viblo.asia/535ac913-7bab-4ebb-b740-07dc950d4a29.png)

> Phần này mình không rõ nguyên do tại sao bị lỗi như vậy, nếu chúng ta không cấu hình xác thực cho mongo thì có thể kết nối bình thường nhưng ngược lại thì chúng ta phải tách ra thành 2 option như phía dưới. Bạn nào biết nguyên do có thể comment cho mình và mọi người biết nha. Cảm ơn các bạn

**app.module.ts**

```typescript:app.module.ts
...
MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (config_service: ConfigService) => ({
        uri: config_service.get<string>('DATABASE_URI'), // Tách database name thành option ở dưới
        dbName: config_service.get<string>('DATABASE_API_NAME'),
      }),
    ...
```

- Kết quả:

![image.png](https://images.viblo.asia/e0b41b4e-1e57-42e7-8265-776ad9259b13.png)

- Truy cập http://localhost:3333/lti để kiểm tra. Nếu kết quả như hình thì đã thành công.

![image.png](https://images.viblo.asia/7a4cfcb9-1020-4326-8e4c-455cc8d35624.png)

Vậy là chúng ta đã Dockerizing thành công với sự hỗ trợ của **ChatGPT**, nhưng dừng ở đây thì quá đơn sơ. Chúng ta sẽ đến với phần tiếp theo để chỉnh sửa lại cho chỉnh chu hơn.

# Cấu hình cho môi trường development

Phần này chúng ta sẽ chỉnh sửa lại để thuận tiện và phù hợp với quá trình phát triển module. Khi nảy chúng ta đã chỉnh sơ qua để kiểm tra nên giờ chỉ cần chỉnh sửa thêm đôi chút:

- Ở **Dockerfile** chúng ta sẽ sử dụng **multi-stage builds** cho các stage khác nhau. Ở đây mình đặt stage build front-end là _builder_, stage development là _development_ và stage production là _production_. Mình cũng sẽ xóa `EXPOSE 3333` và chuyển `CMD ["npm", "run", "start:dev"]` ra ngoài file docker-compose. Thay vào đó thêm vào `RUN npm run build` đảm bảo NestJS được build trước khi chạy.

**Dockerfile**

```Dockerfile:Dockerfile
FROM node:18.12.1-bullseye-slim as development

# Create app directory
WORKDIR /app

# node:18.12.1-bullseye-slim throw `ps: not found` if using nestjs watch mode, so install this stuff will help us resolve this error
RUN apt-get update && apt-get install -y procps && rm -rf /var/lib/apt/lists/*

# Copy the package.json and package-lock.json files
COPY package*.json ./

# Install the dependencies
RUN npm ci

# Copy the rest of the project files
COPY . .

RUN npm run build
```

- Các bạn chú ý mình sẽ sử dụng **npm ci** (**C**lean **I**nstall) thay vì **npm install**. Mục đích giúp chúng ta có thể đồng nhất version của các package căn cứ theo file **package-lock.json**.

> Use npm ci if you need a deterministic, repeatable build. For example during continuous integration, automated jobs, etc. and when installing dependencies for the first time, instead of npm install - [Webwoman](https://stackoverflow.com/questions/52499617/what-is-the-difference-between-npm-install-and-npm-ci)

> Các bạn có thể tận dụng **ChatGPT** để hiểu rõ hơn:
> ![image.png](https://images.viblo.asia/b077026c-55ba-4bd7-a4c1-6fa9c77c8cb5.png)

- File **docker-compose.yml** mình sẽ thay version thành ver cao hơn để có thể sử dụng với _multi stage_. Sau đó thêm vào lệnh build theo stage trong **Dockerfile** - giá trị của target chính là tên stage chúng ta define sau từ khóa **as** của dòng `FROM node:18.12.1-bullseye-slim as development`. Tiếp theo đổi lại tên service và _container_name_ để phân biệt với _production_. Sau cùng thêm vào lệnh run server dev.

```yml:docker-compose.yml
version: '3.8'
services:
  nestjs-dev:
    build:
      context: .
      target: development
    command: npm run start:dev
    container_name: nestjs-dev
    ...
```

- Thêm vào file **.dockerignore** ignore một số folder không cần thiết để rút ngắn thời gian ở Bước `COPY . .` .Ở đây chúng ta sẽ ignore **node_modules** vì nó sẽ được tạo ra trong container khi chúng ta run **npm ci**.

**.dockerignore**

```.dockerignore:.dockerignore
node_modules
mongo-data
```

- Build và chạy lại để kiểm tra kết quả:

`docker-compose up --build`

- Xem lại log của container _nestjs-dev_ vừa tạo nếu kết quả như hình kèm với khi các bạn chỉnh sửa code và save lại thì server tự động restart thì chúng ta đã thành công.

![image.png](https://images.viblo.asia/e0b41b4e-1e57-42e7-8265-776ad9259b13.png)

# Cấu hình cho môi trường production

Đối với môi trường _production_ cho LTI của chúng ta thì mình sẽ cho Docker tự động build front-end ở stage _builder_ sau đó copy vào stage _production_. Cập nhật **Dockerfile** và thêm vào 2 stage trên. Mình sẽ giải thích chi tiết trong file dưới.

**Dockerfile**

```Dockerfile:Dockerfile
FROM node:18.12.1-bullseye-slim as development
...

# Stage builder
FROM node:18.12.1-bullseye-slim as builder

WORKDIR /app

## Sao chép thư mục front-end vào container
COPY front-end ./

## Tiến hành cài đặt package
RUN npm ci

## Build ra file static
RUN npm run build

# Stage production
FROM node:18.12.1-bullseye-slim as production

## Cài đặt biến môi trường
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

WORKDIR /app
## Chúng ta vẫn cần thao tác lặp lại như ở development vì cả 2 stage nằm ở 2 container khác nhau
## và chúng ta chỉ copy nội dung build từ development chứ không copy node_modules
COPY package*.json ./

RUN npm ci --production

COPY . .

## Lấy thư mục đã build được từ development để sử dụng
COPY --from=development /app/dist ./dist

## Tương tự lây thư mục front-end đã build đuọc từ builder
COPY --from=builder /app/build ./public

## Ở đây khi chạy production chúng ta đã compile về Nodejs nên chỉ cần chạy như bình thường.
CMD [ "node", "dist/main" ]
```

> Giải thích: Các bạn sẽ thấy mình đã `npm ci` ở stage _development_ nhưng sẽ tiếp tục `npm ci --production` ở stage _production_ là do ở stage _development_ chúng ta đang dev nên sẽ sử dụng một số devDependencies. Còn ở môi trường _production_ thì chúng ta chỉ cần install các dependencies để giảm nhẹ node*modules. Đó là lý do vì sao chúng ta không copy node_modules ở stage \_development* qua stage _production_. Và tại sao lại copy package\*.json trước rồi mới copy phần còn lại thì việc này giúp chúng ta tận dụng chức năng Cache của Docker. - Cảm ơn bạn **Kỳ Thịnh** đã góp ý!

> Các bạn cũng có thể hỏi **ChatGPT** để xem câu trả lời đúng không:
> ![image.png](https://images.viblo.asia/970e7873-ff25-41d2-8b8d-a170f80c4d55.png)

- Thêm vào service **_nestjs-prod_** vào file **docker-compose.yml** để tiến hành chạy stage _production_. Phần này sẽ có 2 hướng triển khai: 1 là các bạn làm như mình cho ngắn gọn, 2 là các bạn có thể tạo thêm file **docker-compose.prod.yml**. Cách 1 thì sẽ ngắn gọn hơn nhưng sẽ không rõ ràng và dễ đọc hiểu như cách 2. Ở service mongodb chúng ta cũng cho mount volume ra bên ngoài để đảm bảo toàn vẹn dữ liệu.

**docker-compose.yml**

```yml:docker-compose.yml
version: '3.8'
services:
  nestjs-dev:
    ...
  nestjs-prod:
    build:
      context: .
      target: production
    container_name: nestjs-prod
    environment:
      - NODE_ENV=production
    ports:
      - "3333:3333"
    restart: unless-stopped
    networks:
      - app-network
    volumes:
      - .:/app
    depends_on:
      - mongodb
  mongodb:
    ...
    volumes:
      # - .:/app
      - ./mongo-data:/data/db
    ...
```

- Chạy 2 lệnh dưới đây để cập nhật source **front-end** từ **submodule**

`git submodule init`

`git submodule update --recursive --remote`

- Chạy thử với lệnh `docker-compose up --build nestjs-prod`. Nếu gặp lỗi như hình dưới thì các bạn mở file **tsconfig.build.json** thêm vào thư mục front-end.
  > Lỗi này là do khi build Typescript ở submodule front-end chúng ta chưa xài lệnh npm install hoặc npm ci. Có 2 cách giải quyết: hoặc là chỉnh sửa file **tsconfig.build.json** như trên hoặc là vào thư mục front-end gõ lệnh npm install hoặc npm ci.

![image.png](https://images.viblo.asia/0343ce63-cc51-40df-a70a-933ce1d0562d.png)

**tsconfig.build.json**

```json:tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": ["node_modules", "test", "dist", "**/*spec.ts",  "front-end"]
}
```

- Thử lại với lệnh trên, các bạn để ý có thể thấy khi chúng ta build stage _production_ thì **số bước** là **23** thay vì **7** ở development.
  ![image.png](https://images.viblo.asia/01982a5f-b0ad-42d8-b426-2f6825c1bb35.png)
- Nếu kết quả như hình thì chúng ta đã cấu hình xong. Chúng ta sẽ gặp 1 warning từ **ltijs** do **Dev Mode** vẫn còn enable, các bạn có thể vào để tắt đi nhé.

![image.png](https://images.viblo.asia/1a5a0c7d-d2ed-408a-b534-31ec3b100e03.png)

> Bổ sung: sử dụng các lệnh docker-compose đôi khi hơi dài dòng chúng ta sẽ tạo 2 npm script để rút gọn lại:

> **package.json**
>
> ```json
> {
>     ...
>     "script": {
>         ...
>         "docker:dev": "docker-compose up -d nestjs-dev",
>         "docker:prod": "docker-compose up -d nestjs-prod"
>     }
> }
> ```

# Kết luận

Vậy là chúng ta đã hoàn thành xong việc Dockerizing cho dự án LTI của mình cùng với sự giúp sức của **ChatGPT**. Việc phát triển và triển khai sẽ trở nên dễ dàng hơn rất nhiều và sau khi dockerizing thì việc tích hợp CI/CD cũng sẽ dễ dàng hơn. Chúng ta cũng có thể phần nào thấy được khả năng của **ChatGPT**, theo mình thì nếu chúng ta biết cách tận dụng thì **ChatGPT** sẽ là 1 trợ thủ đắc lực trong tương lai. Sắp tới mình sẽ viết 1 series về Lập trình với NestJS kết hợp với sự trợ giúp từ **ChatGPT** các bạn cùng đón xem nha.

Cảm ơn các bạn đã giành thời gian đọc bài viết. Chúc các bạn năm mới vui vẻ và thành công. Source code của bài viết này các bạn có thể tải về [ở đây](https://github.com/nntwelve/dockerizing-lti-nestjs).

# Tài liệu tham khảo

- https://stackoverflow.com/questions/52499617/what-is-the-difference-between-npm-install-and-npm-ci
- https://docs.docker.com/compose
- https://chat.openai.com
