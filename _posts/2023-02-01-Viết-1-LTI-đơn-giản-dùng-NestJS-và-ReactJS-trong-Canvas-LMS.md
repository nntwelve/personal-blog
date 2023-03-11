---
layout: post
title: 'Cài đặt LTI cho Canvas LMS với NestJS Framework'
date: 2023-02-01 17:27:12 -0000
categories: Canvas Learning Management System
---

# Viết 1 LTI đơn giản dùng NestJS và ReactJS trong Canvas LMS

Xin chào mọi người, hôm nay chúng ta sẽ cùng hoàn thành [Series LTI với Canvas LMS](https://viblo.asia/s/canvas-learning-management-system-PwlVmR0045Z) để mọi người có thể có cái nhìn tổng quan về cách thức triển khai LTI.

Phần cuối này chúng ta sẽ viết _Front-end_ cho LTI bằng **ReactJS** sau đó serve static với **NestJS Server**. LTI của chúng ta sẽ có các chức năng liên quan tới việc quản lí yêu cầu vắng buổi học của sinh viên, chi tiết như sau:

- Sinh viên khi vào LTI có thể viết yêu cầu gửi đến giáo viên để xin vắng 1 buổi học.
- Sinh viên xem các yêu cầu mình đã gửi cho giáo viên.
- Giáo viên xem danh sách các yêu cầu nghỉ học của sinh viên.
- Giáo viên đưa ra quyết định cho yêu cầu nghỉ phép của sinh viên.

# Thiết kế database

Đối với các yêu cầu trên và nhằm mục đích rút ngắn phạm vi bài viết mình sẽ chỉ cần tạo 1 table để lưu dữ liệu - ở dự án thực tế các bạn nên tổ chức chặt chẽ hơn.

![image.png](https://images.viblo.asia/b0058bb2-267f-4dc3-8bd0-6f3137842568.png)

- `reason`: lý do xin vắng của sinh viên.
- `student_id, student_name`: thông tin sinh viên.
- `course_id`: id khóa học
- `status`: trạng thái của yêu cầu.
- `date`: ngày xin vắng.
- `confirmed_by_id, confirmed_by_name`: người duyệt yêu cầu (trong khóa học có thể có nhiều giáo viên, có thể dùng để hiển thị xem giáo viên nào duyệt).

# Triển khai API với NestJS

Mình sẽ tiếp tục sử dụng code đã tạo ở [Phần 2](https://viblo.asia/p/cai-dat-lti-cho-canvas-lms-voi-nestjs-framework-MG24BwNEJz3) các bạn có thể tải về source code tại [đây](https://github.com/nntwelve/LTI-with-nestjs).

## 1. Cài đặt các package cần thiết

    npm install --save @nestjs/mongoose mongoose @nestjs/mapped-types @nestjs/serve-static

## 2. Kết nối với MongoDB trong file app.module.ts

Chúng ta sẽ thay đổi _database name_ ở file **.env** để dễ phân biệt với _database name_ đã dùng để lưu data config của **ltijs**, các bạn có thể lưu chung hay riêng tùy theo phạm vi và yêu cầu của dự án.

```shell:.env
DATABASE_API_NAME=lti_absence_requests
DATABASE_LTIJS_NAME=ltijs
...
```

- Cập nhật lại _database name_:

```typescript:lti.middleware.ts
...
 async onModuleInit() {
    lti.setup(
      this.config_service.get<string>('LTI_KEY')!,
      {
        url:
          this.config_service.get<string>('DATABASE_URI') +
          '/' +
          this.config_service.get<string>('DATABASE_LTIJS_NAME') +
          '?authSource=admin',
          ...
```

- Kết nối API đến database

```typescript:app.module.ts
...
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (config_service: ConfigService) => ({
        uri: config_service.get<string>('DATABASE_API_NAME'),
      }),
      inject: [ConfigService],
    }),
    ...
....
```

## 3. Tạo Module Absence Requests

- Sử dụng **Nest CLI** để tạo files trong module:

      cd src && nest g res absence-requests

  Chọn **REST API** và **Y**.

- Tạo file **absence-request.entity.ts** với các fields tương ứng từ schema đã thiết kế

```typescript:absence-request.entity.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export enum ABSENCE_REQUEST_STATUS {
  PENDING = 'PENDING',
  APPROVED = 'APPROVED',
  REJECTED = 'REJECTED',
}

export type AbsenceRequestDocument = HydratedDocument<AbsenceRequest>;

@Schema({
  timestamps: {
    createdAt: 'created_at',
    updatedAt: 'updated_at',
  },
  collection: 'absence-requests',
})
export class AbsenceRequest {
  @Prop({
    required: true,
  })
  reason: string;

  @Prop({
    type: Number,
  })
  course_id: number;

  @Prop({
    type: Number,
    required: true,
  })
  student_id: number;

  @Prop({
    required: true,
  })
  student_name: string;

  @Prop({
    type: String,
    enum: ABSENCE_REQUEST_STATUS,
    default: ABSENCE_REQUEST_STATUS.PENDING,
    required: true,
  })
  status: ABSENCE_REQUEST_STATUS;

  @Prop({
    type: Date,
    required: true,
  })
  date: Date;

  @Prop({
    type: Number,
  })
  confirmed_by_id?: number;

  @Prop()
  confirmed_by_name?: string;
}

export const AbsenceRequestSchema =
  SchemaFactory.createForClass(AbsenceRequest);
```

- Chỉnh sửa lại file **absence-requests.module.ts** để kết nối _entity_ vừa tạo với _collection_ trong **MongoDB**.

```typescript:absence-requests.module.ts
...
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: AbsenceRequest.name, schema: AbsenceRequestSchema },
    ]),
  ],
  ...
...
```

- Sau khi hoàn thành 2 bước trên thì chúng ta đã có thể inject `AbsenceRequest` model trong file **absence-requests.service.ts**. Mình sẽ xóa 1 số hàm không sử dụng được **CLI** tạo sẵn. Sau đó thêm vào model đã tạo để sử dụng service của **mongoDB**:

```typescript:absence-requests.service.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { CreateAbsenceRequestDto } from './dto/create-absence-request.dto';
import { UpdateAbsenceRequestDto } from './dto/update-absence-request.dto';
import { AbsenceRequest, CatDocument } from './entities/absence-request.entity';

@Injectable()
export class AbsenceRequestsService {
  constructor(@InjectModel(AbsenceRequest.name) private absence_requests_model: Model<CatDocument>) {
  }
  create(create_absence_request_dto: CreateAbsenceRequestDto) {
    return 'This action adds a new absenceRequest';
  }

  async findAll() {
    return await this.absence_requests_model.find();
  }

  update(id: number, update_absence_request_dto: UpdateAbsenceRequestDto) {
    return `This action updates a #${id} absenceRequest`;
  }
}
```

- Tương tự với file **absence-requests.controller.ts**, mình chỉ chỉnh sửa lại function `findAll` để kiểm tra xem chúng ta đã kết nối tới database thành công chưa:

```typescript:absence-requests.controller.ts
import { Controller, Get, Post, Body, Patch, Param } from '@nestjs/common';
import { AbsenceRequestsService } from './absence-requests.service';
import { CreateAbsenceRequestDto } from './dto/create-absence-request.dto';
import { UpdateAbsenceRequestDto } from './dto/update-absence-request.dto';

@Controller('absence-requests')
export class AbsenceRequestsController {
  constructor(
    private readonly absence_requests_service: AbsenceRequestsService,
  ) {}

  @Post()
  create(@Body() create_absence_request_dto: CreateAbsenceRequestDto) {
    return this.absence_requests_service.create(create_absence_request_dto);
  }

  @Get()
  async findAll() {
    return await this.absence_requests_service.findAll();
  }

  @Patch(':id')
  update(
    @Param('id') id: string,
    @Body() update_absence_request_dto: UpdateAbsenceRequestDto,
  ) {
    return this.absence_requests_service.update(
      +id,
      update_absence_request_dto,
    );
  }
}
```

- Truy cập http://localhost:3333/absence-requests. Nếu trả về là mảng rỗng thì chúng ta đã tạo module và kết nối database thành công.

## 4. Apply ltijs middleware vào Absence Requests Module

Hiện tại chúng ta đang truy cập vào _Absence requests route_ mà không cần bất cứ xác thực nào. Đến lúc sử dụng chức năng auth của LTI để tăng tính bảo mật cho module.

- Chỉnh sửa file **lti.module.ts**, thêm vào route _absence-requests_:

```typescript:lti.module.ts
...
export class LtiModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LtiMiddleware).forRoutes('lti');
    consumer.apply(LtiMiddleware).forRoutes('absence-requests');
  }
}
```

- Truy cập lại http://localhost:3333/absence-requests để xem kết quả. Có thể thấy hiện tại nếu không có **_ltik_** gửi từ Canvas LMS thì chúng ta không thể truy cập vào route _absence-requests_ được nữa.

![image.png](https://images.viblo.asia/884beccd-0126-4b21-a88f-91244f63d349.png)

- Để có thể truy cập chúng ta cần có **ltik**, **ltijs middeware** sẽ lấy **ltik** từ _query_ của request để xác thực và cho phép chúng ta truy cập. Ở bước triển khai với ReactJS chúng ta sẽ có cách lấy ra khi LTI được hiển thị lên Canvas LMS.
- Cách lấy **ltik** để truy cập các bạn vào Canvas LMS => ấn **F12** để mở **Developer tools** => chọn qua **Tab Network** và chọn **Filter** theo **Doc**. Sau đó chúng ta mở LTI mà lần trước đã tạo ( mình có đổi tên hiển thị để phù hợp với phần này, các bạn muốn đổi chỉ cần vào _Developer key_ đổi _Title_ là được ). Ở giao diện **Developer tools** bên phải các bạn chọn dùng cuối như hình sẽ lấy được **ltik** trên query của request, chúng ta sẽ copy ltik này và gắn vào http://localhost:3333/absence-requests để xem thử kết quả

![ltik.png](https://images.viblo.asia/8930d9f1-c4a5-4d8b-8a26-ab8c46cdfaa3.png)

Kết quả:

![image.png](https://images.viblo.asia/77d71775-a239-43b8-b222-6ce19b8cda88.png)

Các bạn có thể thấy chúng ta đã truy cập được vào _absence-requests_ route tuy nhiên thay vì trả về mảng rỗng như trước thì lại trả về danh sách role. Nguyên nhân do chúng ta chưa chỉnh sửa lại function **onConnect** trong file **lti.middleware.ts**, chúng ta sẽ chỉnh lại như sau:

**lti.middleware.ts**

```typescript:lti.middleware.ts
...
lti.onConnect((token, req: Request, res: Response, next: NextFunction) => {
      if (token) {
        // Chúng ta sẽ thêm vào next function để điều hướng đến route ban đầu.
        next();
      } else res.redirect('/lti/nolti');
    });
