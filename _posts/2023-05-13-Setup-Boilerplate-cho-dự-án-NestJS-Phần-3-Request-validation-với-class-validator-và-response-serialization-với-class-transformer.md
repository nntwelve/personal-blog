---
layout: post
title: 'Setup Boilerplate cho dự án NestJS - Phần 3: Request validation với class-validator và response serialization với class-transformer'
date: 2023-05-13 17:27:12 -0000
categories: NestJS
---

Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---

# Đặt vấn đề

Hiện nay việc xử lý dữ liệu user gửi lên API và dữ liệu trả về từ API luôn luôn là một việc không thể thiếu trong quá trình phát triển dự án, hôm nay chúng ta sẽ cùng tìm hiểu 2 package support rất tốt cho **NestJS** ở lĩnh vực này. Đó chính là **class-validator** và **class-transformer**, cả hai thư viện này giúp cho việc xác thực và chuyển đổi dữ liệu trở nên dễ dàng hơn, tiết kiệm thời gian và giảm thiểu các lỗi liên quan đến dữ liệu.

Sử dụng **class-validator** giúp chúng ta:

- Dễ dàng xác thực dữ liệu đầu vào từ người dùng hoặc các yêu cầu API.
- Tiết kiệm thời gian và giảm thiểu các lỗi liên quan đến dữ liệu không hợp lệ.
- Tự động xác định các lỗi xảy ra và trả về thông báo lỗi cụ thể cho client.
- Tùy chỉnh các quy tắc xác thực dữ liệu phù hợp với nhu cầu của dự án.

Tương tự **class-transformer** sẽ mang đến các công dụng:

- Dễ dàng chuyển đổi đối tượng lớp (class instance) sang đối tượng đơn giản (plain object) và ngược lại.
- Chuyển đổi định dạng của các thuộc tính (property) trong đối tượng.
- Xóa hoặc giữ lại các thuộc tính trong đối tượng.
- Tùy chỉnh quá trình chuyển đổi dữ liệu phù hợp với nhu cầu của dự án.

Để thấy rõ hơn công dụng của chúng mang lại, chúng ta sẽ lần lượt đi đến nội dung chi tiết ở các phần bên dưới. Mình sẽ cố gắng diễn tả chi tiết và đầy đủ nhất có thể thay vì các nội dung cơ bản để các bạn có thể tận dụng được tối đa các công năng của chúng.

# Thông tin package

- "class-validator": "^0.14.0"
- "class-transformer": "^0.5.1"

Các bạn có thể tải về toàn bộ source code của phần này [ở đây](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-2-mongoose-class-validator-class-transformer)

# 1. Class validator

**class-validator** cung cấp các decorator để thêm validation cho các thuộc tính (properties) của thông tin gửi lên từ request payload. Ví dụ, nếu chúng ta muốn đảm bảo rằng property _email_ phải có định dạng hợp lệ, chúng ta có thể sử dụng decorator `@IsEmail()` của **class-validator**. Thường chúng ta sẽ dùng validate request (params, query, body payload, ...) để đảm bảo dữ liệu user gửi lên là hợp lệ.

> Với params và query thì mình thường dùng với Pipe dạng Parse\*.

## 1.1 Cài đặt

Quá trình validation sẽ sử dụng **ValidationPipe** của **NestJS**, và nó cần sự kết hợp của **class-validator** và **class-transfomer** nên chúng ta phải cài đặt cả 2 package này.

`npm i --save class-validator class-transformer`

