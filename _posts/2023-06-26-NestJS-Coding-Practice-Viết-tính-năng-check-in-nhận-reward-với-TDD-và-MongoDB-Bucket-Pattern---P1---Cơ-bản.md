Đây là bài viết nằm trong Series NestJS thực chiến, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

----
Xin chào mọi người, để làm cho [Series NestJS thực chiến](https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA) của chúng ta trở nên thú vị và đúng nghĩa thực chiến hơn, mình sẽ thêm vào các bài viết mà theo mình các bạn có thể sẽ gặp trong các dự án sau này. Khác với các bài viết về các công nghệ sử dụng với NestJS, nội dung các bài dạng này mình sẽ cố gắng viết ngắn gọn và dễ hiểu để các bạn có thể nắm được ý chính và dễ dàng triển khai.   
# Đặt vấn đề
Chức năng check-in là một chủ đề linh động, do đó tùy theo từng dự án sẽ có các yêu cầu khác nhau. Ví dụ như:
* Sau khi check-in sẽ nhận quà ngay (phổ biến thường thấy trong game).
* Check-in liên tục bao nhiêu ngày sẽ nhận được thưởng.
* Check-in đến cuối tháng nhận thưởng.  
* ...

Trong phạm vi bài viết này chúng ta sẽ cùng code cho ví dụ thứ 3: người dùng check-in hằng ngày, sau đó vào cuối tháng nhận điểm thưởng dựa vào số ngày đã check-in trong tháng đó. Các yêu cầu chi tiết như sau:
* Ghi lại thông tin check-in bao gồm:
    * Ngày được check-in.
    * Số lần đăng nhập (yêu cầu này có phần liên quan tới log, ở dự án lớn các bạn nên tách ra riêng).
* Phải thông báo cho FE biết nếu ngày đang check-in là ngày nhận reward để giúp họ hiển thị thông báo cho người dùng.
* Ngày tính reward cho user sẽ có 2 trường hợp:
    * User check-in vào ngày cuối tháng và sẽ được nhận reward vào ngày đó.
    * *User check-in tháng trước đó nhưng đến cuối tháng họ không vào check-in để nhận, thì sang ngày bất kì trong tương lai khi check-in sẽ được nhận.*
* Thiết kế dữ liệu như thế nào để khi trả về cho FE hiển thị dễ nhất.

Sơ qua thì yêu cầu cũng khá đơn giản, tuy nhiên sẽ dễ gây sai sót nếu không xử lí đầy đủ các trường hợp có thể xảy ra, đặc biệt là ở yêu cầu mình in nghiêng. Ví dụ các bạn xử lý trường hợp này bằng cách: 

> "Chỉ kiểm tra ngày check-in gần nhất không thuộc tháng được check-in thì tính reward cho tháng check-in gần gần và tạo check-in cho tháng hiện tại".
 