...
```

Kết quả:
![image.png](https://images.viblo.asia/0827e5ae-d11e-44ae-835b-36fdd2241eff.png)

## 5. Viết API create absence request

- Chúng ta sẽ viết API cho sinh viên tạo yêu cầu xin vắng mặt, sinh viên sẽ nhập vào lý do và ngày xin nghỉ. Mình sẽ luớt qua phần validate cho dữ liệu gửi lên, validate mình sẽ để trong repo các bạn có thể vào để xem chi tiết.

**create-absence-request.dto.ts**

```typescript:create-absence-request.dto.ts
export class CreateAbsenceRequestDto {
  reason: string;
  date: Date;
}
```

- Các thông tin còn lại như student_id, student_name, course_id thì chúng ta sẽ lấy ra từ **ltik**. Mình sẽ tạo **custome decorator** để lấy ra các field đó để giúp code của chúng ta **DRY** hơn.

**lti.decorator.ts**

```typescript:lti.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const LtiVariables = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const custom_var = request.res.locals?.context?.custom;

    return data ? custom_var?.[data] : custom_var;
  },
);
```

- Bước tiếp theo tạo interface chứa các biến custom cần thiết mà chúng ta đã thêm vào Developer key. Các bạn nhớ thêm vào biến **Person.name.full** trong custom của Developer key để lấy ra tên người dùng đang đăng nhập, mình cũng sẽ bỏ _Global navigation_ trong **Placement** vì LTI chúng ta chỉ dùng trong khóa học.

**lti.interface.ts**

```typescript:lti.interface.ts
export interface ILtiVariables {
  userid: number;
  user_name: string;
  courseid: number | string; // Trong trường hợp mở LTI ở global sẽ không có course id
  course_name: string;
  role: string[];
}
```

- Sử dụng **lti decorator** ở _function create_ lấy ra các biến và truyền vào service:

**absence-requests.controller.ts**

```typescript:absence-requests.controller.ts
...
 @Post()
  async create(
    @LtiVariables() lti_vars: ILtiVariables,
    @Body() create_absence_request_dto: CreateAbsenceRequestDto,
  ) {
    return await this.absence_requests_service.create(
      lti_vars,
      create_absence_request_dto,
    );
  }
  ...
