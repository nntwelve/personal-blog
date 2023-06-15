---
layout: post
title: 'Setup Boilerplate cho dự án NestJS - Phần 6.1: Unit test tất cả các thành phần trong NestJS'
date: 2023-06-04 17:27:12 -0000
categories: NestJS
---

Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---

# Đặt vấn đề

**Unit test** không còn quá xa lạ với những lập trình viên chúng ta. Tuy nhiên đôi khi với các bạn chưa quen hoặc mới tìm hiểu thì sẽ hơi khó nhằn vì không biết nên bắt đầu viết từ đâu và tổ chức như thế nào. Như mình khi mới tìm hiểu về nó cũng rất mơ hồ và mất nhiều thời gian để vận dụng. Vì thế ở bài viết này mình sẽ chia sẻ về quá trình viết unit test cho một module từ kinh nghiệm mà mình tích góp được, để phần nào giúp các bạn có cách tiếp cận dễ dàng hơn khi phải dùng nó trong dự án.

# Thông tin package

- "jest": "29.3.1"
- "ts-jest": "29.0.3"
- "@golevelup/ts-jest": "0.3.7"
- "mongodb-memory-server": "8.12.2"

Bài viết này khá dài sẽ được chia thành 2 bài viết và nội dùng gồm 3 phần **lý thuyết** ,**triển khai với Jest** và **các lưu ý để viết unit test hiệu quả** .Bạn nào đã quen và muốn bắt tay vào việc luôn thì có thể skip phần lý thuyết. Các bạn mới làm quen hoặc chưa hiểu rõ thì mình khuyên nên đọc qua để nắm vững lý thuyết.

Các bạn có thể tải về toàn bộ source code của phần này [tại đây](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-6-unit-test-with-jest).

# Lý thuyết

## 1. Unit test là gì?

Về mặt tổng quan, **Unit test** là quá trình kiểm tra tính đúng đắn của một phần nhỏ nhất trong chương trình, ví dụ như một method hoặc một class, thông qua việc chạy các test case được viết sẵn. Mục đích là đảm bảo rằng ứng dụng của chúng ta phát triển hoạt động đúng như mong đợi, và giúp tăng tính ổn định và độ tin cậy. Các bạn sẽ thường nghe đến các framework test như _Jest_, _Mocha_, _Karma_ hay _NUnit_, vv.

> Nói ngắn gọn là để kiểm tra chức năng chúng ta viết đúng như mong đợi hay không và sau này có chỉnh sửa chức năng đó thì có thể phát hiện ra lỗi kịp thời nếu có vấn đề phát sinh.

Cụ thể trong **NestJS**, Unit test là quá trình kiểm tra tính đúng đắn của một phần nhỏ nhất trong nó, ví dụ như một _service_, _controller_, _module_, _middleware_, _pipe_, _interceptor_,...

Lấy ví dụ với module user mà chúng ta thường hay dùng trong thực tế, trong ta có thể viết test case cho các thành phần sau:

- _Service_: để đảm bảo rằng method `create` sẽ tạo ra một đối tượng user mới, method `findById` sẽ tìm và trả về đúng đối tượng user theo id, method `update` sẽ cập nhật thông tin của đối tượng user, và method `delete` sẽ xóa đối tượng user.
- _Controller_: để đảm bảo rằng khi gọi một API từ client, controller sẽ xử lý yêu cầu và trả về đúng kết quả cho client.
- _Middleware_: để đảm bảo rằng middleware sẽ thực hiện chức năng được định nghĩa, chẳng hạn như kiểm tra xác thực người dùng, kiểm tra thông tin header, etc.
- _Pipe_: để đảm bảo rằng các giá trị đầu vào của API đều được class-validator hoặc các package validate khác kiểm tra và xác thực đúng cách.
- _Interceptor_: để đảm bảo rằng interceptor sẽ được gọi và thực thi các hành động được định nghĩa đúng cách.
- ...

## 2. Test Double là gì?

#### Khái niệm

Test double là một kỹ thuật được sử dụng trong unit testing để tạo ra các đối tượng giả để đại diện cho các đối tượng thật trong hệ thống, nhằm kiểm tra việc tương tác giữa chúng và giúp cho việc test có thể được thực hiện dễ dàng hơn.

#### Tại sao phải sử dụng

Các lý do để sử dụng test double trong unit test có thể kể đến như:

