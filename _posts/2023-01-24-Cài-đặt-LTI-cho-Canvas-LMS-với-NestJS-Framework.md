---
layout: post
title: 'Cài đặt LTI cho Canvas LMS với NestJS Framework'
date: 2023-01-24 11:00:12 -0000
categories: Canvas Learning Management System
---

# Cài đặt LTI cho Canvas LMS với NestJS Framework

Xin chào các bạn, Tết của các bạn sao ùi. Hôm nay mùng 2 mình giành tí thời gian để viết tiếp Series, ở [**Phần 1**](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-tren-nodejs-018J2vAEJYK) chúng ta đã tìm hiểu về **LTI** cũng như cài đặt **LTI** trên **NodeJS** thuần, hôm nay chúng ta sẽ cùng đến với Phần 2 của [Series Sử dụng LTI với Canvas LMS](https://viblo.asia/s/canvas-learning-management-system-PwlVmR0045Z). Ở phần này chúng ta sẽ cùng cài đặt **[ltijs](https://www.npmjs.com/package/ltijs)** trên **[NestJS Framework](https://nestjs.com/)** để tận dụng các chức năng cần thiết của framework này.

Mình sẽ nói qua sơ lược về **[NestJS Framework](https://nestjs.com/)** để các bạn chưa sử dụng có thể làm quen, các bạn nào đã sử dụng qua thì có thể kéo xuống trực tiếp phần cài đặt để tiết kiệm thời gian nhé.

# NestJS là gì?

> A progressive Node.js framework for building efficient, reliable and scalable server-side applications.

**NestJS** (Nest) là một framework dùng để phát triển ứng dụng **NodeJS** hiệu quả và có khả năng mở rộng cao. Về cơ bản **NestJS** sử dụng framework [Express](https://expressjs.com/) hoặc [Fastify](https://www.fastify.io/) để làm **HTTP Server**. Mặc dù 2 framework trên đa phần đều giải quyết được đa số nhu cầu của chúng ta trong việc xây dựng và phát triển ứng dụng nhưng nó vẫn chưa đảm bảo được các đặc tính _clean structure_, _highly scalable, testable_ và dễ dàng _maintaince_.

**Nest** kết hợp cả 3 yếu tố **OOP**(Object Oriented Programming), **FP**(Functional Programming), **FRP**(Functional Reactive Programming). Nest supports **Typescript**, mình khá là thích tính năng này, vì Nest giúp chúng ta cài đặt Typescript tự động. Tuy nhiên nếu các bạn chưa làm quen với **Typescript** thì Nest vẫn hỗ trợ các bạn viết bằng **Javascript** thuần.

# Tại sao mình sử dụng NestJS cho LTI?

Có 1 số lý do có thể kể đến như:

- Cho phép develop nhanh và hiệu quả: **Nest** cung cấp [Nest CLI](https://docs.nestjs.com/cli/overview) giúp generate code tự động giúp mình tiết kiệm được thời gian mỗi khi tạo thêm các logic mới. Các bạn có thể tham khảo hình dưới:

![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510521/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/95b00aaf-f09b-4f92-8a9f-b29dc736b398_cmdhzq.png)

Chỉ cần gõ lệnh theo cú pháp của CLI thì sẽ tự tạo ra các file mặc định cho chúng ta.

Ví dụ khi gõ `nest g res lti` thì Nest CLI sẽ tạo ra các file cho chúng ta như hình:
![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510518/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/d03f241f-4b9c-4380-a0a1-3ad304df604b_u8rp0d.webp)

- Hỗ trợ **Typescript**: như mình đã nói ở trên, **Nest** tự động cấu hình **Typescript** compiler nên chúng ta không cần tốn thời gian cài đặt thủ công - lúc mới học **Typescript** mình đã tốn kha khá thời gian cho quá trình config **Typescript** compiler.
- **Nest** sử dụng **Dependency Injection** (**DI**): giúp tự động ủy quyền phụ thuộc các module cho **inversion of control** (**IoC**) thay vì chúng ta phải ủy quyền thủ công.

# Cài đặt LTI trên Canvas

Ở [Phần 1](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-tren-nodejs-018J2vAEJYK) chúng ta đã cài đặt xong **Developer Key** trên **Canvas**, ở phần này chúng ta sẽ cho **LTI** chạy ở route `/lti` nên sẽ chỉnh sửa lại _url_ trỏ về `http://localhost:3333/lti` trong **Developer Key** như hình dưới:
![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510517/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/a0537c47-74b8-4592-aa33-4cab18862b7d_ekbiya.webp)
Nếu bạn nào chưa tạo thì có thể quay lại [Phần 1](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-tren-nodejs-018J2vAEJYK) để xem chi tiết các bước.

# Cài đặt LTI trên NestJS Framework

Chúng ta sẽ tiến hành cài đặt package **ltijs**, thời điểm hiện tại của bài viết mình đang dùng **ubuntu**:24.04, **nestjs/\***:v9.0.0 và package **ltijs**:v5.9.0.

## 1. Tạo thư mục và khởi tạo project nestjs:

       nest new lti-account-role

Sau đó chọn **npm** , **yarn** hoặc **pnpm** tùy theo cách dùng của các bạn, mình sẽ chọn **npm**.

## 2. Install các package của NestJS

        npm install

## 3. Cài đặt package ltijs và các package cần thiết:

        npm install ltijs @nestjs/config joi

- [@nestjs/config](https://docs.nestjs.com/techniques/configuration): cấu hình đọc file chứa biến env
- [joi](https://joi.dev/api/?v=17.7.0): validate biến env

## 4. Sử dụng CLI tạo module LTI trong Nest

- `cd src && nest g res lti`
- Chọn `Y` vì ở đây chúng ta develop theo **REST API** nên các bạn chọn yes.
- Do phạm vi bài viết của chúng ta chỉ cần kết nối **ltijs** nên chúng ta không cần tạo CRUD entry points nên các bạn chọn `n`.

Chúng ta sẽ được cấu trúc như hình dưới:
![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510518/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/c2c6bd8c-72e9-4888-9238-2247b417e57b_pkwzy5.webp)

## 5. Cấu hình biến môi trường:

Tạo file **.env** với nội dung tương tự như phần trước.

```shell:.env
DATABASE_NAME=lti_acocunt_role
DATABASE_USERNAME=admin
DATABASE_PASSWORD=admin
DATABASE_PORT=27022
DATABASE_URI=mongodb://localhost:27022

PORT=3333 // PORT để Canvas kết nối vào LTI

LTI_HOST=https://your-canvas-domain.com // Các bạn đổi thành domain Canvas các bạn đang dùng nhé
LTI_CLIENT_ID=10000000000022 //  Đây là Client ID từ Developer key chúng ta tạo khi nảy
LTI_KEY=HkvZwP0DKtqWTUjX1qNjQdWiSBCZmGWNe7iRR73ke9MiosdVSrY583urVouN8mk5 // tương tự đây là Secret key từ Developer key
LTI_NAME=LTI_ACCOUNT_ROLE
LTI_ISS=https://canvas.instructure.com // ISS phải trùng với ISS trong file security.yml trong config của Canvas
```

- Nest cung cấp package [@nestjs/config](https://docs.nestjs.com/techniques/configuration) giúp cấu hình biến môi trường - về bản chất thì package này sử dụng package **dotenv** như chúng ta đã dùng ở [Phần 1](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-tren-nodejs-018J2vAEJYK) nhưng khi sử dụng sẽ dễ dàng thao tác với **Dependency Injection**.

- Package **[joi](https://joi.dev/api/?v=17.7.0)** giúp chúng ta kiểm tra xem các biến môi trường đã khai báo đầy đủ và đúng định dạng hay chưa. Chúng ta sẽ chỉnh sửa lại file app.module.ts như sau:

```javascript:app.module.ts
...
import * as Joi from 'joi';
@Module({
  imports: [
    LtiModule,
    ConfigModule.forRoot({
        validationSchema: Joi.object({
            DATABASE_NAME: Joi.string().required(),
            DATABASE_USERNAME: Joi.string().required(),
            DATABASE_PASSWORD: Joi.string().required(),
            DATABASE_PORT: Joi.number().required(),
            DATABASE_URI: Joi.string().required(),
            PORT: Joi.number().required(),
            LTI_KEY: Joi.string().required(),
            LTI_HOST: Joi.string().required(),
            LTI_CLIENT_ID: Joi.string().required(),
            LTI_NAME: Joi.string().required(),
            LTI_ISS: Joi.string().required(),
          }),
      isGlobal: true, // cho phép sử dụng Config Service ở mọi nơi
    }),
  ],
  ...
```

## 6. Tạo file LTI middleware

- **Middleware** này sẽ được gọi trước khi các request đến **LTI server**, ở đây sẽ là nơi chúng ta thêm vào logic của **ltijs** để xác thực các request gọi đến, mục đích để đảm bảo chỉ có các request xác thực từ **Canvas LMS** hoặc request gọi **API** từ giao diện của chính **LTI** là có thể thông qua. Để sử dụng **Middleware** trong **Nest** chúng ta implement interface **NestMiddleware**. Sau đó dùng interface **OnModuleInit** để cấu hình **ltijs** cũng như kết nối **ltijs** đến Database. Vì chúng ta đã cấu hình **Config Service** nên có thể inject để sử dụng.
  > 2 interfaces trên được cung cấp sẵn trong @nestjs/common

```typescript:lti.middleware.ts
import { Injectable, NestMiddleware, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request, Response } from 'express';
import { Provider as lti } from 'ltijs';

@Injectable()
export class LtiMiddleware implements NestMiddleware, OnModuleInit {
 /**
  *
  */
 constructor(private readonly config_service: ConfigService) {}
 async onModuleInit() { // Các config ltijs ở đây sẽ tương tự Phần 1
   lti.setup(
     this.config_service.get<string>('LTI_KEY')!, // Cách lấy ra biến môi trường
     {
       url:
         this.config_service.get<string>('DATABASE_URI') +
         '/' +
         this.config_service.get<string>('DATABASE_NAME') +
         '?authSource=admin',
       connection: {
         user: this.config_service.get<string>('DATABASE_USERNAME'),
         pass:
           this.config_service.get<string>('DATABASE_PASSWORD'),
       },
     },
     {
       appRoute: '/',
       invalidTokenRoute: '/invalidtoken',
       sessionTimeoutRoute: '/sessionTimeout',
       keysetRoute: '/keys',
       loginRoute: '/login',
       devMode: true,
       tokenMaxAge: 60,
     },
   );
   // Whitelisting the main app route and /nolti to create a landing page
   lti.whitelist(
     {
       route: new RegExp(/^\/nolti$/),
       method: 'get',
     },
     {
       route: new RegExp(/^\/ping$/),
       method: 'get',
     }
   );
   lti.onConnect((token, req: Request, res: Response) => {
     if (token) {
       res.json(res.locals?.context?.custom?.role);
     } else res.redirect('/lti/nolti');
   });
   await lti.deploy({ serverless: true });
   await lti.registerPlatform({
     url: this.config_service.get<string>('LTI_ISS'),
     name: this.config_service.get<string>('LTI_NAME'),
     clientId: this.config_service.get<string>('LTI_CLIENT_ID'),
     authenticationEndpoint: `${this.config_service.get<string>(
       'LTI_HOST',
     )}/api/lti/authorize_redirect`,
     accesstokenEndpoint: `${this.config_service.get<string>(
       'LTI_HOST',
     )}/login/oauth2/token`,
     authConfig: {
       method: 'JWK_SET',
       key: `${this.config_service.get<string>(
         'LTI_HOST',
       )}/api/lti/security/jwks`,
     },
   });
 }
// Request sẽ đến đây trước khi vào controller lti
// Chúng ta sẽ cho kết nối với ltijs ở đây
 use(req: Request, res: Response, next: () => void) {
   lti.app(req, res, next);
 }
}
```

> Lưu ý: các bạn để ý dòng `await lti.deploy({ serverless: true });` sẽ khác với [Phần 1](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-tren-nodejs-018J2vAEJYK), ở Phần 1 chúng ta dùng **Express Server** của **ltijs** nên sẽ để option _port_, còn ở đây chúng ta sẽ sử dụng **Server** đang chạy **Nest** nên option sẽ là _serverless_

- Thêm **Middleware** vào **LTI Module**

```typescript:lti.module.ts
import { MiddlewareConsumer, Module } from '@nestjs/common';
import { LtiController } from './lti.controller';
import { LtiMiddleware } from './lti.middleware';

@Module({
  controllers: [LtiController],
})
export class LtiModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LtiMiddleware).forRoutes('lti');
  }
}
```

## 7. Tạo controller để test kết nối

```typescript:lti.controller.ts
import { Controller, Get, Req, Res } from '@nestjs/common';
import { Request, Response } from 'express';

@Controller('lti')
export class LtiController {
  @Get('nolti')
  async nolti(@Req() req: Request, @Res() res: Response) {
    res.send(
      'There was a problem getting you authenticated with the attendance application. Please contact support.',
    );
  }

  @Get('ping')
  async ping(@Req() req: Request, @Res() res: Response) {
    res.send('pong');
  }

  @Get('protected')
  async protected(@Req() req: Request, @Res() res: Response) {
    res.send('Insecure');
  }
}
```

## 8. Chạy MongoDB:

Chúng ta vẫn cần chạy **MongoDB** để **ltijs** lưu các cấu hình kết nối.

## 9. Chạy Server bằng lệnh:

        npm run start:dev

- Nếu phản hồi như hình dưới thì server đã khởi chạy thành công.

![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510517/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/7a9fd47f-9bde-4e00-ab31-9f0839f1b7e7_ehawgi.webp)

- Truy cập http://localhost:3333/lti/ping để kiểm tra. Nếu message trả về là "pong" thì đã set up lti thành công.
- Truy cập http://localhost:3333 chúng ta sẽ trả về route mặc định được tạo bởi Nest CLI

![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510517/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/ad58a8fd-ee7e-4c6b-906a-e2116c662d9e_a49k6e.webp)

Chúng ta có thể thấy nếu truy cập vào các route bên trong **LTI** mà không có trong `whitelist` sẽ trả về message **NO_LTIK_OR_IDTOKEN_FOUND**, còn nếu ở ngoài lti thì vẫn có thể truy cập như bình thường.
![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510517/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/58b78dba-4cb4-4ac6-9b9c-5c5860100ac5_zngevn.webp)
Vậy là chúng ta đã cấu hình xong **ltijs** trong **Nest**, giờ chúng ta sẽ quay lại **Canvas LMS** để kiểm tra xem đã kết nối thành công chưa. Chúng ta có thể thấy kết quả trả về tương tự phần 1.

![image.png](https://res.cloudinary.com/ntngoc96/image/upload/v1678510518/blogs/C%C3%A0i%20%C4%91%E1%BA%B7t%20LTI%20cho%20Canvas%20LMS%20v%E1%BB%9Bi%20NestJS%20Framework/d0f89df8-ac82-496a-941e-344c252d2421_z3dye8.webp)

# Kết luận

Vậy là chúng ta đã tạo xong 1 custom **LTI** với **NestJS Framework**, chúng ta có thể tận dụng các công nghệ mà **Nest** mang lại để phát triển dự án một cách thuận tiện và tối ưu. Phần cuối của series mình sẽ là tích hợp **FE** chạy bằng **ReactJS** vào **LTI Server** của phần này để khởi chạy 1 **LTI** hoàn chỉnh.

Cảm ơn các bạn đã quan tâm theo dõi. Nếu có câu hỏi gì các bạn hãy comment phía dưới hoặc inbox riêng cho mình. **_Chúc các bạn mùng 3 Tết vui vẻ và thành công._**

Các bạn có thể tải về source code của phần tại [đây](https://github.com/nntwelve/LTI-with-nestjs).

# Tài liệu tham khảo

- https://viblo.asia/p/dependency-injection-bJzKmoXEl9N
- https://docs.nestjs.com/fundamentals/custom-providers
- https://github.com/txstate-etc/attendance-node/tree/main/api