```

- Cập nhật logic absence-requests service

**absence-requests.service.ts**

```typescript:absence-requests.service.ts
...
async create(
    lti_vars: ILtiVariables,
    create_absence_request_dto: CreateAbsenceRequestDto,
  ) {
    if (isNaN(lti_vars.courseid as number)) {
      throw new BadRequestException();
    }
    return await this.absence_requests_model.create({
      reason: create_absence_request_dto.reason,
      date: new Date(create_absence_request_dto.date),
      student_id: lti_vars.userid,
      student_name: lti_vars.user_name,
      course_id: lti_vars.courseid,
    });
  }
...
```

- Vì đây là POST request nên chúng ta sẽ sử dụng Postman để kiểm tra kết quả:

![image.png](https://images.viblo.asia/e9339e53-9465-4e1e-8d3b-e782fc5355fb.png)

## 6. Viết API cập nhật trạng thái của request

- Logic của controller chúng ta chỉ cần nhận vào ID của request và biến _is_approve_ để biểu thị giáo viên có đồng ý cho sinh viên nghỉ hay không.

**update-absence-request.dto.ts**

```typescript:update-absence-request.dto.ts
export class UpdateAbsenceRequestDto {
  is_approve: boolean;
}
```

**absence-requests.controller.ts**

```typescript:absence-requests.controller.ts
...
 @Patch(':id')
  async update(
    @Param('id') id: string,
    @LtiVariables() lti_vars: ILtiVariables,
    @Body() update_absence_request_dto: UpdateAbsenceRequestDto,
  ) {
    return await this.absence_requests_service.update(
      id,
      lti_vars,
      update_absence_request_dto,
    );
  }