- Đảm bảo tính cách biệt trong việc kiểm thử một thành phần riêng lẻ của hệ thống, giúp giảm thiểu các ảnh hưởng không mong muốn đến các thành phần khác.
- Tạo điều kiện cho việc tái sử dụng các phần kiểm thử, giảm thiểu thời gian và chi phí cho việc phát triển hệ thống.
- Giúp tách biệt các thành phần của hệ thống để phát triển, test và triển khai một cách độc lập.

Ví dụ: các bạn đang viết một ứng dụng chat sử dụng API của ChatGPT, để lấy dữ liệu của câu trả lời bạn cần gọi API của ChatGPT. Tuy nhiên, mỗi lần gọi API sẽ phải trả phí và giới hạn số lần gọi. Việc này sẽ ảnh hưởng đến chi phí và độ ổn định của ứng dụng.

Trong trường hợp này, các bạn có thể sử dụng **Test Double** để tạo ra một đối tượng giả lập (mock object) của API bên ngoài đó, từ đó tách riêng phần logic lấy dữ liệu từ API và phần xử lý dữ liệu trong ứng dụng. Khi chạy các test, chúng ta sử dụng đối tượng giả lập này để lấy câu trả lời mà không phải gọi đến API của ChatGPT, giúp giảm chi phí và tăng độ ổn định của ứng dụng.

#### Các thành phần

Các thành phần của Test double bao gồm:

- **Stub**: giả lập các phản hồi của một đối tượng, cho phép kiểm tra các phần khác nhau của một method hoặc function.
- **Mock object**: giả lập các phản hồi của một đối tượng và kiểm tra các phần khác nhau của method hoặc function.
- **Fake object**: giả lập một đối tượng thực tế để kiểm tra các phần khác nhau của một method hoặc function.
- **Spy**: theo dõi các hoạt động của một đối tượng trong quá trình thực thi.
- **Dummy object**: một đối tượng giả lập đơn giản, thường được sử dụng để đưa dữ liệu vào method hoặc function.

## 3. AAA testing model là gì?

AAA model là một phương pháp đặt tên và phân chia cấu trúc của unit test, nó bao gồm 3 phần chính:

- **Arrange**: phần chuẩn bị dữ liệu, tạo các mock object, stub, fake object, ... để sẵn sàng cho việc test. Phần này nên được đặt ở đầu của test case.
- **Act**: phần thực hiện hành động cần được test, ví dụ như gọi một phương thức hay thay đổi một giá trị biến. Phần này cần được đặt ngay sau phần chuẩn bị dữ liệu.
- **Assert**: phần kiểm tra kết quả trả về từ phần Act, đảm bảo rằng hành động đã được thực hiện đúng và kết quả trả về đúng như mong đợi. Phần này nên được đặt cuối cùng của test case.

Cùng xem xét ví dụ bên dưới:

```typescript
function sum(a, b) {
  return a + b;
}

describe('sum function', () => {
  test('should return the correct sum', () => {
    // Arrange
    const a = 2;
    const b = 3;

    // Act
    const result = sum(a, b);

    // Assert
    expect(result).toBe(5);
  });
});
```

Giải thích:

- Arrange: Khởi tạo các giá trị cần thiết cho việc test. Trong ví dụ này, chúng ta khởi tạo giá trị a và b là 2 và 3.
- Act: Thực thi hàm cần test và lấy ra kết quả trả về. Trong ví dụ này, chúng ta gọi hàm sum với tham số a và b và lưu kết quả trả về vào biến result.
- Assert: So sánh kết quả trả về với giá trị mong đợi. Trong ví dụ này, chúng ta kiểm tra xem kết quả trả về của hàm sum có phải là 5 hay không bằng cách sử dụng expect của Jest.

Có thể thấy phương pháp AAA giúp cho test case được phân chia rõ ràng và dễ hiểu, giúp cho việc đọc và sửa lỗi test case dễ dàng hơn. Ngoài ra, phương pháp này cũng đảm bảo việc chuẩn bị và thực hiện các bước test case được độc lập với nhau, giúp tăng tính ổn định và độ tin cậy của unit test.

# Triển khai với Jest

**Jest** là một framework phổ biến để viết unit test cho javascript/typescript, đặc biệt là cho React, Node.js, và các ứng dụng khác. **Jest** cung cấp cho người dùng các công cụ để viết và chạy các test case, bao gồm cả các công cụ _mock_ và _spy_ để thực hiện _test double_, và một loạt các _assertions_ để kiểm tra kết quả của các test case.