Khi đó nếu user check-in vào ngày cuối cùng của tháng trước và đã nhận reward rồi thì khi họ check-in vào tháng tiếp theo sẽ lại nhận reward tháng trước đó tiếp do đó sẽ dẫn đến sai sót.
# Giải pháp
Trước khi bắt đầu, mình sẽ lấy source code có sẵn mà chúng ta đang dùng cho [Series này](https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA). Ở thời điểm hiện tại series đang ở [Bài thứ 6 về Unit Test](https://viblo.asia/p/setup-boilerplate-cho-du-an-nestjs-phan-61-unit-test-tat-ca-cac-thanh-phan-trong-nestjs-zOQJwnyy4MP) nên mình sẽ check out luôn từ branch của bài viết này, các bạn có thể tải về [tại đây](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-6-unit-test-with-jest-and-golevelup).

Source code hoàn chỉnh của bài viết này các bạn có thể tải về ở branch [daily-checkin-function](https://github.com/nntwelve/Boilerplate-NestJS/tree/daily-checkin-function).

## Flow chart 
Để đáp ứng các yêu cầu trên chúng ta sẽ nhiều cách như:
* Kiểm tra xem trước giờ có điểm danh hay chưa => Hôm nay đã điểm danh chưa => Có phải ngày cuối tháng không => Tháng trước đó nhận reward chưa => ...
* Kiểm tra hôm nay có phải ngày cuối tháng không => Trước giờ có điểm danh chưa => Hôm nay đã điểm danh chưa => Tháng trước đó nhận reward chưa => ...
* ...

Mình sẽ chọn cách đầu tiên cho bài viết này, flow chart chi tiết như sau:
![](https://images.viblo.asia/55e5fdf8-c75b-4eab-ac67-b213e74fc0b3.png)

Vấn đề xác thực và các lỗi liên quan thì chúng ta sẽ không bàn đến ở đây, mình liệt kê đủ để mọi người có cái nhìn tổng quan. Hình bên dưới mình sẽ lược bỏ để dễ nhìn:
![](https://images.viblo.asia/7c45d7e6-8e61-4e72-abf5-1c6e6f674c66.png)


Chú thích: flow sẽ bắt đầu bằng việc kiểm tra từ trước đến giờ user đã check-in chưa (case này cho trường hợp khi user vừa tạo tài khoản), có 2 case xảy ra:
* **Case 1**: **trước giờ chưa điểm danh**, thì sẽ có 2 case bên trong:
    * **Case 1.1**: *hôm nay là ngày cuối tháng* => nhận thưởng tháng này => check-in
    * **Case 1.2**: *hôm nay là ngày trong tháng* => check-in
* **Case 2**: **đã từng điểm danh**, thì sẽ có 2 case xảy ra:
    * **Case 2.1**: *hôm nay đã điểm danh* => tăng access amount
    * **Case 2.2**: *hôm nay chưa điểm danh*, tiếp tục có 2 case:
        * **Case 2.2.1**: **hôm nay là ngày cuối tháng**, sẽ có 2 case cần xử lí:
            * **Case 2.2.1.1**: *tháng trước đó chưa nhận thưởng* => nhận thưởng tháng trước đó => nhận thưởng tháng này => check-in
            * **Case 2.2.1.2**: *tháng trước đó đã nhận thưởng* => nhận thưởng tháng này => check-in
        * **Case 2.2.2**: **hôm nay là ngày trong tháng**, sẽ có 2 trường hợp:
            * **Case 2.2.2.1**: *tháng trước đó chưa nhận thưởng* => nhận của tháng trước đó => check-in
            * **Case 2.2.2.2**: *tháng trước đó đã nhận thưởng* => check-in

## Schema
Chúng ta sẽ thêm và cập nhật lại schema trong code như sau:
```typescript:src/modules/users/entities/daily-check-in.entity.ts
class DailyCheckIn {
	checked_date: Date; // Ngày check in

	access_amount: number; // Số lượng truy cập trong ngày

	eligible_for_reward: boolean; // Nếu là true thì là ngày nhận thưởng

	reward_days_count: number; // Số ngày đã check, dùng để hiển thị hoặc tính phần thưởng
}
```
Các property sẽ tương ứng với yêu cầu ở trên
* `checked_date`: căn cứ vào đây để kiểm tra hôm đó user đã check-in hay chưa.
* `eligible_for_reward`: giúp FE có thể biết được ngày hiện tại có phải là ngày nhận reward hay không.
* `reward_days_count`, `access_amount`: giúp FE hiển thị dữ liệu cho người dùng, tùy theo nhu cầu dự án các bạn có thể lược bỏ. 

```typescript:src/modules/users/entities/user.entity.ts
class User {
...
	daily_check_in?: DailyCheckIn[];
    
    last_check_in: Date // Ngày check-in gần nhất
    
    last_get_check_in_rewards: Date // Ngày nhận quà check-in gần nhất
}
...
```
Như mình vừa đề cập ở trên, nếu chỉ dùng ngày check-in gần nhất làm điều kiện thì vẫn chưa đủ, chúng ta cần thêm một property biểu thị ngày nhận thưởng gần nhất. 
* `last_check_in` sẽ được cập nhật mỗi khi user check-in.
* `last_get_check_in_rewards` sẽ được cập nhật mỗi khi user check-in và nhận được reward.

>Có thể các bạn sẽ thắc mắc nên dùng **embed** hay **reference** cho thông tin các ngày check-in vì chúng đều là quan hệ one-to-one. 
>
>Để giải đáp thì phải xem xét nhiều yếu tố như: document size, dữ liệu có thay đổi thường xuyên không, có truy xuất thường xuyên không. Ở trường hợp của chúng ta, dữ liệu điểm danh mỗi năm chỉ tối đa *366 record*, và với số lượng đó khi nhân lên cho 5 10 năm thì vẫn không thật sự nhiều. Cho nên ở dự án này thay vì tạo collection mới thì mình sẽ embed thẳng vào collection user, nếu dự án các bạn quan tâm đến hiệu năng thì có thể làm ngược lại hoặc backup dữ liệu các năm trước theo thời gian vào collection khác để tối ưu truy vấn.

Kết hợp flow chart và schema đã được thiết kế ở trên, chúng ta sẽ đến với phần tiếp  theo là triển khai code.
## Triển khai
Như đã nói đến ở tiêu đề mình sẽ triển khai theo hướng sử dụng **Test Driven Development** đã nói đến ở bài trước, chúng ta sẽ follow **3 rule** sau từ quyển **Clean Code** của **Uncle Bob**:
>1. You are not allowed to write any production code until you have first written a failing unit test.
>2. You are not allowed to write more of a unit test than is sufficient to fail—and not compiling is failing.
>3. You are not allowed to write more production code that is sufficient to pass the currently failing unit test

![image.png](https://images.viblo.asia/59307d1e-a966-43f9-8d9d-fbe6461d3388.png)

Chúng ta sẽ bắt đầu với trường hợp đầu tiên: "**Case 1.1: trước giờ chưa điểm danh và hôm nay là ngày cuối tháng**". 
```typescript:src/modules/users/test/user.service.spec.ts
import { Test } from '@nestjs/testing';
import { UsersService } from '../users.service';
import { createMock } from '@golevelup/ts-jest';
import { UserRolesService } from '@modules/user-roles/user-roles.service';
import { UsersRepositoryInterface } from '../interfaces/users.interface';
import { UsersRepository } from '@repositories/users.repository';
import { User } from '../entities/user.entity';
import { createUserStub } from './stubs/user.stub';
import { ConfigService } from '@nestjs/config';

jest.mock('../../user-roles/user-roles.service.ts');
describe('UserService', () => {
	let users_service: UsersService;
	let users_repository: UsersRepository;
	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
				UsersService,
				UserRolesService,
				{
					provide: 'UsersRepositoryInterface',
					useValue: createMock<UsersRepositoryInterface>(),
				},
                {
					provide: ConfigService,
					useValue: createMock<ConfigService>(),
				},
			],
		}).compile();
		users_service = module_ref.get(UsersService);
		users_repository = module_ref.get('UsersRepositoryInterface');
	});

	describe('Daily Check-in', () => {
		describe('Case 1: User never check-in before', () => {
			it('should receive reward if it is the last day of month (case 1.1)', async () => {
				// Arrange
				const user = createUserStub();
				const testing_date = '2023-01-31';
				const check_in_time = new Date(testing_date);

				// Act
				await users_service.updateDailyCheckIn(user, testing_date);
				// Assert
				expect(users_repository.update).toBeCalledWith(user._id, {
					point: user.point + 1,
					last_check_in: check_in_time,
					last_get_check_in_rewards: check_in_time,
					daily_check_in: [
						{
							eligible_for_reward: true,
							checked_date: check_in_time,
						},
					],
				});
			});
		});
	});
});
```
Với kiến thức đã đọc được ở bài trước, chúng ta sẽ vận dụng để khởi tạo các thông tin cơ bản sau:
* Tạo mock user bằng `createUserStub` và không có thông tin check-in.
* Vì case này đang test cho trường hợp cuối tháng nên chúng ta tạo mock ngày 31/01/2023. 
* Ở production user check-in chúng ta sẽ lấy ngày hiện tại, còn khi test chúng ta cần 1 biến để mock ngày đang check-in, do đó function `updateDailyCheckIn` sẽ có thêm param `testing_date`.
* Ở trường hợp này khi check-in thì sẽ cộng 1 điểm cho user tương ứng với số ngày check-in. Do đó chúng ta sẽ dùng `expect` để kiểm tra việc *service* gọi đến *repository* có chính xác hay không. Các tham số cho trường hợp này bao gồm điểm được tăng, ngày check-in, ngày nhận reward và thông tin check-in ngày hôm đó.

Chúng ta đã follow theo đúng như **rule 1** và **rule 2**, không viết code trước và không được phép viết thêm unit test hơn mức đủ để nó fail. Tiến hành run test với lệnh `npm run test:watch users.service` để kiểm tra:

![image.png](https://images.viblo.asia/0570a4f8-7b08-4786-8886-091fe769c168.png)

Lỗi không ngoài mong đợi vì chúng ta chưa defined method `updateDailyCheckIn` ở `UsersService`. Việc tiếp theo của chúng ta là làm cho test vừa fail ở trên pass, mình sẽ bắt đầu thêm method trên vào `UsersService`.
```typescript:src/modules/users/users.service.ts
...
export class UsersService extends BaseServiceAbstract<User> {
    constructor(
		...
		private readonly config_service: ConfigService
    )
    ...
    async updateDailyCheckIn(
		user: User,
		date_for_testing?: string,
	): Promise<User> {
        // Assuming with all the rewards of this API: corresponding to one check-in day will get one point
       const check_in_time =
				this.config_service.get('NODE_ENV') === 'production'
					? new Date()
					: new Date(date_for_testing);
        const last_day_of_check_in_month = new Date(
            check_in_time.getFullYear(),
            check_in_time.getMonth() + 1,
            0,
        ).getDate();
        const is_last_date_of_month =
            check_in_time.getDate() === last_day_of_check_in_month;
        // TH1
        if (!user.daily_check_in?.length) {
            // TH1.1:
            if (is_last_date_of_month) {
                return await this.users_repository.update(user._id.toString(), {
                    point: user.point + 1,
                    daily_check_in: [
                        { eligible_for_reward: true, checked_date: check_in_time },
                    ],
                    last_check_in: check_in_time,
                    last_get_check_in_rewards: check_in_time,
                });
            }
        }
	}
    ...
```
Giải thích:
* Đầu tiên chúng ta cần kiểm tra nếu không phải là môi trường production thì mới được chọn ngày check-in.
* Sử dụng `new Date` với giá trị truyền cho tham số date là 0 sẽ trả về ngày cuối cùng của tháng, chúng ta chỉ cần lấy ngày đó kiểm tra với ngày check-in là biết được có phải cuối tháng không.
* Một vài tham số mình không truyền vào như `access_amount`, `reward_days_count` vì đã có giá trị mặc định được tạo bởi option `default` ở schema, giúp cho code gắn gọn hơn. 
Tiến hành lưu lại và xem kết quả ở console.
![image.png](https://images.viblo.asia/35402afa-1faf-4cb1-afe0-ffa56f9520f3.png)

Chúng ta sẽ refactor lại cho code gọn gàng hơn theo như **rule 3**. Bước này là optional, nếu ở bước trên code của các bạn đã clean thì không cần phải chỉnh sửa.

```typescript:src/modules/users/users.service.ts
import { isLastDayOfMonth } from 'src/shared/helpers/date.helper';
...
export class UsersService extends BaseServiceAbstract<User> {
    ...
    async updateDailyCheckIn(
		user: User,
		date_for_testing?: string,
	): Promise<User> {
        // Assuming with all the rewards of this API: corresponding to one check-in day will get one point
         const check_in_time =
				this.config_service.get('NODE_ENV') === 'production'
					? new Date()
					: new Date(date_for_testing);
        const { daily_check_in } = user;
        // Case 1
        if (!daily_check_in?.length) {
            // Case 1.1:
            if (isLastDayOfMonth(check_in_time)) {
                return await this.users_repository.update(user._id.toString(), {
                    point: user.point + 1,
                    daily_check_in: [
                        { eligible_for_reward: true, checked_date: check_in_time },
                    ],
                    last_check_in: check_in_time,
                    last_get_check_in_rewards: check_in_time,
                });
            }
	}
    ...
```

Như vậy là đã xong 1 cycle của TDD, tương ứng với mỗi test case tiếp theo chúng ta cũng sẽ có cycle tương tự như vậy. Các bạn có thể tự mình triển khai cho **case 1.2**, mình sẽ viết tiếp cho trường hợp:  "**Case 2.2.1.1: User đã từng điểm danh trước đây, hôm nay chưa điểm danh và là ngày cuối tháng, tháng vừa rồi chưa nhận thưởng**".
```typescript:src/modules/users/test/user.service.spec.ts
...
    describe('Case 2: User has checked in before', () => {
        ...
        describe('Case 2.2: The day to check-in has not checked in yet', () => {
            describe('Case 2.2.1: The day to check-in is the last day of month', () => {
					it('should receive reward for both of month if the month before has not got reward (case 2.2.1.1)', async () => {
                        // Arrange
						const user = {
							...createUserStub(),
							daily_check_in: [
								{
									checked_date: new Date('2023-01-10'),
									eligible_for_reward: false,
									access_amount: 1,
									reward_days_count: 1,
								},
								{
									checked_date: new Date('2023-01-15'),
									eligible_for_reward: false,
									access_amount: 1,
									reward_days_count: 2,
								},
							],
							last_check_in: new Date('2023-01-15 07:00:00'),
							last_get_check_in_rewards: new Date('2022-12-31 09:00:00'),
						} as unknown as User;
						const testing_date = '2023-02-28 15:00:00';
						const check_in_date = new Date(testing_date);

						// Act
						await users_service.updateDailyCheckIn(user, testing_date);

						// Assert
						expect(users_repository.update).toBeCalledWith(
							user._id.toString(),
							{
								last_check_in: check_in_date,
								last_get_check_in_rewards: check_in_date,
								point: user.point + 3,
								daily_check_in: [
									...user.daily_check_in,
									{
										checked_date: check_in_date,
										eligible_for_reward: true,
									},
								],
							},
						);
					});
                }
        }
```
Giải thích:
* Chúng ta sẽ tạo mock cho dữ liệu check-in tháng trước kèm với ngày đã nhận reward gần nhất và ngày check-in gần nhất.
* Tương ứng với 2 ngày của tháng trước và 1 ngày của tháng được check-in thì chúng ta expect user sẽ được tăng 3 point và cập nhật lại các ngày vừa nêu ở trên

Tương tự khi save chúng ta cũng sẽ gặp lỗi như ví dụ trước:

![image.png](https://images.viblo.asia/86935079-d8b4-4522-bee6-15042d34545b.png)

Nhưng lỗi lần này sẽ khác vì chúng ta đã define method `updateDailyCheckIn` ở ví dụ trên, thay vào đó là lỗi `Number of calls: 0` do method `update` của `users_repository` không được gọi. Nguyên nhân do logic code trong service chưa đáp ứng với thông tin tạo trong test case. Tiến hành cập nhật thêm logic ở `updateDailyCheckIn` cho test case vừa tạo:
```typescript:src/modules/users/users.service.ts
       ...
    async updateDailyCheckIn(...): Promise<User> {
        ...
        if (!daily_check_in?.length) { ... }
        const already_check_in_index = daily_check_in.findIndex(
				(check_in_data) =>
					check_in_data.checked_date.toDateString() ===
					check_in_time.toDateString(),
			);
			// TH 2.1
			if (already_check_in_index !== -1) { return; } // Lưu ý: theo thứ tự thì các bạn phải có logic cho phần này trước
			// TH 2.2
			// TH 2.2.1
			if (isLastDayOfMonth(check_in_time)) {
				//TH 2.2.1.1
				if (
					(user.last_get_check_in_rewards.getMonth() !==
						user.last_check_in.getMonth() ||
						user.last_get_check_in_rewards.getFullYear() !==
							user.last_check_in.getFullYear()) &&
					(user.last_check_in.getFullYear() !== check_in_time.getFullYear() ||
						user.last_check_in.getMonth() !== check_in_time.getMonth())
				) {
					const { previous_month_data, current_month_data } =
						daily_check_in.reduce(
							(result, check_in_data) => {
								if (
									check_in_data.checked_date.getFullYear() ===
										user.last_check_in.getFullYear() &&
									check_in_data.checked_date.getMonth() ===
										user.last_check_in.getMonth()
								) {
									return {
										...result,
										previous_month_data: [
											...result.previous_month_data,
											check_in_data,
										],
									};
								}
								if (
									check_in_data.checked_date.getFullYear() ===
										check_in_time.getFullYear() &&
									check_in_data.checked_date.getMonth() ===
										check_in_time.getMonth()
								) {
									return {
										...result,
										current_month_data: [
											...result.current_month_data,
											check_in_data,
										],
									};
								}
							},
							{
								previous_month_data: [],
								current_month_data: [],
							},
						);
					const previous_month_point = previous_month_data.length;
					const current_month_point = current_month_data.length + 1; // One more point for the day has just checked-in
					return await this.users_repository.update(user._id.toString(), {
						last_check_in: check_in_time,
						last_get_check_in_rewards: check_in_time,
						daily_check_in: [
							...daily_check_in,
							{
								eligible_for_reward: true,
								checked_date: check_in_time,
							},
						],
						point: user.point + previous_month_point + current_month_point,
					});
				}
			}
	}
    ...
```
Đây là phần dễ gây nhầm lẫn nếu bỏ sót mà mình đã nói ở đầu bài, chúng ta cần phải đảm bảo 2 điều bên dưới cho case này:
* **Ngày check-in gần nhất** và **ngày nhận reward gần nhất** không được trùng **tháng** với nhau, nếu trùng thì phải khác **năm**. Tránh trường hợp user đã nhận thưởng ngày 31/01 và sang tháng tiếp theo điểm danh thì tiếp tục nhận thêm phần thưởng của tháng 1 thêm lần nữa.
* **Ngày check-in gần nhất** và **ngày đang check-in** không được trùng **tháng** với nhau, nếu trùng thì phải khác **năm**. Điều kiện này giúp biết được user đã check-in qua tháng mới. 

Hai điều kiện trên sẽ bổ trợ lẫn nhau để hoàn thiện logic test case của chúng ta. Save và kiểm tra lại kết quả ở console:

![image.png](https://images.viblo.asia/e45ab7b5-9a57-431a-bbbb-94ac83cf01d7.png)

Code ở trên vẫn chưa clean lắm nên chúng ta có thể refactor lại với **rule 3** cho code gọn hơn:
```typescript:src/modules/users/users.service.ts
import { isDifferentMonthOrYear, isLastDayOfMonth, isTheMonthOfSameYear } from 'src/shared/helpers/date.helper';
    ...
    async updateDailyCheckIn(...): Promise<User> {
        ...
        if (!daily_check_in?.length) { ... }
        const already_check_in_index = daily_check_in.findIndex(
				(check_in_data) =>
					check_in_data.checked_date.toDateString() ===
					check_in_time.toDateString(),
			);
			// TH 2.1
			if (already_check_in_index !== -1) { return; } // Theo thứ tự thì các bạn sẽ có logic cho phần này trước
			// TH 2.2
			// TH 2.2.1
			if (isLastDayOfMonth(check_in_time)) {
				//TH 2.2.1.1
				if (
					isDifferentMonthOrYear(
						user.last_get_check_in_rewards,
						user.last_check_in,
					) &&
					isDifferentMonthOrYear(user.last_check_in, check_in_time)
				) {
					const { previous_month_data, current_month_data } =
						daily_check_in.reduce(
							(result, check_in_data) => {
								if (isTheMonthOfSameYear(check_in_data.checked_date, user.last_check_in)) {
									return {
										...result,
										previous_month_data: [
											...result.previous_month_data,
											check_in_data,
										],
									};
								}
								if (
                                    isTheMonthOfSameYear(check_in_data.checked_date, check_in_time)) {
                                        return {
                                            ...result,
                                            current_month_data: [
                                                ...result.current_month_data,
                                                check_in_data,
                                            ],
                                        };
								}
							},
							{
								previous_month_data: [],
								current_month_data: [],
							},
						);
					const previous_month_point = previous_month_data.length;
					const current_month_point = current_month_data.length + 1; // One more point for the day has just checked-in
					return await this.users_repository.update(user._id.toString(), {
						last_check_in: check_in_time,
						last_get_check_in_rewards: check_in_time,
						daily_check_in: [
							...daily_check_in,
							{
								eligible_for_reward: true,
								checked_date: check_in_time,
							},
						],
						point: user.point + previous_month_point + current_month_point,
					});
				}
			}
	}
    ...
```

Ở trên là trường hợp mình thấy khó nhằn nhất trong yêu cầu của chúng ta, các trường hợp còn lại các bạn có thể tự mình làm để luyện tập code với TDD. Khi làm xong các bạn có thể kiểm tra lại với code trên repo, mình đã viết đầy đủ cho các trường hợp đã nêu ra.

>**Lưu ý: sau khi viết logic cho service để test case pass, chúng ta nên test lại thủ công một lần nữa để tránh trường hợp có những sai sót do test case của chúng ta chưa bao quát hết.**

Đến đây thì chúng ta đã đáp ứng được các yêu cầu cơ bản của đề bài, chúng ta đã có API check-in chỉ cần tạo thêm API lấy ra thông tin check-in hoặc đơn giản hơn chỉ cần get bằng API `users/profile` hoặc `users/:id`. Bao nhiêu đây thì mình nghĩ đã đủ cho dự án nhỏ không cần chú trọng quá nhiều vào performance và scale up. 

Đi sâu hơn một tí, nếu như bạn nào để ý sẽ thấy, hiện tại chúng ta đang trả về toàn bộ dữ liệu check-in `daily_check_in`, nhưng đại đa số trường hợp người dùng chỉ cần dữ liệu trong tháng hiện tại, chỉ một số ít trường hợp mới cần xem lịch sử check-in trước đó. 

Do đó nếu chúng ta trả về tất cả như vậy sẽ gây ảnh hưởng đến hiệu năng API vì payload trả về các dữ liệu không cần thiết. Vì vậy mục tiêu của chúng ta sẽ thay đổi như sau:
* Chỉ trả về dữ liệu tháng check-in của tháng hiện tại cho người dùng khi họ đăng nhập.
* Có API giúp user kiểm tra dữ liệu check-in các tháng trước đó.

> Việc thay đổi yêu cầu từ phía khách hàng diễn ra rất thường xuyên trong dự án thực tế, do đó chúng ta phải nên chuẩn bị trước các tình huống có thể xảy ra thay đổi để sau này có gặp có thể đáp ứng được, tránh mất nhiều thời gian.



# Kết luận
Vậy là chúng ta đã đi qua phần 1 của bài practice đầu tiên trong Series NestJS thực chiến, phần sau chúng ta sẽ cùng viết thêm nhiều test case nữa kết hợp với MongoDB Bucket Pattern để đáp ứng yêu cầu mới mà chúng ta vừa liệt kê ở trên.

Hy vọng phần này có thể giúp các bạn hiểu thêm đôi nét về cách code với TDD, cũng như biết được cách viết một API check-in như thế nào để tránh mắc phải các sai sót không mong muốn. Bài viết trên dựa vào kiến thức cá nhân của mình nên bạn nào có cách hay hơn hãy comment góp ý để mình và mọi người có thể học hỏi. 

Cảm ơn các bạn đã giành thời gian đọc bài viết, hẹn gặp lại các bạn ở phần 2 của bài viết.

# Tài liệu tham khảo
* Trilon (no date) Applying test driven development with NestJS - Trilon Consulting, Trilon. Available at: https://trilon.io/blog/tdd-with-nestjs (Accessed: 26 June 2023). 
* lan, P.T.P. (2023) Kiến thức cơ bản về TDD( test driven development ), Viblo. Available at: https://viblo.asia/p/kien-thuc-co-ban-ve-tdd-test-driven-development-Do754AWLKM6 (Accessed: 26 June 2023).