```

- Phần service chúng ta chỉ cho phép cập nhật các request có trạng thái PENDING.

```typescript:absence-requests.service.ts
...
async update(
    _id: string,
    lti_vars: ILtiVariables,
    update_absence_request_dto: UpdateAbsenceRequestDto,
  ) {
    await this.absence_requests_model.updateOne(
      {
        _id,
        status: ABSENCE_REQUEST_STATUS.PENDING,
      },
      {
        status: update_absence_request_dto.is_approve
          ? ABSENCE_REQUEST_STATUS.APPROVED
          : ABSENCE_REQUEST_STATUS.REJECTED,
        confirmed_by_id: lti_vars.userid,
        confirmed_by_name: lti_vars.user_name,
      },
    );
    return await this.absence_requests_model.findById(_id);
  }
```

- Tiến hành test thử với ID vừa tạo ở bước 5, có thể thấy response trả về trạng thái đã thay đổi dựa vào giá trị chúng ta gửi lên.

![image.png](https://images.viblo.asia/aad15b54-eef7-4951-9d0b-5cadf0ce0fa5.png)

## 7. Viết API trả về account role

Do ở phía giao diện chúng ta cần biết role của account đang đăng nhập để hiển thị giao diện phù hợp.

- Mình sẽ tạo 1 enum dùng chung để dễ thao tác:

**lti.interface.ts**

```typescript:lti.interface.ts
...
export enum USER_ROLE {
  INSTRUCTOR = 'Instructor',
  STUDENT = 'Student',
  ADMINISTRATOR = 'Administrator',
}
```

- Thêm route get role

**lti.controller.ts**

```typescript:lti.controller.ts
...
 @Get('roles')
  getRole(@LtiVariables('roles') roles: string[]) {
    return roles.includes(USER_ROLE.ADMINISTRATOR)
      ? USER_ROLE.ADMINISTRATOR
      : roles.includes(USER_ROLE.INSTRUCTOR)
      ? USER_ROLE.INSTRUCTOR
      : USER_ROLE.STUDENT;
  }