## 1. Cài đặt và cấu hình Jest

Mặc định thì khi cài đặt **NestJS** đã có sẵn **Jest** nên chúng ta không cần phải cài thêm package. Việc cần làm đó là chỉnh lại một vài cấu hình do ở [Phần 1](https://viblo.asia/p/setup-boilerplate-cho-du-an-nestjs-phan-1-team-co-nhieu-thanh-vien-env-joi-husky-commitlint-prettier-dockerizing-EbNVQxG2LvR) chúng ta đã dùng **Path Alias** nên khi chạy lệnh `npm run test` sẽ gặp lỗi.

![image.png](https://images.viblo.asia/89049cce-3f26-46f8-b2e2-d177b560552e.png)

Các bạn chỉnh sửa phần config Jest trong file **package.json** như sau:

```json:package.json
...
"jest": {
    ...
    "moduleNameMapper": {
        "^src/(.*)$": "<rootDir>/$1",
        "^@modules/(.*)$": "<rootDir>/modules/$1",
        "^@config/(.*)$": "<rootDir>/configs/$1",
        "^@repositories/(.*)$": "<rootDir>/repositories/$1"
    }
}
```

Giải thích: các key bên trái sẽ tương ứng với các giá trị trong option `paths` mà chúng ta đã config ở file **ts.config.json**, giá trị bên phải sẽ giúp Jest hiểu được cấu trúc thư mục để truy xuất nội dung file.

## 2. Triển khai viết unit test

## 2.1 Tìm hiểu cách unit test được tạo từ NestCLI

Trước khi bắt tay vào viết unit test các bạn có để ý khi chúng ta tạo module với **NestCLI** thì nó sẽ tự sinh ra cho chúng ta 2 file unit test là _service_ và _controller_. Để giúp các bạn hiểu sâu hơn mình sẽ từng bước diễn giải từng bước cách mà nội dung trong các file này được hình thành để các bạn có thể hiểu được cách mà nó tạo ra để sau này nếu gặp lỗi các bạn có thể giải quyết được.

> Mình sẽ dùng auth service vì nó có nhiều logic hơn để chúng ta có thể lấy ví dụ

#### Tạo test cơ bản

Đầu tiên chúng ta sẽ tạm thời bỏ qua các quy tắc và tiến hành tạo test cho `AuthService` một cách sơ khai nhất. Viết như một file typescript bình thường:

```typescript:src/modules/auth/test/auth.service.spec.ts
...
describe('AuthService', function () {
	const auth_service = new AuthService(
		new ConfigService(),
		new UsersService(
			new UsersRepository(User as any),
			new UserRolesService(new UserRolesRepository(UserRole as any)),
		),
		new JwtService(),
	);
	it('should be defined', () => {
		expect(auth_service).toBeDefined();
	});
});
```

Giải thích:

- Cách đơn giản nhất để test chức năng cho `AuthService` class là tạo ra instance và từ đó gọi các method của nó để test. Chúng ta sẽ khởi tạo instance từ `AuthService` và truyền vào các tham số cần thiết.
- Chúng ta cũng cần phải tạo các instance cho các method bên trong các tham số của nó.
  Tiến hành chạy test cho các file **.spec** trong module auth với lệnh `npm run test:watch auth/test` (sử dụng regex để chỉ chạy các file bên trong folder `test` của folder `auth`). Có thể thấy từ hình bên dưới, test case kiểm tra việc tạo `auth_service` đã thành công.

![image.png](https://images.viblo.asia/83f48848-e097-4867-b598-ce7e0c3006b8.png)

> Ở đây chúng ta chưa bàn tới các method cần gọi đến Database, khi nào chúng ta tiến hành mock các respository thì sẽ nói đến sau.

#### Phân tách instance

Sau khi tạo được instance như trên thì chúng ta sẽ bắt đầu xét đến các quy tắc đầu tiên của unit test: **independent**. Có thể thấy ở trên nếu có nhiều test case thì toàn bộ chúng đều dùng chung một instance, dễ làm cho các test case bị ảnh hưởng lẫn nhau khi một trong số chúng tác động đến instance. Để tránh việc đó chúng ta dùng `beforeEach` để áp dụng việc tạo instance cho từng test case riêng, như tên của nó, callback sẽ được gọi lại với mỗi test case.

```typescript:src/modules/auth/test/auth.service.spec.ts
...
describe('AuthService', function () {
	let auth_service: AuthService;
	beforeEach(() => {
		auth_service = new AuthService(
			new ConfigService(),
			new UsersService(
				new UsersRepository(User as any),
				new UserRolesService(new UserRolesRepository(UserRole as any)),
			),
			new JwtService(),
		);
	});
	it('should be defined', () => {
		expect(auth_service).toBeDefined();
	});
});
```

#### Sử dụng testing module

Đến đây thì chúng ta đã tách biệt được instance cho các test case, tiếp theo khi nhìn vào constructor của `AuthService` thì các bạn sẽ thấy nó có quá nhiều dependency và chúng ta khởi tạo hàng loạt như vậy thì dễ gây thiếu sót, nhầm lẫn thứ tự và cấu trúc cũng không giống module của **Nest**. Chúng ta có thể cải thiện nó bằng cách dùng method `createTestingModule` từ package `@nestjs/test`, giúp chúng ta giả lập module của **Nest**.

```typescript:src/modules/auth/test/auth.service.spec.ts
...
describe('AuthService', function () {
	let auth_service: AuthService;
	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
                AuthService,
				ConfigService,
				JwtService,
				UsersService,
				UserRolesService,
				{
					provide: 'UsersRepositoryInterface',
					useClass: UsersRepository,
				},
				{
					provide: 'UserRolesRepositoryInterface',
					useClass: UserRolesRepository,
				},
            ],
		}).compile();
		auth_service = module_ref.get<AuthService>(AuthService);
	});
	it('should be defined', () => {
		expect(auth_service).toBeDefined();
	});
});
```

- Giải thích:
  - Config truyền vào `createTestingModule` sẽ tương tự như trong `AuthModule`, tuy nhiên mình sẽ chuyển tất cả các module được import về service vì **chúng ta đang viết _unit test_ chứ không phải _integration test_**.
  - Các **repository** được inject trong service theo tên interface cần dùng `provide` là tên interface kết hợp với `useClass` là class triển khai logic của interface đó.
  - Chúng ta sẽ dùng method `get` lấy ra service để dùng trong các test case.

> Đến đây khi các bạn so sánh với nội dung file được **NestCLI** tạo ra thì đã gần như là tương tự nhau, **đó chính là cách mà nội dung file test được NestCLI tạo ra.**

## 2.2 Viết các mock cơ bản

Tuy nhiên khi các bạn chạy lệnh `npm run test:watch auth/test` sẽ gặp lỗi như sau. Chúng ta sẽ làm tiếp thêm một vài bước để hoàn thiện luôn cho service này

![image.png](https://images.viblo.asia/650a95cd-c68c-4d65-87d3-ff8a60dbc4d5.png)

Đây là 1 lỗi khi hiển thị message lỗi của Jest, cách khắc phục rất đơn giản chúng ta chỉ cần thêm vào option `detectOpenHandles` trong lệnh thực thi Jest.

```json:package.json
    ...
    "scripts": {
        ...
        "test:watch": "jest --detectOpenHandles --watch",
        ...
```

Khi đó nội dung lỗi sẽ được hiển thị rõ ràng ra để chúng ta debug. Lỗi ở đây là do trong repository có sử dụng database model nhưng chưa config trong `createTestingModule`.

![image.png](https://images.viblo.asia/acb39f9e-0a10-4fe1-8c32-2da0ee1650f9.png)

### Mock database

Để giải quyết các vấn đề liên quan đến model trong mongoose khi test chúng ta có thể sử dụng method `getModelToken(model, connectionName)` để mock model.

> Method `getModelToken` nếu không truyền vào `connectionName` thì sẽ trả về tên model được truyền vào kết hợp với 'Model'. Ví dụ kết quả của `getModelToken(User.name)` sẽ là `UserModel`

Sau khi thêm vào các bạn xem lại log ở console sẽ không còn lỗi nữa.

```typescript:src/modules/auth/test/auth.service.spec.ts
import { getModelToken } from '@nestjs/mongoose';
...
describe('AuthService', function () {
	let auth_service: AuthService;
	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
				...
                // Thêm vào 2 config sau
				{
					provide: getModelToken(User.name), // trả về: UserModel
					useValue: {},
				},
				{
					provide: getModelToken(UserRole.name), // trả về: UserRoleModel
					useValue: {},
				},
			],
		}).compile();
		...
```

### Mock các dependency service

Chúng ta đã hoàn thành việc tách biệt với database, tiếp theo chúng ta cần tách logic của các service từ npm như `ConfigService` và `JwtService` vì chúng ta không nên để test case phụ thuộc vào kết quả của các service khác. Để triển khai mình sẽ tạo ra các mock cho các method trong 2 service trên. Tạo folder **mocks** và tạo file **config-service.mock.ts**.

```typescript:src/modules/auth/test/mocks/config-service.mock.ts
export const mockConfigService = {
	get(key: string) {
		switch (key) {
			case 'JWT_ACCESS_TOKEN_EXPIRATION_TIME':
				return '3600';
			case 'JWT_REFRESH_TOKEN_EXPIRATION_TIME':
				return '36000';
		}
	},
};
```

Mình cũng sẽ tạo folder **mocks** để gôm các dữ liệu mock dùng chung lại với nhau

```typescript:src/modules/auth/test/mocks/tokens.mock.ts
const mock_token = 'mock_token';
const mock_access_token = 'mock_access_token';
const mock_refresh_token = 'mock_refresh_token';

export { mock_token, mock_access_token, mock_refresh_token };
```

```typescript:src/modules/auth/test/mocks/jwt.mock.ts
import { mock_token } from '../test/mocks/tokens.mock';

export const mockJwtService = {
	sign: jest.fn().mockReturnValue(mock_token),
};
```

Sau đó cập nhật lại `ConfigService` và `JwtService` với các mock vừa tạo.

```typescript:src/modules/auth/auth.service.spec.ts
import { mockConfigService } from './mocks/config-service.mock';
import { mockJwtService } from './mocks/jwt.mock';
...
describe('AuthService', function () {
	let auth_service: AuthService;
	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
                ...
                // Thay thế ConfigService và JwtService với config sau
				{
					provide: ConfigService,
					useValue: mockedConfigService,
				},
				{
					provide: JwtService,
					useValue: mockedJwtService,
				},
			],
		}).compile();
		...
```

- Giải thích:
  - Chúng ta thay thế các method của `ConfigService` và `JwtService` được dùng trong `AuthService` bằng các mock để không bị phụ thuộc vào kết quả trả về từ các method của service đó. Nếu sau này có sử dụng thêm các method khác chúng ta chỉ cần vào các file mock để thêm vào.
  - Tương tự như với repository chúng ta cũng dùng `useValue` để apply các mock vừa tạo.

Service cuối cùng chúng ta cần mock cho module này là `UsersService`, ở đây mình sẽ tạo folder `__mocks__` sau đó tạo file **users.service.ts** bên trong folder `modules/users/__mocks__`:

> Việc dùng folder `__mocks__` giúp **Jest** kích hoạt tính năng **Manual Mock** giúp tự động replace các real service thành mock service. Các bạn có thể đọc thêm [ở đây](https://jestjs.io/docs/manual-mocks).

```typescript:src/modules/users/__mocks__/users.service.ts
import { userStub } from '../../auth/test/stubs/user.stub';

export const UsersService = jest.fn().mockReturnValue({
    findOneById: jest.fn().mockResolvedValue(userStub()),
	findOneByCondition: jest.fn().mockResolvedValue(userStub()),
	setCurrentRefreshToken: jest.fn().mockResolvedValue(true),
	findAll: jest.fn().mockResolvedValue([userStub()]),
	create: jest.fn().mockResolvedValue(userStub()),
    getUserByEmail: jest.fn().mockResolvedValue(userStub()),
	getUserWithRole: jest.fn().mockResolvedValue(userStub()),
	update: jest.fn().mockResolvedValue(userStub()),
	remove: jest.fn().mockResolvedValue(true),
});
```

Để sử dụng **Manual Mock** chúng ta sẽ dùng `jest.mock()` để replace service thực với mock chúng ta vừa tạo.

```typescript:src/modules/auth/auth.service.spec.ts
...
jest.mock('../../users/users.service');
describe('AuthService', function () {
	...
```

Bằng cách dùng trên `UsersService` sẽ tự động được Jest thay thế bằng mock chúng ta tạo. Vậy là chúng ta đã tìm hiểu qua các bước để có thể tạo thành một file test tương tự với nội dung được tạo từ **NestCLI** và cấu hình cơ bản trước khi viết các test case cụ thể. Phần tiếp theo chúng ta sẽ bắt tay vào viết test case.

## 2.3 Viết test case cho các thành phần trong NestJS

Phần này chúng ta sẽ cùng viết các test case cho các thành phần sau:

- **Service**
- **Controller**
- **Repository**
- **Model** (optional)
- **Guard**
- **Interceptor**
- **Middleware**
- **Pipe**

### Service

Chúng ta sẽ viết cho method `signUp` trong `AuthService` cho 2 trường hợp sau:

- User đăng ký thất bại do email bị trùng và báo lỗi `ConflictException`
- User đăng ký thành công với thông tin hợp lệ

Chúng ta sẽ viết test case cho 2 trường hợp trên, trước tiên tạo các stub dùng chung

```typescript:src/modules/users/test/stubs/user.stub.ts
import { UserRole } from '@modules/user-roles/entities/user-role.entity';
import { GENDER, User } from '@modules/users/entities/user.entity';

export const userStub = (): User => { // Tránh trường hợp object reference nên chúng ta dùng func
	return {
		_id: '643d0fb80a2f99f4151176c4',
		email: 'johndoe@example.com',
		first_name: 'John',
		last_name: 'Doe',
		password: 'strongestP@ssword',
		username: 'johndoe',
		gender: GENDER.MALE,
		role: 'admin' as unknown as UserRole,
		fullName: 'John Doe',
	};
};
```

Tiếp đến thêm vào test case với `describe` và `it`:

```typescript:src/modules/auth/auth.service.spec.ts
import { ConflictException } from '@nestjs/common';
import * as bcrypt from 'bcryptjs';
import { userStub } from '../../users/test/stubs/user.stub';
    ...
   describe('signUp', () => {
		it('should throw a ConflictException if user with email already exists', async () => {
			// Arrange
			jest
				.spyOn(users_service, 'findOneByCondition')
				.mockResolvedValueOnce(createUserStub());
			// Act && Assert
			await expect(auth_service.signUp(createUserStub())).rejects.toThrow(
				ConflictException,
			);
		});
		it('should successfully create and return a new user if email is not taken', async () => {
			// Arrange
			const user_stub = createUserStub();
			const mock_sign_up_dto: SignUpDto = {
				email: 'michaelsmith@example.com',
				first_name: 'Michael',
				last_name: 'Smith',
				password: '123le$$321',
			};
			jest
				.spyOn(users_service, 'findOneByCondition')
				.mockResolvedValueOnce(null);
			jest
				.spyOn(auth_service, 'generateAccessToken')
				.mockReturnValue(mock_access_token);
			jest
				.spyOn(auth_service, 'generateRefreshToken')
				.mockReturnValue(mock_refresh_token);
			jest
				.spyOn(bcrypt, 'hash')
				.mockImplementationOnce(() => mock_sign_up_dto.password);
			jest.spyOn(auth_service, 'storeRefreshToken');

			// Act
			const result = await auth_service.signUp(mock_sign_up_dto);

			// Assert
			expect(users_service.create).toHaveBeenCalledWith({
				...mock_sign_up_dto,
				username: expect.any(String),
			});
			expect(auth_service.generateAccessToken).toHaveBeenCalledWith({
				user_id: user_stub._id,
			});
			expect(auth_service.generateRefreshToken).toHaveBeenCalledWith({
				user_id: user_stub._id,
			});
			expect(auth_service.storeRefreshToken).toBeCalledWith(
				user_stub._id,
				mock_refresh_token,
			);
			expect(result).toEqual({
				access_token: mock_access_token,
				refresh_token: mock_refresh_token,
			});
		});
	});
```

> Với các trường hợp asynchronous code chúng ta cần dùng `expect` với `await` để catch lỗi. Xem thêm các cách khác [ở đây](https://jestjs.io/docs/asynchronous)

Giải thích:

- Tương ứng với mỗi trường hợp chúng ta sẽ biểu diễn với từng function `it`
- Với trường hợp user trùng email:
  - Ở phase _Arrange_ chúng ta sẽ tạo mock value cho kết quả trả về từ method `findOneByCondition` của `UserServices` để đảm bảo lỗi user exist.
  - Ở phase _Act_ và _Assert_ thì chúng ta dùng `expect` với `reject.toThrow` để bắt lỗi, và đảm bảo lỗi trả về phải là `ConflictException`
- Với trường hợp dữ liệu hợp lệ và tạo user thành công:
  - Ở phase _Arrange_:
    - Chúng ta tạo mock dữ liệu đăng ký hợp lệ, và để mô phỏng email được truyền vào chưa tồn tại chúng ta mock value của `findOneByCondition` là `null`.
    - Vì chúng ta có gọi `generateAccessToken` và `generateRefreshToken` trong method `signUp`, mặc dù là cùng nằm trong `AuthService` nhưng theo mình chúng ta vẫn không nên phụ thuộc vào 2 method đó. Thay vào đó chúng ta sẽ có test case riêng để test logic của 2 method này.
    - Chúng ta cũng cần mock response trả về từ **bcryptjs** để lát nữa dùng trong `expect` param truyền vào method `create` của `UsersService`.
  - Ở phase _Act_ tiến hành cho gọi method `signUp` của `AuthService` .
  - Ở phase _Assert_ sẽ so sánh kết quả trả về, cũng như các dữ liệu được truyền vào method của các service bên trong method `signUp` , để đảm bảo các dữ liệu được gửi đi và trả về đều theo như dự định. Lưu ý, ở đây mình có dùng `expect.any(String)` vì username của chúng ta là dynamic, các bạn có thể ứng dụng với các property có value dynamic (xem thêm [ở đây](https://jestjs.io/docs/expect#expectanyconstructor)).

Kiểm tra kết quả thu được ở console:
![image.png](https://images.viblo.asia/0bf79f04-7026-45de-b00c-c9dbfe4a44fd.png)

Có thể thấy kết quả đã thành công, mình sẽ thử thay đổi code trong `AuthService` để xem kết quả có fail hay không. Ví dụ chỉnh sửa lại một tí ở param truyền vào method `generateRefreshToken`

```typescript:src/modules/auth/auth.service.ts
...
    signUp(sign_up_dto: SignUpDto) {
        ...
        const refresh_token = this.generateRefreshToken({
            user_id: user._id.toString(),
            // @ts-ignore
            email: user.email,
        });
        ...
```

Kết quả ở console sẽ thay đổi:
![image.png](https://images.viblo.asia/c0abfc2f-1e29-4151-8103-030eef19fecf.png)

Có thể thấy nhờ có unit test mà chúng ta có thể tránh được các lỗi xảy ra khi có sự thay đổi trong code dẫn đến ảnh hưởng đến các chức năng đã có. Các test case còn lại của `AuthService` các bạn có thể tự viết hoặc tải về từ source code.

### Controller

Phần này mình sẽ hướng dẫn các bạn viết test case cho `AuthController`, vì đây là controller nên trước tiên chúng ta cũng sẽ tạo mock cho `AuthService`

```typescript:src/modules/auth/__mocks__/auth.service.ts
import { createUserStub } from '@modules/users/test/stubs/user.stub';
import { mock_access_token, mock_refresh_token } from '../test/mocks/tokens.mock';

export const AuthService = jest.fn().mockReturnValue({
	signUp: jest.fn().mockResolvedValue({
		access_token: mock_access_token,
		refresh_token: mock_refresh_token,
	}),
	signIn: jest.fn().mockResolvedValue({
		access_token: mock_access_token,
		refresh_token: mock_refresh_token,
	}),
	getAuthenticatedUser: jest.fn().mockResolvedValue(createUserStub()),
	getUserIfRefreshTokenMatched: jest.fn().mockRejectedValue(createUserStub()),
	generateAccessToken: jest.fn().mockReturnValue(mock_access_token),
	generateRefreshToken: jest.fn().mockReturnValue(mock_refresh_token),
});
```

Tiếp theo chúng ta cũng setup tương tự như service, dùng `jest.mock()` để Manual Mock `AuthService`:

```typescript:src/modules/auth/test/auth.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { AuthController } from '../auth.controller';
import { AuthService } from '../auth.service';

jest.mock('../auth.service.ts'); // <--- Ở đây
describe('AuthController', () => {
	let auth_controller: AuthController;

	beforeEach(async () => {
		const module: TestingModule = await Test.createTestingModule({
			controllers: [AuthController],
			providers: [AuthService],
		}).compile();

		auth_controller = module.get<AuthController>(AuthController);
	});

	it('should be defined', () => {
		expect(auth_controller).toBeDefined();
	});
});
```

Mình sẽ viết test case cho method `signIn` các method còn lại các bạn có thể tự viết để luyện tập hoặc tải về từ source code của mình.

```typescript:src/modules/auth/test/mocks/requests.mock.ts
import { createUserStub } from '@modules/users/test/stubs/user.stub';
import { RequestWithUser } from 'src/types/requests.type';

export const mock_request_with_user = {
	user: createUserStub(),
} as RequestWithUser;
```

```typescript:src/modules/auth/test/auth.controller.spec.ts
import { mock_access_token, mock_refresh_token } from './mocks/tokens.mock';
...
describe('AuthController', () => {
	...
    describe('signIn', () => {
		it('should sign in a user and return an access token', async () => {
			// Arrange

			// Act
			const response = await auth_controller.signIn(mock_request_with_user);

			// Assert
			expect(response).toEqual({
				access_token: mock_access_token,
				refresh_token: mock_refresh_token,
			});
		});
	});
```

Chạy lệnh `npm run test:watch auth/test/auth.controller` để xem kết quả:

![image.png](https://images.viblo.asia/ac11792d-922e-465f-9e82-c335f6dc7d73.png)

Khá là đơn giản đúng không, **tuy nhiên vẫn chưa đủ** nếu bạn nào tinh ý có thể thấy, method `signIn` ở controller được protect bởi `LocalAuthGuard`, nếu các bạn thử comment `@UseGuards(LocalAuthGuard)` lại thì test case vẫn pass, chúng ta cần phải xử lí luôn cả trường hợp này. Tiến hành tạo file theo cấu trúc **src/shared/test/utils.ts**

```typescript:src/shared/test/utils.ts
import { CanActivate } from '@nestjs/common';

/**
 * Checks whether a route or a Controller is protected with the specified Guard.
 * @param route is the route or Controller to be checked for the Guard.
 * @param guard_type is the type of the Guard, e.g. JwtAuthGuard.
 * @returns true if the specified Guard is applied.
 */
export function isGuarded(
	route: ((...args: any[]) => any) | (new (...args: any[]) => unknown),
	guard_type: new (...args: any[]) => CanActivate,
) {
	const guards: any[] = Reflect.getMetadata('__guards__', route);

	if (!guards) {
		throw Error(
			`Expected: ${route.name} to be protected with ${guard_type.name}\nReceived: No guard`,
		);
	}

	let found_guard = false;
	const guard_list: string[] = [];
	guards.forEach((guard) => {
		guard_list.push(guard.name);
		if (guard.name === guard_type.name) found_guard = true;
	});

	if (!found_guard) {
		throw Error(
			`Expected: ${route.name} to be protected with ${guard_type.name}\nReceived: only ${guard_list}`,
		);
	}
	return true;
}
```

Giải thích:

- Param `route` sẽ là method handler của controller hoặc chính controller.
- Param `guard_type` sẽ là guard mà chúng ta muốn kiểm tra.
- `Reflect.getMetadata('__guards__', route)` dùng để lấy ra các guard đang được gán cho `route`.
- Chúng ta sẽ kiểm tra `guard_type` có nằm trong danh sách guard lấy ra từ method handler hoặc controller không, sau đó sẽ giả lập kết quả tương tự như Jest để hiển thị ra console.

Sau cùng chúng ta chỉnh sửa lại test case để thêm vào trường hợp vừa nêu

```typescript:src/modules/auth/test/auth.controller.spec.ts
import { isGuarded } from 'src/shared/test/utils';
import { LocalAuthGuard } from '../guards/local.guard';
...
    describe('signIn', () => {
		it('should be protected with LocalAuthGuard', () => {
			expect(isGuarded(AuthController.prototype.signIn, LocalAuthGuard));
		});
		it('should sign in a user and return an access token', async () => {
			// Arrange

			// Act
			const response = await auth_controller.signIn(mock_request_with_user);

			// Assert
			expect(response).toEqual({
				access_token: mock_access_token,
				refresh_token: mock_refresh_token,
			});
		});
	});
...
```

Thử comment lại `@UseGuards(LocalAuthGuard)` để kiểm tra kết quả, có thể thấy test case đã fail và chỉ ra cho chúng ta lỗi. Các bạn có thể áp dụng biện pháp này cho các thành phần khác của Nest.

![image.png](https://images.viblo.asia/00743fe9-e905-4c96-9792-f7fad7d857c4.png)

**Phần 1** của bài viết đến đây là hết, hẹn các bạn ở **phần 2** để chúng ta cùng viết unit test cho các thành phần còn lại sau đó cùng tìm hiểu một vài lưu ý để có thể viết unit test hiệu quả.
