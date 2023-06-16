---
layout: post
title: 'Setup Boilerplate cho dự án NestJS - Phần 1: Team có nhiều thành viên - Env, Joi, Husky, Commitlint, Prettier, Dockerizing'
date: 2023-03-28 17:27:12 -0000
categories: NestJS
---

Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---

Xin chào mọi người, ở [bài viết trước](https://viblo.asia/p/cach-request-lifecycle-hoat-dong-trong-nestjs-y3RL1awpLao) chúng ta đã tìm hiểu sơ qua về **Request Lifecycle** để biết được cách request hoạt động như thế nào trong **NestJS**. Hôm nay mình sẽ đi vào sâu hơn trong quá trình lập trình với dự án thực tế.

# Đặt vấn đề

Khi setup dự án cho trong team có nhiều thành viên, chúng ta cần cân nhắc nhiều vấn đề để làm sao cho việc lập trình trở nên thuận tiện, đồng nhất và ít phụ thuộc vào môi trường trên máy của họ. Bên cạnh đó cũng giúp cho quá trình quản lí, theo dõi, triển khai dự án của chúng ta rõ ràng, dễ dàng hơn và không gặp các lỗi vặt liên quan đến môi trường dev của team member ( phiên bản, hệ điều hành,... ). Các lợi ích có thể kể đến như:

- Team member:
  - Dễ dàng start dev server với vài câu lệnh. Không lo về vấn đề version của các program và không cần phải cài đặt thêm các tool hỗ trợ ở máy local nếu không thực sự cần.
  - Tránh các lỗi do sự khác nhau giữa code formatter của các thành viên dẫn đến **Git** hiển thị các thay đổi không cần thiết.
  - Tránh lỗi về việc thiếu các biến môi trường do member khác thêm vào ở các feature của họ mà ta không biết dẫn đến mất thời gian tìm lỗi.
- Team leader:
  - Định dạng trước code formatter cho team member, buộc họ phải theo một quy chuẩn nhất định. Giúp quá trình review code dễ dàng hơn.
  - Chạy các script để kiểm tra trước khi member thực hiện các hành động trong Git ( commit, push, pull ).
  - Giúp commit message của các thành viên trong team rõ ràng và đầy đủ ngữ cảnh hơn.
  - Setup các môi trường với Docker sẽ giúp quá trình lập trình và triển khai production nhanh gọn hơn.

# Thông tin packages

- Docker version 23.0.1
- Docker Compose version v2.16.0
- npm version 9.5.0
- NestJS CLI version 9.2.0
- NestJS version 9.0.0
- Joi version 17.9.0
- Husky version 8.0.3
- @commitlint/cli version 17.5.0
- @commitlint/config-conventional version 17.4.4

Các bạn có thể tải về toàn bộ source code của phần này [ở đây.](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-1-dotenv-joi-husky-commitlint-docker)

# Prerequisites

Kiến thức cơ bản về **NestJS** và **Docker**

# Cài đặt NestJS

Phạm vi series này chúng ta sẽ không nói về các tính năng của **NestJS** nữa, mà sẽ tập trung vào quá trình phát triển để không làm mất thời gian của các bạn. Mình sẽ tiến hành khởi tạo dự án bằng **NestJS CLI** và đi thẳng vào vấn đề của bài viết.

`nest new boilerplate-nest-js`

> Do mình quen dùng snake_case cho các biến và property và CamelCase cho class và method mong các bạn thông cảm nếu làm các bạn khó chịu.

# 1. Environment variables

## 1.1 Mục đích sử dụng

Package **dotenv** không còn gì xa lạ gì với chúng ta, trong **NestJS** nó được cải tiến thành package **@nestjs/config**. Do đó có thêm nhiều tính năng để chúng ta có thể tích hợp vào dự án của mình. Các tính năng có thể kể đến như:

- Tự động load các giá trị cấu hình: tự động đọc các giá trị từ file .env.
- Parse các giá trị chuỗi: tự động parse các giá trị chuỗi thành các giá trị số, boolean, và json object, giúp cho việc đọc và sử dụng các giá trị cấu hình dễ dàng hơn.
- Tùy chọn file cấu hình linh hoạt: cho phép bạn cấu hình các tùy chọn linh hoạt cho việc đọc và sử dụng các giá trị cấu hình, như là tùy chọn mặc định, đường dẫn tới file cấu hình, một danh sách các file cấu hình khác nhau, v.v.
- Đọc cấu hình từ các nguồn khác nhau: cho phép đọc các giá trị cấu hình từ nhiều nguồn khác nhau, bao gồm các _biến môi trường_, các _file cấu hình_, các _command line arguments_, v.v.
- Hỗ trợ cho các module khác trong **NestJS**: cho phép sử dụng trong các module khác thông qua dependency injection.

## 1.2 Cài đặt

- Cài đặt: `npm i --save @nestjs/config`
- Cấu hình: thêm vào **AppModule** và dùng option `isGlobal:true`, mình sẽ dùng cho toàn bộ ứng dụng để không cần phải inject ở các module khác. Các bạn có thể dùng cho từng module cũng được.

```typescript:app.module.ts
...
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true })
   ]
  ...
```

Việc cài đặt chỉ đơn giản vậy thôi, tiếp theo chúng ta sẽ sử dụng một số tính năng đã liệt kê ở trên.

## 1.3 Tùy chọn file env linh hoạt

Đầu tiên trong quá trình development chúng ta có thể có nhiều môi trường khác nhau dành cho các trường hợp nhất định. Vì vậy cần có những file **.env** riêng cho các môi trường đó. Ví dụ ở đây mình sẽ có file **.env.dev** cho dev dùng ở máy của mình và file **.env** dùng cho các trường hợp còn lại.

```shell:.env.dev
PORT=3333
```

```shell:.env
PORT=3000
```

- Để sử dụng chúng ta sẽ thêm option `envFilePath` trong **ConfigModule**

```typescript:app.module.ts
...
@Module({
  imports: [
    ConfigModule.forRoot({
        isGlobal: true,
        envFilePath: process.env.NODE_ENV === 'development' ? '.env.dev' : '.env',
    })
  ]
  ...
```

- Tuy nhiên ban đầu chúng ta chưa có giá trị cho biến **NODE_ENV** nên mình sẽ chỉnh sửa lại lệnh start của **NestJS** trong **package.json**.

  ```json:package.json
  ...
      "scripts": {
          ...
          "start:dev": "NODE_ENV=development nest start --watch",
          ...
  ```

> Bạn nào có dự định thêm `NODE_ENV` vào các file env để không phải thêm ở đây thì nên lưu ý: chúng ta chỉ lấy được các giá trị trong file env **sau khi** nó được **dotenv** load thành công, còn chúng ta dùng `envFilePath` là trước khi nó được load.

- Các bạn thêm vào lệnh log ra `process.env.NODE_ENV` và `process.env.PORT` trong file **main.ts** để kiểm tra kết quả. Có thể thấy khi chúng ta thay đổi môi trường thì file **env** tương ứng sẽ được đọc.

![](https://images.viblo.asia/c10db65c-a335-452a-bcd8-f3450c2a4d20.gif)

## 1.4 Custom configuration files

Nếu trong trường hợp dự án có nhiều biến môi trường và chúng ta muốn gộp lại thành các nhóm có liên quan với nhau thì có thể tiến hành custom lại và dùng option `load`.

- Lấy ví dụ chúng ta có các biến env liên quan tới database:

```shell:.env.dev
DATABASE_PORT=27017
DATABASE_HOST=localhost
DATABASE_URI=mongodb://localhost
```

- Các bạn có thể custom lại vào file như bên dưới :

```typescript:src/configs/configuration.config.ts
export interface DatabaseConfig {
  host: string;
  port: number;
  uri: string;
}

export const database_config = () => ({
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10),
    uri: process.env.DATABASE_URI,
  },
});
```

- Sau đó thêm option `load` kèm với `database_config`, ở đây giá trị của option `load` nhận là Array nên chúng ta có thể có thêm nhiều config khác

  ```typescript:app.module.ts
  import { database_config } from './configs/configuration.config';
  ...
  @Module({
   imports: [
     ConfigModule.forRoot({
         isGlobal: true,
         envFilePath: process.env.NODE_ENV === 'development' ? '.env.dev' : '.env',
         load: [database_config], // <== Ở đây
     })
   ]
   ...
  ```

- Thử lấy ra biến database trong file **main.ts** để kiểm tra kết quả.
  ```typescript:main.ts
  ...
  async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    const config_service = app.get(ConfigService);
    const logger = new Logger(bootstrap.name);
    const database_env = config_service.get<DatabaseConfig>('database');
    logger.debug(database_env);
    await app.listen(3000);
  }
  bootstrap()
  ```
- Có thể thấy bên cạnh việc gôm nhóm các biến lại, chúng ta còn có thể truyền vào _interface_ để sử dụng **IntelliSense** của **VSCode**
  ![](https://images.viblo.asia/62bd8c31-df82-40b2-b62d-0f729ad546d3.gif)

## 1.5 Cache và ExpandVariables

- Theo như tài liệu của **NestJS** thì truy cập trực tiếp vào `process.env` có thể không được tối ưu, việc sử dụng option `cache` giúp tăng hiệu năng khi **ConfigService#get** gọi đến các biến trong `process.env`.
- Option **expandVariables** giúp chúng ta truy cập vào một biến môi trường khác trong file env. Ví dụ như bên dưới nếu không có option **expandVariables** thì kết quả khi gọi **ConfigService#get('DATABASE_URI')** sẽ là `mongodb://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@localhost` thay vì `mongodb://admin:admin@localhost`

  ```shell:.env.dev
  PORT=3333

  DATABASE_USERNAME=admin
  DATABASE_PASSWORD=admin
  DATABASE_PORT=27017
  DATABASE_HOST=localhost
  DATABASE_URI=mongodb://${DATABASE_USERNAME}:${DATABASE_PASSWORD}@localhost
  ```

- Thêm 2 option trên vào **ConfigModule**

  ```typescript:app.module.ts
  import { database_config } from './configs/configuration.config';
  ...
  @Module({
   imports: [
     ConfigModule.forRoot({
         isGlobal: true,
         envFilePath: process.env.NODE_ENV === 'development' ? '.env.dev' : '.env',
         load: [database_config],
         cache: true, // <== Ở đây
         expandVariables: true, // <== Ở đây
     })
   ]
   ...
  ```

# 2. Joi

## 2.1 Mục đích sử dụng

**Joi** là một thư viện validation dữ liệu trong **Node.js** với hơn 8 triệu lượt tải hàng tuần. Nó cho phép xác thực dữ liệu đầu vào và định dạng theo cách chúng ta muốn.
Các công dụng chính của **Joi** bao gồm:

- _Xác thực dữ liệu đầu vào_: **Joi** cung cấp một số phương thức để kiểm tra tính hợp lệ của dữ liệu đầu vào. Ví dụ, chúng ta có thể sử dụng phương thức **string()** và **required()** để kiểm tra xem giá trị đầu vào có tồn tại và phải là chuỗi hay không.

- _Định dạng dữ liệu_: chúng ta cũng có thể sử dụng **Joi** để định dạng dữ liệu đầu vào theo một cách mà mình muốn. Ví dụ, sử dụng phương thức **trim()** để xóa khoảng trắng ở đầu và cuối chuỗi.

- _Tạo schema_: dự án này mình sẽ sử dụng **Joi** để tạo schema, đó là một đối tượng mô tả các thuộc tính và kiểu dữ liệu cho một đối tượng. Chúng ta sẽ tạo một schema để đảm bảo các biến môi trường có kiểu dữ liệu và định dạng đúng.

- _Xử lý lỗi_: Nếu dữ liệu đầu vào không hợp lệ, Joi sẽ trả về một đối tượng lỗi chi tiết, cho biết vì sao dữ liệu không hợp lệ và ở đâu.

Phạm vi bài viết này chúng ta sẽ dùng **Joi** để validate dữ liệu của biến môi trường để đảm bảo khi team member clone source code về phải cung cấp đủ các biến mới có thể sử dụng để tránh các lỗi không mong muốn.

> Các bạn cũng có thể sử dụng **class-validator** và **class-transformer** để validate các biến môi trường. Việc custom với dự án có nhiều biến môi trường sẽ mất thời gian, nên mình thích dùng **Joi** để nhanh gọn hơn, tham khảo docs ở đây https://docs.nestjs.com/techniques/configuration#custom-validate-function

## 2.2 Cài đặt

- Tiến hành cài đặt:`npm i --save joi`

- Thêm vào **AppModule**, tạo `validationSchema` để validate cho biến môi trường.
  ```typescript:app.module.ts
  import * as Joi from 'joi';
  ...
  @Module({
   imports: [
     ConfigModule.forRoot({
         validationSchema: Joi.object({
             NODE_ENV: Joi.string()
               .valid('development', 'production', 'test', 'provision','staging').default('development'),
             PORT: Joi.number().default(3000),
         }),
       ...
  ```

Mình sẽ giải thích chi tiết ở phần dưới

## 2.3 Validate và Convert biến môi trường trong file env

- Ở mục trên chúng ta validate để đảm bảo `NODE_ENV` phải thuộc vào các giá trị trong `valid` và bằng cách thêm vào `default` là `developemnt` giúp chúng ta chỉ định mặc định `NODE_ENV=development`.
  > Tuy nhiên việc chỉ định `default` ở đây khác với trong scripts của **package.json** chúng ta thêm ở trên (`"start:dev": "NODE_ENV=development nest start --watch"`), ở **envFilePath** option lúc này **ConfigModule** vẫn còn đang trong quá trình khởi tạo nên không tồn tại giá trị `default` của **Joi**. Vì thế chúng ta nên thêm vào giá trị `NODE_ENV` trước các scripts để tránh các lỗi không mong muốn.
- Ban đầu nếu chúng ta chưa sử dụng **Joi** thì mặc định kiểu dữ liệu của các biến môi trường đều là _string_. Sau khi sử dụng thì kiểu dữ liệu của `PORT` đã được chuyển thành _number_
  ![](https://images.viblo.asia/3772081c-366f-493a-a532-9aec79261dda.gif)

## 2.4 Validation Options

**Joi** cung cấp cho chúng ta nhiều _option_ để thêm vào **ConfigModule** với các công dụng khác nhau tùy theo nhu cầu dự án để các bạn chọn. Ở đây mình chỉ sử dụng `abortEarly: false` để hiển thị toàn bộ các biến môi trường có lỗi thay vì mặc định chỉ hiển thị biến đầu tiên. Các bạn có thể tham khảo các option còn lại [ở đây](https://joi.dev/api/?v=17.8.3#anyvalidatevalue-options).

```typescript:app.module.ts
import * as Joi from 'joi';
...
@Module({
 imports: [
   ConfigModule.forRoot({
       validationSchema: Joi.object({
           NODE_ENV: Joi.string()
             .valid('development', 'production', 'test', 'provision','staging').default('development'),
           PORT: Joi.number().default(3000),
       }),
       validationOptions: {
         abortEarly: false,
       },
     ...
```

- Kết quả như hình bên dưới
  ![](https://images.viblo.asia/3af97fc3-9f7f-46ed-819f-6e081ee24f14.gif)

# 3. Dockerizing

Trong quá trình dev đồng nhất môi trường code giữa các thành viên trong team là vô cùng quan trọng, tránh các lỗi phát sinh do môi trường không đồng nhất. Bên cạnh đó việc triển khai production cũng không kém phần quan trọng. Docker sẽ là thứ trợ giúp chúng ta trong các vấn đề trên, cùng mình điểm qua một số công dụng của **Docker** trong quá trình dev:

- _Đảm bảo sự đồng nhất giữa môi trường phát triển_: Mỗi thành viên trong team sử dụng **Docker** để xây dựng và triển khai ứng dụng, giúp đảm bảo rằng môi trường phát triển của mỗi người đều đồng nhất, tránh các lỗi phát sinh do khác nhau trong cấu hình hệ thống.

- _Dễ dàng quản lý phát triển và triển khai ứng dụng_: **Docker** cho phép đóng gói toàn bộ ứng dụng và các dependencies của nó vào một container, giúp việc triển khai ứng dụng trở nên đơn giản và dễ dàng hơn. Việc quản lý phiên bản và triển khai ứng dụng cũng trở nên thuận tiện hơn khi sử dụng Docker.

- _Đảm bảo tính cô lập giữa các ứng dụng_: Khi sử dụng **Docker**, mỗi ứng dụng sẽ được đóng gói vào một container riêng biệt, giúp đảm bảo tính cô lập giữa các ứng dụng và tránh các xung đột về tài nguyên.

- _Giảm thiểu thời gian triển khai ứng dụng_: Với **Docker**, chúng ta có thể dễ dàng chia sẻ các container đã đóng gói ( dùng chuyển cho team test hoặc deploy cho khách hàng ), giúp giảm thiểu thời gian triển khai ứng dụng và tăng tính linh hoạt trong việc phát triển và triển khai ứng dụng.

- _Dễ dàng mở rộng và mở rộng ứng dụng_: Khi sử dụng **Docker**, có thể dễ dàng mở rộng ứng dụng của mình bằng cách thêm các container mới ( Database, MessageQueue, ELK Stack... ) hoặc cấu hình lại các container hiện có ( ví dụ chuyển MongoDB sang chạy Cluster ). Điều này giúp tăng tính linh hoạt và khả năng mở rộng của ứng dụng.

## 3.1 Mục đích sử dụng

Mục đích chúng ta sử dụng Docker trong dự án này sẽ bao gồm tất cả các công dụng nêu ở trên. Sử dụng trong quá trình dev, sau đó đóng gói api thành container kết hợp với **Gitlab CI/CD** để triển khai deploy lên các môi trường test, staging, production.

## 3.2 Cài đặt

- Phần cài đặt **Docker** thì tùy theo hệ điều hành mà các bạn có cách cài đặt riêng. Mình dùng Ubuntu nên sẽ cài theo [tài liệu này](https://docs.docker.com/engine/install/ubuntu).
- Tiếp theo tạo **Dockerfile** để cấu hình cho container. **Dockerfile** này nội dung sẽ tương tự nhưng chúng ta đã dùng ở [Series Dockerizing cho LTI](https://viblo.asia/p/dockerizing-lti-nestjs-trong-canvas-lms-voi-su-ho-tro-cua-chatgpt-5OXLAobr4Gr). Điểm khác biệt là mình sẽ dùng **Node 19** để tận dụng tính năng **crypto** cho bài viết tiếp theo về **Authentication**.

  ```Dockerfile:Dockerfile
  FROM node:19-alpine as development

  WORKDIR /usr/src/app

  COPY package*.json ./

  RUN npm install glob rimraf

  RUN npm install --only=development

  COPY . .

  RUN npm run build

  CMD [ "npm", "run", "start:dev" ]

  FROM node:19-alpine as production

  ARG NODE_ENV=production
  ENV NODE_ENV=${NODE_ENV}

  WORKDIR /usr/src/app

  COPY package*.json ./

  RUN npm install --only=production

  COPY . .

  COPY --from=development /usr/src/app/dist ./dist

  CMD [ "node", "dist/main" ]
  ```

- Tiến hành build thử image development với lệnh:

  `docker build --target development -t flash_cards_api_dev .`

- Kết quả thành công
  ![image.png](https://images.viblo.asia/40f4f5bd-4540-45b6-9e19-58886393e7cb.png)
- Sau khi có image tiến hành run thử với lệnh sau để kiểm tra kết quả:

  `docker run -p 3000:3333 -v .:/usr/src/app --name flash_cards_api_dev flash_cards_api_dev:latest`

## 3.3 Docker Compose

Với **Docker** thôi thì vẫn chưa đủ để tối ưu quá trình dev, **Docker Compose** sẽ giúp chúng ta kết hợp các container để cài đặt cũng như triển khai dễ dàng hơn.

- Tạo **docker-compose.dev.yml** để sử dụng cho môi trường dev

      ```yaml:docker-compose.dev.yml
      version: '3.8'

      services:
        flash_cards_api_dev:
          container_name: flash_cards_api_dev
          image: flash_cards_api_dev:1.0.0
          build:
            context: .
            target: development
          command: npm run start:dev
          ports:
            - ${PORT}:${PORT}
          volumes:
            - ./:/usr/src/app
          restart: unless-stopped

      networks:
        default:
          driver: bridge
      ```

  > **Lưu ý:** các option như `env_file` và `environment` chỉ ghi đè lên các biến môi trường bên trong container chứ không tác động đến các biến trong file docker-compose. Mặc định Docker Compose sẽ tự động đọc file .env ở cùng cấp thư mục với file docker-compose đang được chạy, cho nên biến `${PORT}` trên option `ports` sẽ lấy giá trị trong file **.env**. Các bạn thử run lệnh `docker compose config` để xem thử các giá trị các biến được thêm vào file docker-compose.
  >
  > Nếu các bạn thêm option `env_file` hoặc `environment` thì giá trị trong các biến này sẽ ghi đè lên các biến môi trường khi nảy chúng ta config ở **ConfigModule** (`envFilePath: process.env.NODE_ENV === 'development' ? '.env.dev' : '.env'`) nên các bạn phải hết sức lưu ý. Lúc trước mình từng gặp mà mãi mới tìm ra nguyên nhân, đọc thêm [ở đây](https://docs.docker.com/compose/environment-variables/envvars-precedence/).
  > ![image.png](https://images.viblo.asia/e2325e37-5ba6-47f7-a331-0c78a38e1bb0.png)
  >
  > **Bổ sung:** trong trường hợp nếu các bạn muốn chọn file env khác cho file docker-compose thay vì mặc định là file **.env** có thể dùng option **--env-file** khi run file docker-compose
  >
  > `docker compose --env-file ./.env.dev -f docker-compose.dev.yml up` - trick của bạn [Long Nguyen](https://stackoverflow.com/questions/39542905/docker-compose-not-recognizing-env-file-file-location-and-still-tries-to-use-th)

- Thêm vào scripts `docker:dev` trong **package.json**. Nếu muốn chạy ở **detach mode** thì các bạn thêm vào `-d` phía sau `up`.

```json:package.json
...
    scripts: {
        ...
         "docker:dev": "docker compose --env-file ./.env.dev -f docker-compose.dev.yml up",
       ...
```

- Chạy thử với lệnh `npm run start:dev`

![](https://images.viblo.asia/8a42e7a0-3dcf-4959-949e-e64d7dd5f8b4.gif)

Vậy là chúng ta đã cấu hình xong cho phần development của team, tiếp theo sẽ đến các package liên quan tới code formatter và quá trình chuẩn bị trước khi push code lên codebase.

# 4. Husky & Commitlint

**Husky** và **Commitlint** là hai công cụ được sử dụng trong quá trình phát triển phần mềm, đặc biệt là trong các dự án sử dụng **Git** để quản lý phiên bản và phát triển phần mềm theo phương pháp **Gitflow**.

- **Husky** giúp kích hoạt các lệnh hoặc hành động _trước_ hoặc _sau_ khi thực hiện các lệnh **Git** như _commit_, _push_, _pull_, _merge_, ... **Husky** cho phép thiết lập các kịch bản tự động hóa, giúp tăng hiệu quả và đảm bảo tính chính xác trong quá trình phát triển phần mềm.

- **Commitlint** giúp kiểm tra commit message trong quá trình commit code vào Git. Đảm bảo commit message đáp ứng các tiêu chuẩn và quy tắc cụ thể. Các tiêu chuẩn này có thể được định nghĩa và tùy chỉnh để phù hợp với quy trình phát triển phần mềm của từng dự án.

## 4.1 Mục đích sử dụng

Mình sẽ sử dụng **Husky** để kiểm tra trước khi một thành viên trong team commit code phải đảm bảo không vi phạm các rule của **eslint**. Và dùng **Commintlint** để đưa ra các chuẩn cho commit message.

## 4.2 Cài đặt

- Tiến hành install, vì chúng ta chỉ cần dùng cho môi trường development nên sẽ sử dụng -D ( --save-dev ) . Version hiện tại **@commitlint/cli:**^17.5.0, **@commitlint/config-conventional:**^17.4.4 và **husky:**^8.0.3.

  `npm install -D husky @commitlint/cli @commitlint/config-conventional`

- Để setup **Husky** chúng ta sẽ sử dụng lệnh `npx husky install`, tuy nhiên khi các thành viên clone source về mặc định sẽ không tự động cài đặt. Do đó, mình sẽ tạo script `prepare` trong **package.json** để tự động cài **Husky**, vì khi run lệnh `npm install` mặc định **NodeJS** sẽ tự động trigger script `prepare`. Các bạn có thể tham khảo thêm các cách khác [ở đây.](https://typicode.github.io/husky/#/?id=disable-husky-in-cidockerprod)

  ```json:package.json
  ...
      "scripts": {
          "prepare": "test -d node_modules/husky && husky install || echo \"husky is not installed\"",
          ...
  ```

> `test -d node_modules/husky` dùng để kiểm tra husky có sẵn hay không trước khi install, tránh trường hợp khi sử dụng CI/CD bị gặp lỗi `sh: husky: command not found`

- Tiến hành chạy lệnh cài đặt **Husky**

  `npm install`

- Kết quả folder **.husky** được tạo ra, nếu kết quả như bên dưới thì dùng ta đã cài đặt thành công.

![image.png](https://images.viblo.asia/5947b439-0b50-4f98-a2f5-13d531fc34e0.png)

- Tiếp theo chúng ta sẽ cấu hình cho **Commitlint**, tạo file **commitlint.config.js**, chúng ta sẽ cấu hình chỉ cho phép team member tạo các commit message bắt đầu bằng một trong các từ trong enum ở dưới và thêm một số ràng buộc khác.

  ```json:commitlint.config.js
  module.exports = {
    extends: ['@commitlint/config-conventional'],
    rules: {
      'type-enum': [
        2,
        'always',
        [
          'feat', // Tính năng mới
          'fix', // Sửa lỗi
          'improve', // Cải thiện code
          'refactor', // Tái cấu trúc code
          'docs', // Thêm tài liệu
          'chore', // Thay đổi nhỏ trong quá trình phát triển
          'style', // Sửa lỗi kiểu chữ, định dạng, không ảnh hưởng đến logic
          'test', // Viết test
          'revert', // Revert lại commit trước đó
          'ci', // Thay đổi cấu hình CI/CD
          'build', // Build tệp tin
        ],
      ],
      'type-case': [2, 'always', 'lower-case'],
      'type-empty': [2, 'never'],
      'scope-empty': [2, 'never'],
      'subject-empty': [2, 'never'],
      'subject-full-stop': [2, 'never', '.'],
      'header-max-length': [2, 'always', 72],
    },
  };
  ```

- Giải thích
  _ Số 2 trong config ở trên là mức độ, bao gồm:
  _ 0: Không có lỗi (off)
  _ 1: Cảnh báo (warning)
  _ 2: Lỗi (error) \* _type-enum_: Giá trị của type (loại commit) phải thuộc danh sách được định nghĩa (từ khóa trong mảng bên trong rule). Giá trị này luôn phải xuất hiện trong message của commit. \* _type-case_: Type luôn phải được viết thường (lower-case). \* _type-empty_: Message không được để trống. \* _scope-empty_: Scope không được để trống. \* _subject-empty_: Subject không được để trống. \* _subject-full-stop_: Subject không được kết thúc bằng dấu chấm. \* _header-max-length_: Chiều dài của commit message không được vượt quá 72 ký tự.
  Quá trình cài đặt **Husky** và cấu hình **Commitlint** chỉ đơn giản vậy thôi, các bạn có thể tham khảo docs của [Husky](https://typicode.github.io/husky/#/) và [Commitlint](https://commitlint.js.org/#/reference-configuration) để cấu hình thêm tùy theo nhu cầu dự án.

## 4.3 Cách sử dụng

- Để sử dụng mình sẽ cho Husky dùng 2 hooks:

  - **Pre-commit**: dùng để kiểm tra code đã pass hết các rule của eslint chưa.
    - Sử dụng lệnh: `npx husky add .husky/pre-commit 'npm run lint'`
  - **Commit-msg**: sẽ dùng commitlint để check commit message căn cứ theo file cấu hình ở trên.
    - Dùng lệnh: `npx husky add .husky/commit-msg 'npx commitlint --edit $1'`

- Husky sẽ tạo cho chúng ta 2 file mới trong folder .husky như hình dưới
  ![image.png](https://images.viblo.asia/41924cf2-5e32-40b4-9b09-5901ab0e71f2.png)

- Test thử để xem kết quả, mình sẽ chỉnh sửa file **app.service.ts** để cho khi chạy lệnh `npm run lint` sẽ gặp lỗi eslint

  ```typescript:app.service.ts
  import { Injectable } from '@nestjs/common';

  @Injectable()
  export class AppService {
    constructor() {} // <=== Thêm vào constructor ở đây
    getHello(): string {
      return 'Hello word!';
    }
  }

  ```

![](https://images.viblo.asia/e3ba86fd-20eb-4b74-81db-33ff0c52bf8f.png)

- Có thể thấy trước khi commit thì **Husky** đã chạy lệnh `npm run lint` và phát hiện ra lỗi ở **app.service.ts**. Chỉnh file **app.service.ts** quay về như cũ và kiểm tra lại.
  ![image.png](https://images.viblo.asia/822cdcde-13d6-4389-a12c-7797ac4703f9.png)

- Lỗi **Eslint** đã hết, chúng ta tiếp tục nhận được lỗi về commitlint, do commit message của mình không đúng như định dạng ở file cấu hình nên không được chấp nhận. Thay commit message mới để xem kết quả (dev-env là scope, ở các feature khác các bạn sẽ thay bằng scope feature đó).

![image.png](https://images.viblo.asia/77d81acb-0a7b-44ed-8cea-56132f7934dc.png)

> Bạn nào gặp lỗi như bên dưới thì nguyên nhân là do quên tạo file **commitlint.config.js** như mình hoặc file **commitlint.config.js** không nằm đúng vị trí ( cùng cấp với **.husky** )
> ![](https://images.viblo.asia/ea831e3a-5cd2-4e20-b50b-1eba4b916d5d.png)

## 4.4 Bypass

Cho một số trường hợp nhất định, chúng ta có thể bypass `pre-commit` và `commit-msg` hook bằng cách thêm vào option `--no-verify`

`git commit -m "yolo!" --no-verify`

# 5. Prettier

## 5.1 Mục đích sử dụng

**Prettier** là một công cụ tự động định dạng mã nguồn (code formatter) cho các ngôn ngữ lập trình phổ biến như JavaScript, TypeScript, CSS, HTML, JSON, YAML, v.v. Prettier có thể tự động định dạng và sắp xếp lại mã nguồn theo một quy chuẩn nhất định, bao gồm cách thụt lề, dấu ngoặc, khoảng trắng, dòng trống, v.v. Prettier giúp tăng tính nhất quán và dễ đọc của mã nguồn, giảm thiểu sự khác nhau giữa các thành viên trong đội ngũ phát triển.

Trong dự án có nhiều thành viên thì Prettier đảm nhiệm các nhiệm vụ sau:

- Tăng tính nhất quán trong source code giữa các thành viên trong đội ngũ phát triển.
- Giảm thiểu thời gian và công sức cần thiết để định dạng và sắp xếp lại source code.
- Giúp tránh các lỗi về định dạng source code, đặc biệt là khi các thành viên có thói quen viết source code khác nhau.
- Dễ dàng tích hợp vào các công cụ quản lý source code như Git, để đảm bảo các thay đổi trong source code luôn tuân thủ quy chuẩn định dạng.

## 5.2 Cài đặt

Mặc định khi cài **NestJS** chúng ta đã có sẵn **Prettier**, cụ thể là file **.prettierrc** và version hiện tại là **2.3.2**
![image.png](https://images.viblo.asia/856d1ea2-b665-4ebd-8151-039ec6bde225.png)

Thêm vào file **.prettierignore** để ignore các file không cần format

```.prettierignore
dist/
node_modules/
```

## 5.3 Cách sử dụng

- Cải tiến file **.prettierrc** lại để thêm một số config thông dụng:

```json:.prettierrc
{
  "trailingComma": "all",
  "useTabs": true,
  "tabWidth": 2,
  "semi": true,
  "singleQuote": true,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

- Giải thích:
  - _trailingComma_: "all": Cho phép sử dụng dấu phẩy ở cuối cùng của một đối tượng hoặc mảng.
  - _useTabs_: true": Sử dụng tab thay vì space để thụt lề.
  - _tabWidth_: 2": Độ rộng của một tab là 2 ký tự.
  - _semi_: true": Sử dụng dấu chấm phẩy ở cuối câu lệnh.
  - _singleQuote_: true": Sử dụng dấu nháy đơn để bao quanh chuỗi.
  - _bracketSpacing_: true": Thêm khoảng trống giữa dấu ngoặc đơn và các phần tử bên trong nó.
  - _arrowParent_: "always": Luôn luôn sử dụng dấu ngoặc đơn cho tham số của hàm mũi tên.
- Cập nhật lại hook `pre-commit` để format lại source code trước khi commit. Nếu sử dụng VSCode và có cài extension Prettier thì mặc định khi save sẽ tự format theo file .pretterrc, nhưng đôi khi nếu có thành viên nào đó xài IDE khác thì chúng ta nên format trước khi commit cho đồng nhất.

  ```shell:pre-commit
  #!/usr/bin/env sh
  . "$(dirname -- "$0")/_/husky.sh"

  npm run format && npm run lint
  ```

- Add và Commit để kiểm tra kết quả, có thể thấy trong quá trình commit Prettier được gọi để check và format lại code cho chúng ta. Lưu ý sau khi Prettier format xong chúng ta cần phải commit lại những gì vừa được format.

![image.png](https://images.viblo.asia/8bb1ad39-793a-4ff1-8cac-c7b130fa9b51.png)

# 6. Path Aliases

Trong quá trình lập trình thường thì các IDE giúp chúng ta tự động tìm và import các module khi khớp với keyword chúng ta gõ. Tuy nhiên nó sẽ không được dễ nhìn cho lắm, ví dụ như: `../../../users/users.service.ts`. Để giải quyết vấn đề đó chúng ta sẽ sử dụng Path Aliases để làm câu lệnh import dễ nhìn hơn. Tiến hành cập nhật lại file **tsconfig.json**

```tsconfig.json
{
  "compilerOptions": {
    ...
    "paths": {
      "@modules/*": [
        "src/modules/*"
      ],
      "@configs/*": [
        "src/configs/*"
      ],
      "@repositories/*": [
        "src/repositories/*"
      ]
    },
  }
}
```

Với cài đặt trên, ví dụ ở trên của chúng ta có thể thay đổi thành `@modules/users/users.service.ts`. Trông dễ nhìn và gọn gàng hơn nhiều so với bạn đầu.

# Kết luận

Trong bài viết này, chúng ta đã tìm hiểu và cài đặt các công cụ hữu ích để giúp quản lý code và tăng hiệu quả cho dự án NestJS.

- Đầu tiên, chúng ta đã tìm hiểu về **Joi** - một thư viện validation mạnh mẽ để đảm bảo tính chính xác và đáng tin cậy cho các dữ liệu trong dự án của chúng ta.
- Tiếp theo, chúng ta đã cài đặt **Husky** - một công cụ giúp chúng ta kích hoạt các hook của git để tự động hóa việc kiểm tra và đảm bảo tính chính xác của các commit message.
- Sau đó, chúng ta đã tìm hiểu về **Commitlint** - một công cụ để đảm bảo rằng các commit message tuân thủ theo chuẩn định dạng và quy tắc cụ thể, giúp cho việc theo dõi commit dễ dàng hơn.
- Chúng ta cũng đã tìm hiểu về **Prettier** - một công cụ để định dạng code tự động, giúp cho code được viết đồng nhất và đẹp hơn, đồng thời giúp tăng hiệu quả trong việc đọc và hiểu code của các thành viên trong team.
- Cuối cùng, chúng ta đã tìm hiểu về **Dockerizing** - một phương pháp để đóng gói ứng dụng và đảm bảo sự đồng nhất trong môi trường chạy của ứng dụng.

Bài viết tiếp theo chúng ta sẽ đi sâu hơn vào lập trình, tìm hiểu về **Swagger** để cấu hình document, **class-validator** và **class-transformer** để validate dữ liệu.

Cảm ơn các bạn đã dành thời gian đọc bài viết, hẹn gặp lại vào các bài viết tiếp theo ^\_^.

# Tài liệu tham khảo

- Husky - git hooks. Available at: https://typicode.github.io/husky/#/ (Accessed: March 23, 2023).
- Lint commit messages commitlint. Available at: https://commitlint.js.org/#/ (Accessed: March 23, 2023).
- OpenAI. Available at: https://openai.com/ (Accessed: March 23, 2023).
- Documentation: Nestjs - a progressive node.js framework NestJS. Available at: https://docs.nestjs.com/techniques/configuration (Accessed: March 23, 2023).