```

- Kiểm tra kết quả:

![image.png](https://images.viblo.asia/b6ecc069-d1fa-409f-adea-ae12e52d3ac1.png)

Vậy là chúng ta đã cài đặt xong các API cần cho LTI, tiếp theo chúng ta sẽ viết giao diện cho LTI bằng ReactJS.

# Triển khai giao diện với ReactJS

Vì đây là phần code giao diện nên đối với mỗi bạn khác nhau sẽ có những cách triển khai khác nhau. Vì thế mình sẽ tập trung vào các phần quan trọng để các bạn có thể dựa vào đó mà triển khai giao diện của riêng mình. Hoặc các bạn có thể sử dụng source FE của mình [tại đây.](https://github.com/nntwelve/LTI-with-nestjs-and-reactjs.git) Hầu hết các phần trong source này mình sẽ triển khai với _hook_ và vẫn sẽ sử dụng Typescript, dưới đây là cấu trúc thư mục:
![image.png](https://images.viblo.asia/cea75c76-2a3e-420f-8716-c7f2b38dfedb.png)

## 1. Yêu cầu

Giao diện của chúng ta sẽ đáp ứng các yêu cầu cơ bản sau:

- Giao diện của Giáo viên sẽ hiển thị danh sách yêu cầu của tất cả sinh viên
- Giáo viên cũng có thể thực hiện thao tác _đồng ý_ hoặc _từ chối_ với các yêu cầu.
- Giao diện của Sinh viên chỉ có thể xem các yêu cầu mình đã tạo.
- Giao diện Sinh viên sẽ có nút tạo yêu cầu xin nghỉ học.

## 2. Setup môi trường dev

Đầu tiên khi vào bạn dev với lệnh `npm start`, do chúng ta chưa build vào serve bằng LTI Serve trong Canvas LMS nên khi gọi vào API sẽ bị báo lỗi **_NO_LTIK_OR_IDTOKEN_FOUND_**.

- Để giải quyết vấn đề này các bạn chỉ cần làm tương tự như lúc chúng ta test với API - truy cập vào LTI trên Canvas để lấy **ltik**:
  ![ltik.png](https://images.viblo.asia/8930d9f1-c4a5-4d8b-8a26-ab8c46cdfaa3.png)
- Sau đó gắn **ltik** vào query để chúng ta có thể lấy **ltik** từ đó cho vào config của **axios hook**. Việc này giúp chúng ta không cần mỗi lần request API phải lấy từ query ra nữa hay nói nôm na là Config 1 lần dùng cả dự án.

![image.png](https://images.viblo.asia/4e3d2074-d7b3-44f8-9a3b-9940f9eaa9af.png)

- Mình sẽ tạo 1 helper để lấy **ltik** từ query.

**lti.helper.ts**

```typescript:lti.helper.ts
export function getLtik() {
  const queries = window.location.search;
  const urlParams = new URLSearchParams(queries);
  const ltik = urlParams.get('ltik');
  return ltik;
}
```

- Chúng ta sẽ gắn vào _config axios hooks_ như sau:

**axios.config.ts**

```typescript:axios.config.ts
import Axios from 'axios';
import { configure } from 'axios-hooks';
import LRU from 'lru-cache';
import { AppConfig } from './app-config';
import { getLtik } from './helpers/lti.helper';

const axios = Axios.create({
  baseURL: AppConfig.apiBase,
});

const cache = new LRU({ max: 10 });