Mình sẽ sử dụng cho toàn bộ ứng dụng nên sẽ apply vào application level hay **Global Pipe**. Bạn nào chưa đọc về **Request Lifecycle** có thể xem lại bài viết trước của mình [ở đây.](https://viblo.asia/p/cach-request-lifecycle-hoat-dong-trong-nestjs-y3RL1awpLao)

```typescript:main.ts
import { Logger, ValidationPipe } from '@nestjs/common';
...

async function bootstrap() {
	...
	app.useGlobalPipes(new ValidationPipe()) // <== Thêm vào đây
	await app.listen(config_service.get('PORT'), () =>
		logger.log(`Application running on port ${config_service.get('PORT')}`),
	);
}
bootstrap();
```

Có nhiều option có thể truyền vào tham số của ValidationPipe, tuy nhiên dưới đây là những option mình hay sử dụng trong dự án:

- `whitelist`: nếu là true, sẽ loại bỏ các property không được liệt kê với **class-validator.**
- `forbidNonWhitelisted`: nếu là true thì thay vì bỏ qua bởi `whitelist` sẽ trả về lỗi
- `skipMissingProperties`: nếu property truyền vào không tồn tại hoặc có giá trị `undefined/null` sẽ bỏ qua validate cho property đó.
- `groups`: một số schema sẽ có hơn 1 loại validation, groups giúp chúng ta phân loại, thực thi cho từng group khác nhau. Option này thường được dùng cho các Pipe bên trong **Global Pipe**.

## 1.2 Validate object

Chúng ta sẽ bắt đầu bằng việc validate một object đơn giản, tiến hành thêm các decorator cho `CreateUserDto` để validate các property:

```typescript:src/modules/users/dto/create-user.dto.ts
import {
	IsEmail,
	IsNotEmpty,
	IsStrongPassword,
	MaxLength,
} from 'class-validator';

export class CreateUserDto {
	@IsNotEmpty() // Bắt buộc phải gửi lên
	@MaxLength(50) // Tối đa 50 ký tự
	first_name: string;

	@IsNotEmpty()
	@MaxLength(50)
	last_name: string;

	@IsNotEmpty()
	@MaxLength(50)
	@IsEmail() // Phải là định dạng email
	email: string;

	@IsNotEmpty()
	@MaxLength(50)
	username: string;

	@IsNotEmpty()
	@IsStrongPassword() // Password phải đủ độ mạnh
	password: string;

    @IsOptional() // Không bắt buộc phải gửi lên
	address?: CreateAddressDto[];
}
```

> **Lưu ý**: Trước khi tiếp tục mình muốn lưu ý các bạn một trường hợp mà khi làm việc với NestJS rất có thể các bạn sẽ mắc phải. Đó là khi generate code từ NestCLI sẽ có dạng `this.users_repository.create(create_user_dto)` để lưu data từ `CreateUserDto` vào database, việc này dẫn đến một cơ hội để ai đó có thể lợi dụng. Ví dụ trong **User** schema có property `point` là số tiền user nạp vào, nếu chúng ta dùng luôn như trên mà không chỉnh sửa gì hoặc chỉnh sửa và dùng _spread operator_ `this.users_repository.create({password: hash_password, ...create_user_dto})` thì khi ai đó gửi kèm `point: 99999999` sẽ dẫn đến số point của user sẽ thành con số mà họ gửi lên (mình lấy ví dụ để minh họa thôi chứ thực tế chắc ít ai để logic code tạo user như vậy 😅). Có nhiều cách để giải quyết vấn đề trên như:
>
> - Truyền từng property hợp lệ vào function `create` thay vì toàn bộ property trong `CreateUserDto`: cách này ổn nhưng nếu schema nào có nhiều property sẽ làm code dài dòng.
> - Dùng delete với object để delete các property không hợp lệ sau đó mới truyền vào function `create`: tương ứng với n các property cần xóa sẽ có n lệnh delete được tạo ra.
> - Tạo **Factory Pattern** để tạo các property hợp lệ: cách này hay và làm cho code rõ ràng dễ đọc hơn, tuy nhiên phải mất thời gian viết thêm Factory cho từng schema.
> - Dùng desctructuring (`const {point, role, ...process_create_user_dto} = create_user_dto`): cách này là cải tiến của delete ở trên, khá ổn và có thể sử dụng. Nhưng đôi khi nếu không cẩn thận để sót property cần xóa thì cũng như không.
> - Sử dụng option `whitelist` của `ValidationPipe`: đây là cách hiện tại mình đang dùng, chỉ cần thêm option đó vào, các property nào không có decorator của **class-validator** sẽ bị bỏ qua. Đảm bảo chỉ những property trong `CreateUserDto` mới có thể đi qua `ValidationPipe`. Nếu không có decorator nào phù hợp cho property chúng ta có thể dùng decorator `@Allow`.

- Gọi **POST** http://localhost:3333/users để tạo user và kiểm tra kết quả
  - Các property không hợp lệ sẽ trả về lỗi
    ![image.png](https://images.viblo.asia/43ca7fee-1961-4705-bcf8-dacab21e2c5f.png)
  - Thử với trường hợp dữ liệu hợp lệ và kèm với `point`. Có thể thấy dù đã gửi lên nhưng sẽ bị `ValidationPipe` loại ra
    ![image.png](https://images.viblo.asia/498c7a09-f8dc-40cb-8015-e1192e8daff9.png)

Tiếp theo chúng ta sẽ đến với `UpdateUserDto`

```typescript:src/modules/users/dto/update-user.dto.ts
import { OmitType, PartialType } from '@nestjs/swagger';
import {
	IsDateString,
	IsEnum,
	IsOptional,
	IsPhoneNumber,
	MaxLength,
} from 'class-validator';
import { GENDER } from '../entities/user.entity';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(
	OmitType(CreateUserDto, ['email', 'password', 'username']),
) {
	@IsOptional()
	@IsPhoneNumber()
	phone_number?: string;

	@IsOptional()
	@IsDateString()
	date_of_birth?: Date;

	@IsOptional()
	@IsEnum(GENDER)
	gender?: string;

	@IsOptional()
	@MaxLength(200)
	headline?: string;
}
```

- Vì là update đa số chỉ cần cập nhật một số property nên chúng ta sẽ dùng `PartialType` chuyển các property của `CreateUserDto` về trạng thái _optional_. Tuy nhiên đôi khi chúng ta cũng sẽ cần loại bỏ một số property tùy theo logic nghiệp vụ như username, email, password để tránh không cho user cập nhật trong các API nhất định, do đó mình dùng `OmitType` để loại bỏ các property đó.
  > Các method hữu ích khác như: `PickType(DTOObject, ['field_name'] as const)`, `IntersectionType(DTO1, DTO2)`
- **Lưu ý**: mình dùng mapped types từ `@nestjs/swagger` chứ không dùng của `@nestjs/mapped-types` vì trong series này các bài tiếp theo mình sẽ dùng Swagger cho API docs và theo khuyến nghị từ tài liệu của NestJS "_Therefore, if you used `@nestjs/mapped-types` (instead of an appropriate one, either `@nestjs/swagger` or `@nestjs/graphql` depending on the type of your app), you may face various, undocumented side-effects._". Nên để có thể sử dụng các bạn cài đặt package này giúp mình:
  - Cài đặt **@nestjs/swagger**: `npm install --save @nestjs/swagger`
  - Chúng ta cũng cần thêm vào `"plugins": ["@nestjs/swagger"]` trong **nest-cli.json** do khác với **@nestjs/mapped-types**, mặc định `PartialType` của **@nestjs/swagger** không làm cho các property _optional_ ( xem thêm [ở đây](https://github.com/nestjs/swagger/issues/1043) ).
    ```json:nest-cli.json
    {
       "$schema": "https://json.schemastore.org/nest-cli",
       "collection": "@nestjs/schematics",
       "sourceRoot": "src",
       "compilerOptions": {
           "deleteOutDir": true,
           "plugins": ["@nestjs/swagger"] // <=== Thêm vào đây
       }
    }
    ```
- Gọi đến API **PATCH** http://localhost:3333/users/:id để cập nhật user và kiểm tra kết quả
  - Với dữ liệu không hợp lệ
    ![image.png](https://images.viblo.asia/122ac9d2-20d0-43ca-845b-ccb702a4d75f.png)
  - Dữ liệu hợp lệ thì các trường được thêm vào sẽ được cập nhật. Có thể thấy chúng ta cập nhật được cả field `first_name` từ `CreateUserDto`.
    ![image.png](https://images.viblo.asia/b5bb5678-45c7-4620-952e-89d4bc559141.png)

## 1.3 Validate nested object

Chúng ta sẽ đến với một trường hợp cũng rất hay gặp phải, payload gửi đi có bao gồm _nested object_, khi đó chúng ta cũng cần đảm bảo giá trị gửi lên bên trong _nested object_ phải được validate. Ví dụ cụ thể với property `address` bên trong user mà chúng ta đã dùng để minh họa quan hệ one-to-one ở bài viết trước.

- Đầu tiên chúng ta cập nhật lại `CreateAddressDto` để validate cho các property truyền vào khi cần tạo address. Cách validation cũng tương tự như các module trên, chỉ có `postal_code` chúng ta sẽ dùng method `IsPostalCode` để đảm bảo tính thực tế.

```typescript:src/modules/users/dto/create-address.dto.ts
import {
	IsNotEmpty,
	IsNumber,
	IsOptional,
	IsPostalCode,
	MaxLength,
	MinLength,
} from 'class-validator';

export class CreateAddressDto {
	@IsOptional()
	@MinLength(2)
	@MaxLength(120)
	street?: string;

	@IsNotEmpty()
	@MinLength(2)
	@MaxLength(50)
	state: string;

	@IsNotEmpty()
	@MinLength(2)
	@MaxLength(50)
	city: string;

	@IsOptional()
	@IsNumber()
	@IsPostalCode('US')
	postal_code: number;

	@IsNotEmpty()
	@MinLength(2)
	@MaxLength(50)
	country: string;
}
```

- Sau đó thêm `CreateAddressDto` vào `CreateUserDto` với property address

```typescript:src/modules/users/dto/create-user.dto.ts
import { CreateAddressDto } from './create-address.dto';
import { Type } from 'class-transformer';
...
export class CreateUserDto {
	...
	@IsOptional()
	@ValidateNested()
	@Type(() => CreateAddressDto)
	address?: CreateAddressDto;
}
```

- Giải thích:
  - `ValidateNested`: đầu tiên để validate nested object chúng ta cần dùng decorator này, giúp class-validator đi đến các object bên trong.
  - `Type`: `ValidateNested` thôi thì vẫn chưa đủ, dữ liệu address được gửi lên ban đầu ở dạng plain object vì thế cần được transform về instance của `CreateAddressDto` để **class-validator** có thể validate. `Type` được import từ package **class-transformer** mà phần sau chúng ta sẽ tìm hiểu.

> Do đó chỉ cần thiếu một trong 2 decorator trên thì validate cho property đó sẽ bị bỏ qua.

- Tiến hành test thử:
  - Thiếu các property bắt buộc sẽ báo lỗi:
    ![image.png](https://images.viblo.asia/1e1ef8b0-fdd2-49f5-809c-822406f90206.png)
  - Không truyền address trong lúc tạo thì sẽ bỏ qua validate cho address do chúng ta dùng `IsOptional`:
    ![image.png](https://images.viblo.asia/b90edabe-1008-44d7-9fb5-b0c2d755dec7.png)
  - Trường hợp tắt 1 trong 2 decorator `ValidateNested` hoặc `Type` thì dù gửi address lên nhưng validate cho property của address vẫn sẽ bị bỏ qua, vì thế chúng ta gặp validate ở **mongoose**.
    ![](https://images.viblo.asia/b1d0cdb3-8a04-4dc5-ad48-5f75c98f6b90.gif)

## 1.4 Validate Array

Trong trường hợp các bạn muốn validate các item bên trong **Array**, chẳng hạn các item phải thuộc enum nào đó . Ví dụ chúng ta sẽ thêm property `interested_languages` cho User schema để biểu thị các ngon ngữ mà user hứng thú. Khi tạo user sẽ gửi các ngôn ngữ đó với payload.

- Thêm property `interested_languages` vào **user.entity.ts**

```typescript:src/modules/users/entities/user.entity.ts
export enum LANGUAGES {
	ENGLISH = 'English',
	FRENCH = 'French',
	JAPANESE = 'Japanese',
	KOREAN = 'Korean',
	SPANISH = 'Spanish',
}
...
    @Prop({
		type: [String],
		enum: LANGUAGES,
	})
	interested_languages: LANGUAGES[];
    ...
```

- Cập nhật `CreateUserDto`, đảm bảo nếu user gửi lên `interested_languages` thì mảng phải có ít nhất 1 phần tử và các phần tử mảng phải thuộc enum `TOPIC`

```typescript:src/modules/users/dto/create-user.dto.ts
import { LANGUAGES } from '../entities/user.entity';
...
export class CreateUserDto {
    @IsOptional() // Không bắt buộc
	@IsArray() // Nếu có dữ liệu phải là dạng array
	@ArrayMinSize(1) // Array phải có tối thiểu 1 phần tử
	@IsEnum(LANGUAGES, { each: true }) // Các phần tử của array phải thuộc enum LANGUAGES
	interested_languages: LANGUAGES[];
    ...
```

- Giải thích:
  - `IsOptional`: vì chúng ta không bắt buộc user phải gửi address nên nếu không có thì sẽ bỏ qua không cần các trigger các decorator bên dưới.
  - `IsArray`: bắt buộc data gửi lên phải là array
  - `ArrayMinSize`: để tránh trường hợp user gửi mảng rỗng ( trong ví dụ của chúng ta không cần check cái này cũng được, nhưng trong một vài trường hợp thực tế phải đảm bảo mảng gửi lên không được rỗng )
  - `IsEnum( each: true )`: option each giúp validate từng phần tử trong mảng

## Validate Array Object

Chúng ta đến với trường hợp phức tạp hơn khi item bên trong Array là Object và bắt buộc cần phải validate. Ví dụ có yêu cầu từ phía khách hàng cần thay đổi cho phép user lưu nhiều address, khi này chúng ta sẽ chỉnh sửa lại:

- Chỉnh sửa lại property address trong User Schema thành Array

```typescript:src/modules/users/entities/user.entity.ts
...
    @Prop({
		type: [
			{
				type: AddressSchema,
			},
		],
	})
	address: Address[];
    ...
```

- Cập nhật `CreateUserDto`, đảm bảo nếu user gửi lên address thì mảng phải có ít nhất 1 phần tử và các phần tử mảng phải thuộc validate với `CreateAddressDto`

```typescript:
...
export class CreateUserDto {
    ...
    @IsOptional()
	@IsArray()
	@ArrayMinSize(1)
	@ValidateNested({ each: true })
	@Type(() => CreateAddressDto)
	address?: CreateAddressDto[];
    ...
```

- Giải thích: chúng ta sẽ dùng kết hợp giữa validate nested object và validate array để giải quyết trường hợp này. Tất nhiên các bạn không được bỏ sót `ValidateNested` hoặc `Type` như đã nói ở trên.

# 2. Serialization với class-transformer

Trong NestJS, **class-transformer** thường được sử dụng để thực hiện việc chuyển đổi đối tượng lớp (class instances) sang đối tượng đơn giản (plain objects) và ngược lại, giúp thao tác với dữ liệu trở nên dễ dàng cũng như điều chỉnh lại reponse phù hợp trước khi trả về cho người dùng. Cụ thể các

## 2.1 Cài đặt

Chúng ta đã cài đặt class-transformer ở trên cùng với class-validator.

## 2.2 Cách sử dụng

Ví dụ ở đây mình sẽ thêm property `stripe_customer_id` để sau này dùng cho bài viết thanh toán với _Stripe_, sau đó sử dụng decorator `@Exclude` loại bỏ property đó khỏi các response.

```typescript:src/modules/users/entities/user.entity.ts
import { Exclude } from 'class-transformer';
...
    @Prop({
		default: 'cus_mock_id',
	})
	@Exclude()
	stripe_customer_id: string;
```

Tạo lại user xem property `stripe_customer_id` có bị loại bỏ chưa.
![](https://images.viblo.asia/eccbcc7b-a1cb-47af-9457-5c650a77fea1.png)

- Có thể thấy vẫn chưa như chúng ta mong đợi, chúng ta cần thêm một bước là dùng **Interceptors** nữa mới có thể apply logic trên. Ở đây mình sẽ apply cho toàn bộ các API trong user module nên dùng `ClassSerializerInterceptor` cho **Controller Interceptor**.

```typescript:src/modules/users/users.controller.ts
import { UseInterceptors, ClassSerializerInterceptor } from '@nestjs/common';
@UseInterceptors(ClassSerializerInterceptor)
export class UsersController {
...
```

- Thử gọi API get lại user chúng ta vừa tạo
  ![](https://images.viblo.asia/d5f5b05e-3141-4ccd-8a60-a27a3d18fef5.png)
- Lần này không những property `stripe_customer_id` không mất mà còn kéo theo sự xuất hiện của hàng loạt property khác của MongoDB. Nguyên nhân không phải do chúng ta sai, mà là do response của MongoDB không tương thích với cách mà **class-transformer** response (nếu các bạn dùng các hệ quản trị cơ sở dữ liệu khác thì sẽ không bị vấn đề này).
- Để giải quyết vấn đề, chúng ta cần custom lại `ClassSerializerInterceptor` để đồng nhất cách response giữa 2 package trên.

```typescript:src/interceptors/mongoose-class-serializer.interceptor.ts
import {
	ClassSerializerInterceptor,
	PlainLiteralObject,
	Type,
} from '@nestjs/common';
import { ClassTransformOptions, plainToClass } from 'class-transformer';
import { Document } from 'mongoose';

function MongooseClassSerializerInterceptor(
	classToIntercept: Type,
): typeof ClassSerializerInterceptor {
	return class Interceptor extends ClassSerializerInterceptor {
		private changePlainObjectToClass(document: PlainLiteralObject) {
			if (!(document instanceof Document)) {
				return document;
			}
			return plainToClass(classToIntercept, document.toJSON());
		}

		private prepareResponse(
			response:
				| PlainLiteralObject
				| PlainLiteralObject[]
				| { items: PlainLiteralObject[]; count: number },
		) {
			if (!Array.isArray(response) && response?.items) {
				const items = this.prepareResponse(response.items);
				return {
					count: response.count,
					items,
				};
			}

			if (Array.isArray(response)) {
				return response.map(this.changePlainObjectToClass);
			}

			return this.changePlainObjectToClass(response);
		}

		serialize(
			response: PlainLiteralObject | PlainLiteralObject[],
			options: ClassTransformOptions,
		) {
			return super.serialize(this.prepareResponse(response), options);
		}
	};
}

export default MongooseClassSerializerInterceptor;
```

- Giải thích:
  - Chúng ta sẽ tạo ra 1 **HoC** với mục đích nhận vào schema.
  - Sau đó chúng ta return về custom class kế thừa `ClassSerializerInterceptor` để overide lại method `serialize` của nó.
  - Các bạn chú ý để method `prepareResponse`, đây là nơi chúng ta kiểm tra trước khi serialize response.
    - Ở dòng `if (!Array.isArray(response) && response?.items)` mình dùng để xử lý cho method `findAll`. Chúng ta gọi đệ quy bên trong dùng cho trường hợp schema có nested object bên trong (ví dụ user bên trong flash-card).
    - Dòng `if (Array.isArray(response))` dùng để trả về cho trường hợp response là array (ví dụ address bên trong user hoặc với `findAll` sau khi đệ quy ở if đầu tiên sẽ gặp dòng if này để xử lí tiếp).
  - Với method `changePlainObjectToClass` sẽ là nơi chính chúng ta giải quyết vấn đề không đồng nhất giữa mongoose và class-transfomer. `if (!(document instanceof Document))` nếu response không phải là Document của mongoose thì chúng ta sẽ trả về. Ngược lại nếu là Document chúng ta cần chuyển nó về JSON sau đó transfer sang Class của Schema đó.

```typescript:src/modules/users/users.controller.ts
import { UseInterceptors } from '@nestjs/common';
import { User } from './entities/user.entity';
import { MongooseClassSerializerInterceptor } from 'src/interceptors/mongoose-class-serializer.interceptor';
@UseInterceptors(MongooseClassSerializerInterceptor(User)) // Lưu ý không được quên User schema
export class UsersController {
...
```

- Thử lại xem kết quả đã hoạt động chưa

![](https://images.viblo.asia/c234f58b-02d8-4e24-afab-9bc6c7123094.png)

Kết quả đã thành công loại bỏ các property mình muốn khỏi response. Có nhiều option khác để các bạn có thể sử dụng như:

- Đối với các schema cần loại bỏ nhiều property thì chúng ta có thể dùng @Exclude sau đó dùng `@Expose` để hiển thị các property cần thiết

```typescript:src/modules/user-roles/entities/user-role.entity.ts
import { Exclude, Expose } from 'class-transformer';
...
@Exclude()
export class UserRole extends BaseEntity {
	@Prop({
		unique: true,
		default: USER_ROLE.USER,
		enum: USER_ROLE,
		required: true,
	})
	@Expose()
	name: USER_ROLE;

	@Prop()
	description: string;
}
...
```

- Trong trường hợp các bạn thiết kế các property private bắt đầu với prefix "_" thì có thể dùng option như sau `excludePrefixes: ["_"]` để mặc định loại bỏ các property bắt đầu bằng "\_".

```typescript:src/interceptors/mongoose-class-serializer.interceptor.ts
...
export function MongooseClassSerializerInterceptor(
	classToIntercept: Type,
): typeof ClassSerializerInterceptor {
    return class Interceptor extends ClassSerializerInterceptor {
        private changePlainObjectToClass(document: PlainLiteralObject) {
            if (!(document instanceof Document)) { return document }
            return plainToClass(classToIntercept, document.toJSON(), { excludePrefixes: ['_'] });
        }
        ...
```

## 2.3 Các case thông dụng

### planToClass

Phương thức này chuyển đổi một javascript object thành instance của class cụ thể. Ở `changePlainObjectToClass` chúng ta đã chuyển Document về JSON sau đó mới dùng **planToClass** để chuyển lại instance của input class.

### Transform

Được dùng cho trường hợp khi gọi API users có bao gồm cả object của role, như vậy thì đôi khi dài dòng và không hay, nên chúng ta có thể chỉnh lại cho ngắn gọn hơn như bên dưới.

```typescript:src/modules/users/entities/user.entity.ts
import { Transform, Type } from 'class-transformer';
...
export class User extends BaseEntity {

...
    @Prop({
		type: mongoose.Schema.Types.ObjectId,
		ref: UserRole.name,
	})
	@Type(() => UserRole)
	@Transform((value) => value.obj.role?.name, { toClassOnly: true })
	role: UserRole;
```

Kết quả sau khi transform, property `role` chuyển từ `role: { name: "User" }` sang `role: "User"` như bên dưới
![image.png](https://images.viblo.asia/a5575889-3059-4b24-9df2-341b4892077f.png)

### Working with nested objects

Đây cũng là trường hợp được sử dụng thông dụng mà các bạn thường hay bắt gặp. Ví dụ, giả sử với property address của user, chúng ta muốn ẩn đi property `postal_code` đối với request gọi đến các API của user module. Nếu bình thường chúng ta chỉ dùng `@Exclude` thôi thì vẫn chưa đủ.

```typescript:src/modules/users/entities/address.entity.ts
import { Exclude } from 'class-transformer';
...
export class Address extends BaseEntity {
    ...
	@Prop({ required: false, minlength: 2, maxlength: 50 })
	@Exclude()
	postal_code?: number;
...
```

Khi tạo, cập nhật hoặc get user thì property `postal_code` vẫn được trả về trong `address`.

![image.png](https://images.viblo.asia/58a930d2-3040-4ddd-93fc-296334525889.png)

Nguyên nhân là do _"Since Typescript does not have good reflection abilities yet, we should implicitly specify what type of object each property contain"_. Để giải quyết vấn đề này chúng ta cần thêm vào decorator `@Type` cho property address trong user schema.

```typescript:src/modules/users/entities/user.entity.ts
import { Exclude, Expose, Type } from 'class-transformer';
...
export class User extends BaseEntity {
    @Prop({
		type: [
			{ type: AddressSchema },
		],
	})
	@Type(() => Address)
	address: Address[];
    ...
```

Thử lại với những gì vừa chỉnh sửa, có thể thấy property `postal_code` đã được ẩn đi.
![image.png](https://images.viblo.asia/8ec9d9af-59f6-4a12-ba07-672a0adc397d.png)

### Exposing properties with different names

Ví dụ chúng ta expose ra `getter fullName` để truy xuất tên đầy đủ của user, sau đó dùng option `name` để đổi lại tên theo ý chúng ta muốn

```typescript:src/modules/users/entities/user.entity.ts
...
export class User extends BaseEntity {
    ...
    @Expose({ name: 'full_name' })
	get fullName(): string {
		return `${this.first_name} ${this.last_name}`;
	}
    ...
```

Kết quả
![image.png](https://images.viblo.asia/581f283d-5213-47de-a4d2-094ac6d8165d.png)

**Lưu ý:** nếu dùng với các property thì phải sử dụng với option `toPlainOnly: true`, nếu không property sẽ biến mất thay vì được đổi tên.

- Ví dụ mình sẽ rename property email sang mail như bên dưới, property email sẽ bị mất trong response.

```typescript:src/modules/users/entities/user.entity.ts
import { Exclude, Expose, Type } from 'class-transformer';
...
export class User extends BaseEntity {
    @Prop({
		required: true,
		unique: true,
		match: /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/,
	})
	@Expose({ name: 'mail'})
	email: string;
    ...
```

![image.png](https://images.viblo.asia/27ce30c7-6aa3-45d6-addf-6f7f2ed0f672.png)

- Thêm vào option `toPlainOnly: true` để giải quyết vấn đề trên

```typescript:src/modules/users/entities/user.entity.ts
import { Exclude, Expose, Type } from 'class-transformer';
...
export class User extends BaseEntity {
    @Prop({
		required: true,
		unique: true,
		match: /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/,
	})
	@Expose({ name: 'mail', toPlainOnly: true })
	email: string;
    ...
```

![image.png](https://images.viblo.asia/8eabf6c2-66e2-43ec-8671-8ea948bcf96a.png)

> Tuy nhiên trong trường hợp nếu dùng `@Exclude` để loại bỏ toàn bộ property sau đó `@Expose` các property cần hiển thị thì không được dùng option name, sẽ gặp lỗi mất property như trên. Mình đang tìm nguyên nhân, các bạn nào hiểu rõ có thể comment giải thích giúp mình và mọi người nha. Ví dụ như bên dưới sẽ làm mất property name mặc dù chúng ta sử dụng `toPlainOnly`
>
> ```typescript:src/modules/user-roles/entities/user-role.entity.ts
> ...
> @Exclude()
> export class UserRole extends BaseEntity {
> 	@Prop({...})
> 	@Expose({ name: 'role', toPlainOnly: true })
> 	name: string;
> ...
> ```

### Skipping specific properties

Có 2 cách như chúng ta đã ví dụ ở trên

- Dùng `@Exclude` cho 1 property cụ thể để loại bỏ property đó.
- Dùng `@Exclude` cho toàn schema để loại bỏ tất cả các property, sau đó `@Expose` các property còn lại để hiển thị chúng ra.

### Skipping some prefixed properties

Chúng ta đã sử dụng `excludePrefixes` trong `MongooseClassSerializerInterceptor` để loại bỏ các property bắt đầu bằng "\_", các bạn có thể thêm vào các prefix khác trong array value của `excludePrefixes` để loại bỏ hàng loạt các property mình muốn.

Hoặc các bạn có thể dùng decorator `@SerializeOptions` với option `excludePrefixes` cho các API cụ thể để loại bỏ các property mình muốn dựa vào prefix. Ví dụ mình sẽ loại bỏ property bắt đầu bằng `first` và `last` trong API **findAllUsers**.

```typescript:src/modules/users/users.controller.ts
import { SerializeOptions } from '@nestjs/common';
...
export class UsersController {
    ...
    @SerializeOptions({
		excludePrefixes: ['first', 'last'],
	})
	@Get()
	findAll() { return this.users_service.findAll(); }
    ...
```

- Gọi http://localhost:3333/users, các property đã mất theo dự định của chúng ta.
  ![image.png](https://images.viblo.asia/e6e5c2c5-2bb4-4e7a-8b16-efc49429becb.png)
- Tuy nhiên ở đây có một vấn đề phát sinh, property `_id` đã biến mất. Chúng ta cần expose ra id để dùng cho các mục đích khác, chỉnh sửa lại `BaseEntity` để giải quyết vấn đề trên.

  ```typescript:src/modules/shared/base/base.entity.ts
  import { Prop } from '@nestjs/mongoose';
  import { Expose, Transform } from 'class-transformer';

  export class BaseEntity {
      @Expose()
      @Transform((value) => value.obj?._id?.toString(), { toClassOnly: true })
      id?: string;
      ...
  }
  ```

- Thử lại xem response đã có `id` như chúng ta mong muốn hay chưa
  ![image.png](https://images.viblo.asia/c6d61618-97ab-4c7b-a658-09e9117e3011.png)

# Kết luận

Trong bài viết này chúng ta đã thực hiện việc cài đặt và cấu hình thư viện **class-validator** và **class-transformer** trong ứng dụng **NestJS**. Tiếp theo, chúng ta đã tạo một DTO (Data Transfer Object) để **validate** các trường của request và kết hợp với class-validator để thực hiện validation. Sau đó, chúng ta đã sử dụng class-transformer để **serialize** đối tượng response trả về từ controller, giúp chúng ta có thể tuỳ chỉnh định dạng của response dựa trên các quy tắc chuyển đổi được định nghĩa sẵn. Cuối cùng, chúng ta đã tìm hiểu về cách sử dụng **ValidationPipe** để tự động xác thực các request và serialize các response trả về. Với ValidationPipe, chúng ta có thể tuỳ chỉnh các tùy chọn và quy tắc validation theo ý muốn.

Tuy nhiên, việc bảo mật ứng dụng không chỉ dừng lại ở việc validate request và serialize response. Trong bài viết tiếp theo, chúng ta sẽ tìm hiểu về cách thực hiện xác thực với JWT sử dụng thuật toán bất đối xứng dùng package **crypto** của NodeJS mới. Điều này giúp chúng ta đảm bảo tính an toàn cho các request được gửi đến từ client và tăng cường tính bảo mật cho ứng dụng của mình.

Bài viết mới này sẽ giúp bạn hiểu rõ hơn về JWT, cách sử dụng thuật toán bất đối xứng để mã hóa và giải mã token, cách áp dụng JWT trong NestJS để xác thực và bảo mật các request gửi đến từ client. Hãy tiếp tục theo dõi series của chúng ta để tìm hiểu thêm về chủ đề này.

# Tài liệu tham khảo

- Documentation: Nestjs - a progressive node.js framework (no date) NestJS. Available at: https://docs.nestjs.com/techniques/validation (Accessed: 12 May 2023).
- Typestack (no date) Typestack/class-validator: Decorator-based property validation for classes., GitHub. Available at: https://github.com/typestack/class-validator (Accessed: 12 May 2023).
- Typestack (no date a) Typestack/class-transformer: Decorator-based transformation, serialization, and deserialization between objects and classes., GitHub. Available at: https://github.com/typestack/class-transformer (Accessed: 12 May 2023).
