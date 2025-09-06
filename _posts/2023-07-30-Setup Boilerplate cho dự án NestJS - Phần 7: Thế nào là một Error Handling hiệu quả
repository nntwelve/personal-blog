Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---
# Đặt vấn đề
![](https://images.viblo.asia/6b9fa443-b636-43c7-ac14-6c6bc4214a52.png)

**Error Handling** là một chủ đề không mới và luôn luôn hiện diện trong bất kì dự án nào mà chúng ta tham gia. Vì thế việc triển khai như thế nào để hiệu quả và bảo mật là điều chúng ta cần quan tâm. Thông thường chúng ta cần đáp ứng các yêu cầu sau:

* **(1)**. **Recognizable**: Phải giúp người sử dụng API **phân biệt được lỗi** giữa từ **client** và lỗi từ **server**: 
    * Lỗi từ client sẽ là các lỗi 4xx (unauthenticate, validation error,...): khi đó họ phải chỉnh sửa lại thông tin request.
    * Lỗi từ server là các lỗi 5xx(bad gateway, service unavailable): với các lỗi này người dùng có thể thử gọi lại mà không cần thay đổi gì. 
* **(2)**. **Give Context**: Lỗi trả về từ API phải bao gồm context để có thể dễ dàng tìm ra nguyên nhân nơi nó phát sinh để giải quyết. 
* **(3)**. **Human Readability**:  **(2)** là giúp cho BE fix error, còn phía FE chỉ cần một vài thông tin để kiểm soát lỗi nên chúng ta cần làm sao để khi nhìn vào, họ có thể biết được cơ bản lỗi đó là gì.
* **(4)**. **High security**: Khi triển khai môi trường production, các thông tin lỗi trả về phải đảm bảo các vấn đề bảo mật. Thường chỉ là dạng dictionary, không được chứa bất cứ thông tin gì về context.

# Thông tin package
Các bạn có thể tải về toàn bộ source code của phần này [tại đây](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-7-error-handling-and-serialize-response).
# Triển khai
Để bắt đầu chúng ta sẽ viết một **GlobalExceptionFilter** để override lại default của NestJS, ở đây chúng ta sẽ chuẩn hóa lại các thông tin response để đảm bảo các yêu cầu ở phần trên.
```typescript:src/exception-filters/global-exception.filter.ts
import {
	ArgumentsHost,
	Catch,
	ExceptionFilter,
	HttpException,
} from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Response } from 'express';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
	constructor(private readonly config_service: ConfigService) {}
	catch(exception: any, host: ArgumentsHost) {
		const ctx = host.switchToHttp();
		const response = ctx.getResponse<Response>();

		const status =
			exception instanceof HttpException ? exception.getStatus() : 500;

		const message =
			exception instanceof HttpException
				? exception.message
				: 'Internal server error';
		response.status(status).json({
			statusCode: status,
			message,
			error:
				this.config_service.get('NODE_ENV') !== 'production'
					? {
							response: exception.response,
							stack: exception.stack,
					  }
					: null,
		});
	}
}
```
Giải thích:
* Để đáp ứng yêu cầu **(1)** chúng ta sẽ kiểm tra `exception` catch được có phải là `HttpException` không. Nếu đúng thì là thường là lỗi từ client, còn lại là của server hoặc các third-party package.
* `response.status(status)...` chỗ này có nhiều cách triển khai tùy vào team dev, có team sẽ luôn trả về 200 và chỉ trả về 500 khi lỗi server, có team thì luôn trả về 200 và lỗi thì trả về 400 và họ đều có lý do riêng của mình. Phần này là tùy thuộc vào mỗi người, còn mình không làm cách đó mà sẽ trả đúng status để tránh khi có lỗi phía FE phải check 2 lần mới biết được message lỗi là gì. 
* `...json({ statusCode: status, message, ...})` với các lỗi từ phía client chúng ta sẽ hiển thị ra message, ở phía dưới chúng ta sẽ nói rõ hơn nội dung của message. Còn về phần statusCode thì mình chỉ dùng cho trường hợp FE cần, các bạn có thể bỏ cũng được.
* Các thông tin về stack trace của lỗi hoặc response chi tiết sẽ được ẩn ở môi trường production **(4)**. 

Để sử dụng chúng ta sẽ thêm vào `AppModule` (do chúng ta có inject `ConfigService` nên không thể dùng trong file main.ts):
```typescript:src/app.module.ts
import { APP_FILTER } from '@nestjs/core';
import { GlobalExceptionFilter } from './exception-filters/global-exception.filter';
...
@Module({
	...
	providers: [
		AppService,
		{
			provide: APP_FILTER,
			useClass: GlobalExceptionFilter,
		},
	],
})
export class AppModule {}
```

Chúng ta đã xong với yêu cầu **(1)** và **(4)**, tiếp theo để làm cho error chúng ta trở nên đầy đủ context hơn mình sẽ tạo ra một dictionary chứa danh sách các lỗi bằng message code để làm ví dụ.
```typescript:src/constraints/error-dictionary.constraint.ts
export enum ERRORS_DICTIONARY {
	// AUTH
	EMAIL_EXISTED = 'ATH_0091',
	WRONG_CREDENTIALS = 'ATH_0001',
	CONTENT_NOT_MATCH = 'ATH_0002',
	UNAUTHORIZED_EXCEPTION = 'ATH_0011',

	// TOPIC
	TOPIC_NOT_FOUND = 'TOP_0041',

	// USER
	USER_NOT_FOUND = 'USR_0041',

	// CLASS VALIDATOR
	VALIDATION_ERROR = 'CVL_0001',
}
```

> Việc quy định error code có định dạng như thế nào tùy thuộc vào các thành viên trong team thống nhất, làm sao khi nhìn vào các bạn có thể biết ngay lỗi đó thuộc về module nào và loại lỗi là gì. Ví dụ ở đây ATH_0091: ATH = Auth, 009 = 409 và 1 dùng để phân biệt giữa các lỗi 409 của module Auth (conflic email, username,...). 

Khi sử dụng chúng ta sẽ kết hợp với các sub-class của `HttpException` để `GlobalExceptionFilter` có thể nhận biết đó là `instanceof HttpException`:
```typescript:src/modules/auth/auth.service.ts
import { ERRORS_DICTIONARY } from 'src/constraints/error-dictionary.constraint';
...
export class AuthService {
    ...
    async getAuthenticatedUser(email: string, password: string): Promise<User> {
		try {
			...
		} catch (error) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.WRONG_CREDENTIALS,
			});
		}
	}
```
Có thể thấy ở phía môi trường dev, codebase chúng ta sử dụng rất tường minh, khi nhìn vào là biết ngay là lỗi gì - có thể chi tiết hơn nữa `WRONG_CREDENTIALS` thành `WRONG_EMAIL_PASS_CREDENTIALS` để phân tách giữa các loại của nó. 

Khi đăng nhập với thông tin không hợp lệ kết quả sẽ hiển thị như sau:

![image.png](https://images.viblo.asia/9f87bb2f-3f80-49cd-9e10-1f7a496a7615.png)

Đứng ở phía BE khi nhìn vào response, chúng ta đã có thể biết được lỗi đó nằm ở đâu trong code dựa vào stack trace, kết hợp với error code `ATH_0001` chúng ta biết được đó là lỗi do chưa đăng nhập nên không có quyền truy cập. Như vậy chúng ta tiếp tục thỏa mãn được yêu cầu **(2)**.

Tuy nhiên ở phương diện FE, khi nhìn vào response trên họ sẽ không biết ngay lỗi là gì mà phải vào xem chú thích để tra dựa vào error code, việc đó rất bất tiện. Để khắc phục chúng ta cần làm gì đó để cho nó có tính **Human Readability** hơn, đây cũng là yêu cầu **(3)**.
```typescript:src/modules/auth/auth.service.ts
import { ERRORS_DICTIONARY } from 'src/constraints/error-dictionary.constraint';
...
export class AuthService {
    ...
    async getAuthenticatedUser(email: string, password: string): Promise<User> {
		try {
			...
		} catch (error) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.WRONG_CREDENTIALS,
				details: 'Wrong credentials!!',
			});
		}
	}
```
Bằng cách thêm vào property `details` chúng ta có thể diễn giải được nội dung lỗi chi tiết hơn, khi đó FE hoặc bất kì bên nào sử dụng API của chúng ta của có thể nắm bắt được cơ bản của vấn đề. 

Kết quả cuối cùng chúng ta mong đợi như sau:

![](https://images.viblo.asia/ced3396f-c72d-40d0-8abd-fe8e190f6895.png)

Nếu ở môi trường production thì sẽ như sau:

![image.png](https://images.viblo.asia/2f3ac3c1-d564-4a12-992e-87f0855efd5b.png)

Có thể thấy được, việc chuẩn bị Error Handling cho dự án trước khi bắt tay vào code sẽ giúp ít cho chúng ta cũng như người sử dụng API rất nhiều. Hạn chế tối đa được các tình huống khi những thành viên khác trong team sử dụng API gặp lỗi và phải liên hệ cho chúng ta vì không biết được đó là lỗi gì.

# Kết luận
Tuy bài viết này khá ngắn gọn nhưng chúng ta đã bao quát được khá nhiều về việc xử lý lỗi trong dự án NestJS và triển khai các giải pháp nhằm đáp ứng các yêu cầu quan trọng như phân biệt lỗi client/server, cung cấp context, đảm bảo đọc được cho người đọc lỗi và bảo mật thông tin. Việc chuẩn bị quy trình xử lý lỗi từ đầu sẽ giúp giảm thiểu các tình huống lỗi và tăng tính ổn định của ứng dụng, đồng thời cung cấp trải nghiệm tốt hơn cho người dùng API.

Hẹn gặp lại các bạn vào các bài viết tiếp theo. Cảm ơn các bạn đã giành thời gian đọc bài viết. 
# Tài liệu tham khảo
* Fronczak, S. (2023) Web api error handling: How to make debugging easier, Stackify. Available at: https://stackify.com/web-api-error-handling/ (Accessed: 30 July 2023). 
* Sandoval, K. (2017) Best practices for API error handling: Nordic apis |, Nordic APIs. Available at: https://nordicapis.com/best-practices-api-error-handling/ (Accessed: 30 July 2023). 
* PostgreSQL error codes (2021) PostgreSQL Documentation. Available at: https://www.postgresql.org/docs/9.6/errcodes-appendix.html (Accessed: 30 July 2023). 
* Documentation: Nestjs - a progressive node.js framework (no date) NestJS. Available at: https://docs.nestjs.com/exception-filters (Accessed: 30 July 2023).