// request interceptor to add ltik to request headers
axios.interceptors.request.use(
  async (config) => {
    const ltik = getLtik();
    if (ltik) {
      config.params = {
        ...config.params,
        ltik,
      };
    }
    return config;
  },
  (error) => Promise.reject(error)
);

configure({ axios, cache });
```

## 3. Hiển thị giao diện theo account role

Sau khi chúng ta config ltik xong, sẽ gọi đến API account role để kiểm tra xem account đang đăng nhập là giáo viên hay sinh viên để có thể hiển thị giao diện phù hợp. Mình sẽ tạo hook lấy account role như sau:

**hooks/roles/useLtiRole.ts**

```typescript:hooks/roles/useLtiRole.ts
import useAxios from '../shared/useAxiosWrapper';

function useLtiRole() {
  return useAxios<string>({
    method: 'GET',
    url: `/lti/roles`,
  });
}

export default useLtiRole;
```

- Giao diện giáo viên sẽ có danh sách các yêu cầu và có thể đồng ý hoặc từ chối:

![image.png](https://images.viblo.asia/3ea8ef84-d8af-4062-ad1e-7be9af0ee399.png)

- Giao diện sinh viên sẽ hiển thị các yêu cầu của sinh viên đó và có thể thêm yêu cầu mới:

![image.png](https://images.viblo.asia/17b8ca24-dac4-427a-acd3-da019f7ce25d.png)

Khi đã giao diện đã hoàn chỉnh, việc cuối cùng của chúng ta là build từ ReactJS sang file static để serve bằng NestJS Server

# Build và serve ReactJS vào NestJS

## 1. Build ReactJS

Chúng ta dùng lệnh sau để build thành file static:

`npm run build`

Sau khi build hoàn tất chúng ta sẽ có được thư mục **_build_** như hình dưới:
![image.png](https://images.viblo.asia/68d727d5-84b3-498a-b517-d054940b9cd4.png)

## 2. Serve static file với NestJS Server

- Các bạn trở lại thư mục chứa server và cập nhật lại file app.module.ts để config **Serve Static Module**:

**app.module.ts**

```typescript:app.module.ts
...
@Module({
  imports: [
   ...
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '../client'),
    }),
    ...
```

- Tạo folder **_client_** cùng cấp với thư mục **_src_** sau đó copy các file trong thư mục **_build_** ở bước 1 vào trong đó.

![image.png](https://images.viblo.asia/62af6f7f-69b7-4f05-951a-004823afe59f.png)

- Tạo thêm 1 route ở controller để serve thư mục client chúng ta vừa tạo

**lti.controller.ts**

```typescript:lti.controller.ts
...
@Controller('lti')
export class LtiController {
  @Get()
  serveUI(@Req() req: Request, @Res() res: Response) {
    return req.res.sendFile(join(__dirname, '../../client'));
  }
  ...
```

- Restart lại server và mở LTI ở Canvas LMS để kiểm tra kết quả:

![image.png](https://images.viblo.asia/c16fa6cf-2e21-4f73-9ada-aa7f1871b75a.png)

# Kết luận

Như vậy mình và các bạn đã tạo xong 1 Custom LTI có đầy đủ cả ReactJS ở FE và NestJS ở BE. LTI trên của chúng ta có nhiều phần vẫn còn rất đơn sơ và chưa hoàn thiện, nên các bạn có thể tự mình nâng cấp để phát triển thành 1 LTI hoàn chỉnh.

Nếu các bạn gặp lỗi gì trong quá trình triển khai ví dụ trên hãy để lại comment hoặc inbox cho mình nhé. Cảm ơn các bạn đã giành thời gian đồng hành với mình trong Series này. Các bạn có thể tải về source code:

- [Front-end React](https://github.com/nntwelve/lti-with-nestjs-reactjs-fe)
- [NestJS Server](https://github.com/nntwelve/LTI-with-nestjs-and-reactjs)

Chúc các bạn mùng 10 Tết vui vẻ phát tài.

# Tài liệu tham khảo

- https://docs.nestjs.com/recipes/serve-static
- https://reactjs.org/
- https://github.com/Cvmcosta/ltijs
