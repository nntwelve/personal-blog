Đây là bài viết nằm trong Series NestJS thực chiến, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

----
Xin chào các bạn ở [Phần 1 của bài viết](https://viblo.asia/p/nestjs-coding-practice-viet-tinh-nang-check-in-nhan-reward-voi-tdd-va-mongodb-bucket-pattern-p1-co-ban-gwd43Mj3LX9) chúng ta đã tìm hiểu các bước cơ bản xây dựng chức năng điểm danh và nhận reward vào mỗi cuối tháng. Hôm nay chúng ta sẽ đến với phần nâng cao hơn để tối ưu chức năng này bằng cách áp dụng **Bucket Pattern** trong MongoDB.

![](https://images.viblo.asia/8fc733f3-8d3c-4220-80f4-3e27f4c3b837.png)

# Đặt vấn đề
Ở phần cuối của bài viết trước mình đã nhắc đến một số vấn đề mà chúng ta cần phải giải quyết nếu muốn nâng cao performance và scale up. Cụ thể như sau:
* Hiện tại chúng ta đang trả về toàn bộ dữ liệu check-in `daily_check_in` trong khi đa số trường hợp user chỉ cần trong tháng hiện tại.
* Cung cấp cho user API xem lại lịch sử tùy theo filter (month, year,...) thay vì trả về toàn bộ dữ liệu check-in.

Các bạn có thể tải về toàn bộ source code của phần này [ở đây.](https://github.com/nntwelve/Boilerplate-NestJS/tree/daily-checkin-function)
# Giải pháp
## 1. Bucket Pattern là gì?
Nói một cách ngắn gọn thì Bucket Pattern là cách gôm nhóm các document tương tự lại với nhau để tổng hợp dễ dàng hơn. Pattern này thường được dùng trong **Internet of Things (IoT), Real-Time Analytics** hoặc **Time-Series data**. Các lợi ích mà nó mang lại có thể kể đến như:
* Giúp organize các group data cụ thể dễ dàng hơn.
* Tăng khả năng discover các xu hướng lịch sử hoặc đưa ra dự báo trong tương lai.
* Tối ưu hóa việc sử dụng bộ nhớ của MongoDB.

Tuy nhiên đi kèm với các lợi ích trên thì cũng sẽ có một vài điều chúng ta cần lưu tâm:
* Tăng độ phức tạp: triển khai Bucket pattern yêu cầu viết thêm logic bổ sung để xử lý việc phân phối và truy xuất dữ liệu trên nhiều document hoặc collection.
* Data inconsistency: việc xóa một record gây ra không đồng nhất dữ liệu. Ví dụ khi xóa một document thì sẽ có những bucket không đạt số lượng nhất định. 
> Rất may ở chức năng check-in chúng ta đang phát triển có thể bỏ qua được vấn đề này do không cần quan tâm số lượng bucket đủ hay không.
* Reduced query performance: trong một số trường hợp khi query của chúng ta cần kết hợp nhiều collection với nhau sẽ dẫn đến chậm hơn khi chỉ query từ một collection.
## 2. Áp dụng Bucket Pattern vào giải quyết vấn đề
Cụ thể trong chức năng của chúng ta, dữ liệu check-in thuộc dạng **Time-Series data** nên thay vì lưu tất cả trong property User model như chúng ta đã làm ở [Phần 1](https://viblo.asia/p/nestjs-coding-practice-viet-tinh-nang-check-in-nhan-reward-voi-tdd-va-mongodb-bucket-pattern-p1-co-ban-gwd43Mj3LX9), chúng ta có thể gôm nó lại (bucketing) theo các bucket như *năm*, *tháng* hoặc *tuần* để dễ quản lý hơn. 

Do yêu cầu chức năng của chúng ta là tính điểm theo tháng nên bucket *tháng* là hợp lý nhất trong trường hợp này. Dữ liệu sẽ được gôm lại theo dạng như sau:
```typescript
{
    _id: ObjectId('64b17877dae2badc43e47370'),
    deleted_at: null,
    user: ObjectId('6445b9d38095a7adc6514db9'),
    month_year: '7-2023',
    check_in_data: [
        {
            checked_date: ISODate('2023-07-14T00:00:00.000Z'),
            access_amount: 2,
            eligible_for_reward: false,
            reward_days_count: 1,
            _id: ObjectId('64b17877dae2badc43e47371')
        },
        {
            checked_date: ISODate('2023-07-15T00:00:00.000Z'),
            access_amount: 1,
            eligible_for_reward: false,
            reward_days_count: 2,
            _id: ObjectId('64b17877dae2badc43e47372')
        }
    ],
}
```
Như đã nói ở trên, mỗi document sẽ tương ứng với dữ liệu của **1 tháng** trong năm. Chi tiết như sau:
* Thông tin lưu sẽ bao gồm tháng và cả năm để tránh trùng.
* Mỗi document cũng sẽ có user id để kiểm tra xem là của user nào.
* `check_in_data` sẽ chứa thông tin các ngày được check-in trong tháng đó. Chúng ta vẫn giữ nguyên cấu trúc của mảng này tương tự như trong User model.

Toàn bộ dữ liệu check-in của user sẽ được tách ra khỏi `User` model và bucketing dữ liệu đó vào collection mới với tên `DailyCheckIn` model. `User` model chỉ lưu trữ check-in data của tháng đó, khi đó quá trình cập nhật hay lấy dữ liệu ra sẽ trở nên dễ dàng hơn. 

Ví dụ khi cần lấy check-in data của tháng 6 front-end chỉ cần gửi lên `month=6`, chúng ta chỉ cần gọi đến `DailyCheckIn` model với query `month_year: '6-2023'` là sẽ lấy ra được dữ liệu, tương tự khi lấy ra theo quý hoặc năm cũng vậy.

Sơ lược qua là như vậy, chúng ta sẽ cùng đến với phần triển khai để hiểu rõ hơn về Bucket pattern và practice thêm về TDD.
# Triển khai
Đầu tiên chúng ta sẽ tách check-in data từ User model ra thành collection `DailyCheckIn` như đã nói ở trên. Tiến hành dùng NestCLI để tạo module **daily-check-in** cho nhanh gọn.
```typescript:src/modules/daily-check-in/entities/daily-check-in.entity.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import mongoose, { HydratedDocument, ObjectId } from 'mongoose';

import { User } from '@modules/users/entities/user.entity';
import { CheckInData, CheckInDataSchema } from './check-in-data.entity';
import { BaseEntity } from '@modules/shared/base/base.entity';

export type DailyCheckInDocument = HydratedDocument<DailyCheckIn>;
@Schema({
	collection: 'daily-check-in',
})
export class DailyCheckIn extends BaseEntity {
	@Prop({
		required: true,
		ref: User.name,
		type: mongoose.Schema.Types.ObjectId,
	})
	user: User | ObjectId | string;

	@Prop({
		required: true,
	})
	month_year?: string;

	@Prop({
		type: [CheckInDataSchema],
		required: true,
	})
	check_in_data: CheckInData[];
}

export const DailyCheckInSchema = SchemaFactory.createForClass(DailyCheckIn);
```

Thông tin `CheckInData` sẽ không thay đổi: 
```typescript:src/modules/daily-check-in/entities/check-in-data.entity.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';

@Schema()
export class CheckInData {
	@Prop({
		default: new Date(),
		required: true,
	})
	checked_date?: Date; // Ngày check in

	@Prop({
		min: 1,
		required: true,
		default: 1,
	})
	access_amount?: number; // Số lượng truy cập trong ngày

	@Prop({
		required: true,
	})
	eligible_for_reward: boolean; // Nếu là true thì là ngày nhận thưởng

	@Prop({
		min: 1,
		required: true,
		default: 1,
	})
	reward_days_count?: number; // Số ngày đã check, dùng để tính phần thưởng
}

export const CheckInDataSchema = SchemaFactory.createForClass(CheckInData);
```

> Các bạn có thể viết unit test cho các model này để đảm bảo nếu có đủ thời gian.

Sau khi đã tạo xong `DailyCheckIn` model chúng ta sẽ cập nhật lại `User` model:

```typescript:src/modules/users/entities/user.entity.ts
import { CheckInData, CheckInDataSchema } from '@modules/daily-check-in/entities/check-in-data.entity';
...
export class User extends BaseEntity { 
    ...
    @Prop({
		type: [CheckInDataSchema],
	})
	@Type(() => CheckInData)
	daily_check_in?: CheckInData[];
```

Giờ chúng ta có thể tiến hành cập nhật lại logic, quá trình cập nhật sẽ xoay quanh method `updateDailyCheckin` của `UsersService`. Chúng ta sẽ bắt đầu các cycle TDD mới cho các use-case của method này. Trước tiên chúng ta cần tạo mock và thêm vào provider để sử dụng trong các test case:

```typescript:src/modules/users/test/users.service.spec.ts
import { DailyCheckInService } from '@modules/daily-check-in/daily-check-in.service';
import { DailyCheckIn } from '@modules/daily-check-in/entities/daily-check-in.entity';
import { CheckInData } from '@modules/daily-check-in/entities/check-in-data.entity';
    ...
    describe('UserService', () => {
	let daily_check_in_service: DailyCheckInService;
    ...
        beforeEach(async () => {
            const module_ref = await Test.createTestingModule({
                providers: [
                    ...
                    {
                        provide: DailyCheckInService,
                        useValue: createMock<DailyCheckInService>(),
                    },
                ],
            })
                .compile();
            daily_check_in_service = module_ref.get(DailyCheckInService);
            ...
        });
    ...
```

## 1. Case 1.1
* Chúng ta sẽ bắt đầu cập nhật lại **`case 1.1: hôm nay là ngày cuối tháng => nhận thưởng tháng này => check-in`** như sau:
```typescript:src/modules/users/test/users.service.spec.ts
    ...
    describe('Daily Check-in', () => {
		describe('Case 1: User never check-in before', () => {
			it('should receive reward if it is the last day of month (case 1.1)', async () => {
				// Arrange
				const user = createUserStub();
				const check_in_date = new Date('2023-01-31 11:11:11');
				const check_in_data = [
					{
						eligible_for_reward: true,
						checked_date: check_in_date.toDateString(),
					},
				];

				// Act
				await users_service.updateDailyCheckIn(user, check_in_date);

				// Assert
				expect(daily_check_in_service.create).toBeCalledWith({
					user,
					month_year: `${
						check_in_date.getMonth() + 1
					}-${check_in_date.getFullYear()}`,
					check_in_data,
				});
				expect(users_repository.update).toBeCalledWith(user._id, {
					point: user.point + 1,
					last_check_in: check_in_date,
					last_get_check_in_rewards: check_in_date,
					daily_check_in: check_in_data,
				});
			});
```
Chúng ta sẽ có một số cập nhật trong logic test case:
* Gộp `testing_date` vào chung `check_in_date` cho ngắn gọn.
* `checked_date` trong `check_in_data` mình sẽ chỉ lưu ngày-tháng-năm (date) và bỏ qua giờ-phút-giây (time) vì ứng dụng của chúng ta không yêu cầu lưu lại chính xác thời gian. Việc này giúp viết logic so sánh ở case 2.1 dễ dàng hơn.
* Chúng ta cho check việc implement logic tách check-in data từ collection User sang lưu ở cả 2 collection User và DailyCheckIn.

Tiến hành save lại và kết thúc **`Phase 1: write a falling test`** của **Cycle** với kết quả lỗi như bên dưới.

![image.png](https://images.viblo.asia/ea631fe1-fb72-45c6-b729-4752ee10a208.png)

**`Phase 2: make the test pass`** như thường lệ căn cứ vào thông tin từ test case chúng ta sẽ cập nhật lại `UsersService` để làm cho test ở trên pass.

```typescript:src/modules/users/users.service.ts
...
export class UsersService extends BaseServiceAbstract<User> {
    constructor(
		...
		private readonly daily_check_in_service: DailyCheckInService,
    )
    ...
    async updateDailyCheckIn( user: User, date_for_testing?: string ): Promise<User> {
           ...
           // Case 1
			if (!daily_check_in?.length) {
				// Case 1.1: PASS
				if (isLastDayOfMonth(check_in_time)) {
					const check_in_data = [
						{
							eligible_for_reward: true,
							checked_date: check_in_time.toDateString() as unknown as Date,
						},
					];
					const [updated_user] = await Promise.all([
						this.users_repository.update(user._id.toString(), {
							point: user.point + 1,
							daily_check_in: check_in_data,
							last_check_in: check_in_time,
							last_get_check_in_rewards: check_in_time,
						}),
						this.daily_check_in_service.create({
							user,
							month_year: `${
								check_in_time.getMonth() + 1
							}-${check_in_time.getFullYear()}`,
							check_in_data,
						}),
					]);
					return updated_user;
				}
	}
    ...
```

Ở case này chúng ta chỉ cần thêm logic lưu check-in data vào `DailyCheckIn` model nên không có vấn đề gì, save lại và xem test đã pass hay chưa.

![image.png](https://images.viblo.asia/fcbca5ef-4509-4602-bbf7-ef7e0ec0fcde.png)

Cuối cùng là **`Phase 3: Refactor`**, tuy nhiên code ở trên có thể thấy không còn gì để refactor, do đó chúng ta có thể skip và kết thúc **Cycle** cho test case này ở đây. 
> Sau khi xong cycle các bạn có thể gọi API để kiểm tra lần nữa xem đã hoạt động đúng như ý muốn chưa. Vì đôi khi test case chúng ta viết bị thiếu hoặc sai dẫn đến lỗi.

**Case 1.2** sẽ tương tự với case trên nên mình sẽ nhường các bạn, chúng ta đi đến case tiếp theo
## 2. Case 2.1
**`Case 2.1: hôm nay đã điểm danh => tăng access amount`**

```typescript:src/modules/users/test/users.service.spec.ts
...
describe('Case 2: User has checked in before', () => {
	it('should increase amount access time and last check-in if the day to check-in has already checked in  (case 2.1)', async () => {
        // Arrange
        const user = {
            ...createUserStub(),
            daily_check_in: [
                {
                    checked_date: new Date('2023-01-31'),
                    eligible_for_reward: true,
                    access_amount: 1,
                },
            ],
            last_check_in: new Date('2023-01-31 07:00:00'),
        } as unknown as User;
        const daily_check_in = {
            user,
            month_year: `1-2023`,
            check_in_data: [
                {
                    eligible_for_reward: true,
                    access_amount: 2,
                    reward_days_count: 1,
                    checked_date: new Date('2023-01-31'),
                },
            ],
        } as DailyCheckIn;
        const check_in_date = new Date('2023-01-31 15:00:00');
        jest
            .spyOn(daily_check_in_service, 'increaseAccessAmount')
            .mockResolvedValueOnce(daily_check_in);

        // Act
        await users_service.updateDailyCheckIn(user, check_in_date);

        // Assert
        expect(daily_check_in_service.increaseAccessAmount).toBeCalledWith(
            user._id,
            check_in_date,
        );
        expect(users_repository.update).toBeCalledWith(user._id, {
            daily_check_in: daily_check_in.check_in_data,
            last_check_in: check_in_date,
        });
    });
```
Giải thích:
* Logic test case này sẽ được cập nhật nhiều hơn, để tăng số lượng truy cập của user trong ngày hôm đó chúng ta sẽ gọi đến `DailyCheckInService`. Mình sẽ tạo method tên là `increaseAccessAmount` để xử lý trường hợp này.
* Dữ liệu check-in ở user sẽ được đồng bộ từ kết quả trả về của method `increaseAccessAmount` để dữ liệu được đồng nhất.

Sau khi save lại thì test case vừa tạo sẽ failed và chúng ta có thể kết thúc **Phase 1**. Tuy nhiên chúng ta **tạm thời không thể sang** **Phase 2** vì sự xuất hiện method `increaseAccessAmount` và do mình đang viết theo style **Inside-out** trong TDD nên trước khi sang **Phase 2** cần viết test cho `increaseAccessAmount` trước rồi mới có thể tiếp tục.

Tiến hành tạo **Cycle** mới cho method `increaseAccessAmount` ở `DailyCheckInService`:

```typescript:src/modules/daily-check-in/test/daily-check-in.service.spec.ts
import { Test } from '@nestjs/testing';
import { createMock } from '@golevelup/ts-jest';

// INNER
import { DailyCheckInRepository } from '@repositories/daily-check-in.repository';
import { DailyCheckInService } from '../daily-check-in.service';
import { DailyCheckInRepositoryInterface } from '../interfaces/daily-check-in.interface';

// OUTER
import { createUserStub } from '@modules/users/test/stubs/user.stub';

describe('DailyCheckInService', () => {
	let daily_check_in_service: DailyCheckInService;
	let daily_check_in_repository: DailyCheckInRepository;
	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
				DailyCheckInService,
				{
					provide: 'DailyCheckInRepositoryInterface',
					useValue: createMock<DailyCheckInRepositoryInterface>(),
				},
			],
		})
			.compile();
		daily_check_in_repository = module_ref.get(
			'DailyCheckInRepositoryInterface',
		);
		daily_check_in_service = module_ref.get(DailyCheckInService);
	});

	afterEach(() => {
		jest.clearAllMocks();
	});

	it('should be defined', () => expect(daily_check_in_service).toBeDefined());

	describe('increaseAccessAmount', () => {
		it('should update check in data by call repository to apply', async () => {
			// Arrange
			const check_in_date = new Date('2023-01-05');
			const user = createUserStub();

			// Act
			await daily_check_in_service.increaseAccessAmount(
				user._id.toString(),
				check_in_date,
			);

			// Assert
			expect(daily_check_in_repository.increaseAccessAmount).toBeCalledWith(
				user._id,
				check_in_date,
			);
		});
	});
});
```

Vì ban đầu chúng ta thiết kế theo Repository Design Pattern nên việc triển khai logic database phải được xử lý ở tầng repository. Do đó chúng ta cần thêm **Cycle** nữa cho `DailyCheckInRepository`. 

> Đây là bất lợi mà mình đã nhắc ở phần đầu về việc Bucker Pattern làm tăng độ phức tạp của codebase.

```typescript:src/repositories/test/daily-check-in.repository.spec.ts
import { getModelToken } from '@nestjs/mongoose';
import { Test } from '@nestjs/testing';
import { Model } from 'mongoose';

// INNER
import {
	DailyCheckIn,
	DailyCheckInDocument,
} from '@modules/daily-check-in/entities/daily-check-in.entity';
import { DailyCheckInEntity } from './supports/daily-check-in.entity';
import { DailyCheckInRepository } from '@repositories/daily-check-in.repository';

// OUTER
import { createUserStub } from '@modules/users/test/stubs/user.stub';

describe('DailyCheckInRepository', () => {
	let daily_check_in_model: Model<DailyCheckInDocument>;
	let daily_check_in_repository: DailyCheckInRepository;

	beforeEach(async () => {
		const module_ref = await Test.createTestingModule({
			providers: [
				DailyCheckInRepository,
				{
					provide: getModelToken(DailyCheckIn.name),
					useClass: DailyCheckInEntity,
				},
			],
		}).compile();
		daily_check_in_repository = module_ref.get(DailyCheckInRepository);
		daily_check_in_model = module_ref.get(getModelToken(DailyCheckIn.name));
	});

	afterEach(() => jest.clearAllMocks());

	describe('increaseAccessAmount', () => {
		it('should be call to model to increase access amount of given date', async () => {
			// Arrange
			const user = createUserStub();
			const check_in_date = new Date('2023-01-05');
			jest.spyOn(daily_check_in_model, 'findOneAndUpdate');
			// Act
			await daily_check_in_repository.increaseAccessAmount(
				user._id.toString(),
				check_in_date,
			);

			// Assert
			expect(daily_check_in_model.findOneAndUpdate).toBeCalledWith(
				{
					user: user._id,
					'check_in_data.checked_date': check_in_date.toDateString(),
				},
				{
					$inc: {
						'check_in_data.$.access_amount': 1,
					},
				},
				{
					new: true,
				},
			);
		});
	});
});
```
Giải thích: 
* Chúng ta tăng số lần truy cập của một ngày bằng `$inc` với filter được truyền vào là ngày đang check-in và id của user thực hiện check-in.
* Chúng ta dùng option `new: true` để trả về dữ liệu mới nhất, dùng cho việc lưu vào `User` model.

**Phase 1** của `increaseAccessAmount` trong `DailyCheckInRepository` đã xong, chúng ta cùng đến với **Phase 2**:
```typescript:src/repositories/daily-check-in.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';

// INNER
import {
	DailyCheckIn,
	DailyCheckInDocument,
} from '@modules/daily-check-in/entities/daily-check-in.entity';
import { DailyCheckInRepositoryInterface } from '@modules/daily-check-in/interfaces/daily-check-in.interface';

// OUTER
import { BaseRepositoryAbstract } from './base/base.abstract.repository';

@Injectable()
export class DailyCheckInRepository
	extends BaseRepositoryAbstract<DailyCheckInDocument>
	implements DailyCheckInRepositoryInterface
{
	constructor(
		@InjectModel(DailyCheckIn.name)
		private readonly daily_check_in_model: Model<DailyCheckInDocument>,
	) {
		super(daily_check_in_model);
	}

	async increaseAccessAmount(user_id: string, check_in_date: Date) {
		try {
			return await this.daily_check_in_model.findOneAndUpdate(
				{
					user: user_id,
					'check_in_data.checked_date': check_in_date.toDateString(),
				},
				{
					$inc: {
						'check_in_data.$.access_amount': 1,
					},
				},
				{
					new: true,
				},
			);
		} catch (error) {
			throw error;
		}
	}
}
```

> Trước đó các bạn đừng quên thêm type cho method `increaseAccessAmount` ở `DailyCheckInRepositoryInterface`.

Logic sẽ được apply tương tự chúng ta đã liệt kê ở test case, các bạn save lại và kiểm tra kết quả.

![image.png](https://images.viblo.asia/ce74529e-e261-42d6-8140-6e720369ab95.png)

Chúng ta đã xong **Cycle** cho `DailyCheckInRepository`, giờ chúng ta có thể quay lại **Phase 2** của **Cycle** ở `DailyCheckInService`
```typescript:src/modules/daily-check-in/daily-check-in.service.ts
import { Inject } from '@nestjs/common';

// INNER
import { DailyCheckIn } from './entities/daily-check-in.entity';
import { DailyCheckInRepositoryInterface } from './interfaces/daily-check-in.interface';

// OUTER
import { BaseServiceAbstract } from 'src/services/base/base.abstract.service';

export class DailyCheckInService extends BaseServiceAbstract<DailyCheckIn> {
	constructor(
		@Inject('DailyCheckInRepositoryInterface')
		private readonly daily_check_in_repository: DailyCheckInRepositoryInterface,
	) {
		super(daily_check_in_repository);
	}

	async increaseAccessAmount(
		user_id: string,
		check_in_date: Date,
	): Promise<DailyCheckIn> {
		try {
			return await this.daily_check_in_repository.increaseAccessAmount(
				user_id,
				check_in_date,
			);
		} catch (error) {
			throw error;
		}
	}
}
```
Chúng ta chỉ cần thêm vào method và gọi đến `increaseAccessAmount` của `DailyCheckInRepository` là xong. Save lại và kiểm tra kết quả:

![image.png](https://images.viblo.asia/74af3995-67ae-4031-8c66-ad637f81a609.png)

Test case đã pass và chúng ta có thể kết thúc **Phase 2** cũng như **Cycle** `increaseAccessAmount` của `DailyCheckInService` để quay trở về với **Phase 2** của **Cycle** **`Case 1.2`**.
```typescript:src/modules/users/users.service.ts
...
export class UsersService extends BaseServiceAbstract<User> {
    constructor(
		...
		private readonly daily_check_in_service: DailyCheckInService,
    )
    ...
    async updateDailyCheckIn( user: User, date_for_testing?: string ): Promise<User> {
           ...
           // Case 1
			if (!daily_check_in?.length) { ... // Case 1.1, 1.2 }
            else {
                // Case 2.1
				if ( user.last_check_in.toDateString() === check_in_time.toDateString() ) {
					const current_daily_check_in =
						await this.daily_check_in_service.increaseAccessAmount(
							user._id.toString(),
							check_in_time,
						);
					return await this.users_repository.update(user._id.toString(), {
						daily_check_in: current_daily_check_in.check_in_data,
						last_check_in: check_in_time,
					});
				}
            }
            ...
	}
    ...
```
Giải thích: kết quả trả về từ `increaseAccessAmount` sẽ được cập nhật thẳng vào `daily_check_in` của `User` model vì dữ liệu trả về chắc chắn là tháng được điểm danh. 

![image.png](https://images.viblo.asia/7acfb94e-53c4-45d6-9844-de6e5ce1100c.png)

Thật vậy, test case đã pass như chúng ta mong đợi, các bạn có thể thử gọi API check-in một vài lần để kiểm tra kết quả. **Phase 2** kết thúc cũng đã khép lại cycle của **`Case 2.1`**. 

> Có thể thấy quá trình triển khai khi có cập nhật logic mà phát sinh liên quan giữa nhiều thành phần khá vất vả. Tuy nhiên nhờ nó chúng ta mới thấy được khi chỉnh sửa logic liên quan đến nhiều thành phần, có thể sinh ra các bug tiềm ẩn trong lúc chúng ta sơ suất hoặc thiếu tập trung. 
> 
> Việc viết test tuy khó khăn, vất vả và phiền phức như việc chỉ có mấy dòng đơn giản mà cũng phải viết như `increaseAccessAmount` của `DailyCheckInService` ở trên, nhưng đổi lại thành quả mà nó mang lại luôn xứng đáng hơn những gì chúng ta bỏ ra. 

## 3. Case 2.2.1.1
Chúng ta sẽ tiếp tục chỉnh sửa với **`Case 2.2.1.1: tháng trước đó chưa nhận thưởng => nhận thưởng tháng trước đó => nhận thưởng tháng này => check-in`**.

```typescript:src/modules/users/test/users.service.spec.ts
    ...
    describe('Case 2.2: The day to check-in has not checked in yet', () => {
        describe('Case 2.2.1: The day to check-in is the last day of month', () => {
            it('should receive reward for both months if the previous month has not got reward yet (case 2.2.1.1)', async () => {
                // Arrange
                const previous_month_check_in_data: CheckInData[] = [
                    {
                        checked_date: new Date('2023-01-10'),
                        eligible_for_reward: false,
                        access_amount: 1,
                        reward_days_count: 1,
                    },
                ];

                const user = {
                    ...createUserStub(),
                    daily_check_in: previous_month_check_in_data,
                    last_check_in: new Date('2023-01-10 07:00:00'),
                    last_get_check_in_rewards: new Date('2022-12-31 09:00:00'),
                } as unknown as User;
                const check_in_date = new Date('2023-02-28 15:00:00');
                const check_in_data = [
                    {
                        checked_date: check_in_date,
                        eligible_for_reward: true,
                        access_amount: 1,
                        reward_days_count: 1,
                    },
                ];
                jest
                    .spyOn(daily_check_in_service, 'addCheckInData')
                    .mockResolvedValueOnce({
                        user,
                        month_year: `2-2023`,
                        check_in_data,
                    });

                // Act
                await users_service.updateDailyCheckIn(user, check_in_date);

                // Assert
                expect(daily_check_in_service.addCheckInData).toBeCalledWith(
                    user._id,
                    check_in_date,
                );
                expect(users_repository.update).toBeCalledWith(
                    user._id.toString(),
                    {
                        last_check_in: check_in_date,
                        last_get_check_in_rewards: check_in_date,
                        point: user.point + user.daily_check_in.length + 1,
                        daily_check_in: check_in_data,
                    },
                );
            });
       });
```
Giải thích:
* Mình sẽ tách dữ liệu check-in ra một biến riêng là `previous_month_check_in_data` để tăng tính readability.
* Ở đây chúng ta tiếp tục cần một method mới có chức năng thêm dữ liệu ngày check-in vào trong bucket tháng. Mình sẽ đặt tên là `addCheckInData`.
* Logic cho việc tính điểm vẫn sẽ giữ nguyên là tổng số ngày của tháng trước đó cộng với 1 điểm của tháng hiện tại. Dữ liệu tháng trước đó sẽ được lấy từ `daily_check_in` trong User model, giúp chúng ta không cần mất thời gian tìm xem tháng trước đó là tháng mấy rồi mới lấy ra từ `DailyCheckIn` model.

Tương tự với ở trên chúng ta kết thúc **Phase 1** với test case fail và do cần method từ `DailyCheckInService` nên bị tạm dừng và bắt đầu **Cycle** khác bên trong.

```typescript:src/modules/daily-check-in/test/daily-check-in.service.spec.ts
...
describe('DailyCheckInService', () => {
   ...
    describe('addCheckInData', () => {
		it('should add check in data by call repository to apply', async () => {
			// Arrange
			const check_in_date = new Date('2023-02-28');
			const user = createUserStub();

			// Act
			await daily_check_in_service.addCheckInData(
				user._id.toString(),
				check_in_date,
			);

			// Assert
			expect(daily_check_in_repository.addCheckInData).toBeCalledWith(
				user._id,
				check_in_date,
			);
		});
	});
});
```

Method này logic xử lý cũng sẽ giống với `increaseAccessAmount` nên chúng ta không cần phải bận tâm, tiếp tục cycle cho `DailyCheckInRepository`.

```typescript:src/repositories/test/daily-check-in.repository.spec.ts
...
describe('DailyCheckInRepository', () => {
    ...
    describe('addCheckInData', () => {
		it('should be call to model to add check in data for given date', async () => {
			// Arrange
			const user = createUserStub();
			const check_in_date = new Date('2023-02-28');
			jest.spyOn(daily_check_in_model, 'findOneAndUpdate');

			// Act
			await daily_check_in_repository.addCheckInData(
				user._id.toString(),
				check_in_date,
			);

			// Assert
			expect(daily_check_in_model.findOneAndUpdate).toBeCalledWith(
				{
					user: user._id,
					month_year: `${
						check_in_date.getMonth() + 1
					}-${check_in_date.getFullYear()}`,
				},
				{
					$push: {
						check_in_data: {
							eligible_for_reward: true,
							checked_date: check_in_date.toDateString(),
							reward_days_count: 1,
						},
					},
				},
				{
					new: true,
					upsert: true,
				},
			);
		});
	});
});
```
Giải thích:
* Để thêm hoặc cập nhật check-in data vào tháng mình sẽ dùng `findOneAndUpdate` với option `upsert` vì khi sang tháng mới chúng ta sẽ không có sẵn dữ liệu.
* `$push` là một method phù hợp nhất cho trường hợp này, giúp chúng ta đẩy dữ liệu check-in mới vào các dữ liệu có sẵn hoặc khởi tạo nếu chưa tồn tại.

> Cycle repository của case này hoặc case trước đó một số bạn có thể nghĩ đến việc loại bỏ và thực hiện logic trực tiếp trên `DailyCheckInService` bằng cách dùng method `findOne` sau đó process dữ liệu ở *service layer* và dùng `update` cập nhật lại vào database. Cách dùng đó tuy ngắn gọn hơn nhưng sẽ không tận dụng được sức mạnh của MongoDB cũng như có thể gây ra **race condition** - một trong những thứ mà chúng ta nên tránh.

```typescript:src/repositories/daily-check-in.repository.ts
...
export class DailyCheckInRepository	extends BaseRepositoryAbstract<DailyCheckInDocument>
	implements DailyCheckInRepositoryInterface
{
    ...
    async addCheckInData(user_id: string, check_in_date: Date) {
		try {
			const daily_check_in = await this.daily_check_in_model.findOne({
				user: user_id,
				month_year: `${
					check_in_date.getMonth() + 1
				}-${check_in_date.getFullYear()}`,
			}).exec();
			return await this.daily_check_in_model.findOneAndUpdate(
				{
					user: user_id,
					month_year: `${
						check_in_date.getMonth() + 1
					}-${check_in_date.getFullYear()}`,
				},
				{
					$push: {
						check_in_data: {
							eligible_for_reward: true,
							checked_date: check_in_date.toDateString(),
							reward_days_count: daily_check_in
								? daily_check_in.check_in_data.length + 1
								: 1,
						},
					},
				},
				{
					new: true,
					upsert: true,
				},
			);
		} catch (error) {
			throw error;
		}
	}
```
Giải thích: 
* Để đảm bảo tính đúng đắn của `reward_days_count` trong ta cần kiểm tra xem trước đó đã check-in được bao nhiêu ngày trong tháng đó và dùng nó làm dữ liệu để thêm vào ngày được check-in.
* Do mock `DailyCheckInEntity` chúng ta có return về method `exec` nên gọi ở đây để không bị báo lỗi khi run test.

Sau khi đã viết xong method `addCheckInData` và pass test case, chúng ta có thể kết thúc **Phase 2** của `DailyCheckInRepository` và quay về với `DailyCheckInService`
```typescript:src/modules/daily-check-in/daily-check-in.service.ts
...
export class DailyCheckInService extends BaseServiceAbstract<DailyCheckIn> {
    ...
    async addCheckInData(
		user_id: string,
		check_in_date: Date,
	): Promise<DailyCheckIn> {
		try {
			return await this.daily_check_in_repository.addCheckInData(
				user_id,
				check_in_date,
			);
		} catch (error) {
			throw error;
		}
	}
}
```

Chúng ta cũng sẽ kết thúc **Phase 2** của nó vì không có gì để bàn và trở lại **Phase 2** của `UsersService`
```typescript:src/modules/users/users.service.ts
...
    async updateDailyCheckIn(user: User, date_for_testing = new Date()): Promise<User> {
        ...
        if (!daily_check_in?.length) { Case 1 }
        else {
            if (user.last_check_in.toDateString() === check_in_time.toDateString()) { Case 2.1 }
            if (isLastDayOfMonth(check_in_time)) {
                // Case 2.2.1.1
                if (
                    !user.last_get_check_in_rewards ||
                    (isDifferentMonthOrYear(
                        user.last_get_check_in_rewards,
                        user.last_check_in,
                    ) &&
                        isDifferentMonthOrYear(user.last_check_in, check_in_time))
                ) {
                    const previous_month_point = user.daily_check_in.length;
                    const current_daily_check_in =
                        await this.daily_check_in_service.addCheckInData(
                            user._id.toString(),
                            check_in_time,
                        );
                    return await this.users_repository.update(user._id.toString(), {
                        last_check_in: check_in_time,
                        last_get_check_in_rewards: check_in_time,
                        daily_check_in: current_daily_check_in.check_in_data,
                        point:
                            user.point +
                            previous_month_point +
                            current_daily_check_in.check_in_data.length,
                    });
                }
                ...
```
Giải thích:
* Việc lấy dữ liệu tháng trước đã đơn giản hơn trước kia rất nhiều, chúng ta chỉ cần lấy trong user thay vì phải loop qua tất cả dữ liệu cũ để tìm tháng trước đó.
* Gọi method vừa tạo để lưu dữ liệu vào DailyCheckIn model và lấy về data, kết hợp với dữ liệu tháng trước để cập nhật điểm và các thông tin liên quan cho user.

![image.png](https://images.viblo.asia/6b557a43-946f-4292-8071-bf0d8194cca2.png)

Với kết quả trên thì chúng ta đã có thể kết thúc **Phase 2**, nếu bạn nào thấy có cần chỉnh sửa gì thì có thể bắt đầu **Phase 3** và refactor. Ở đây mình thấy đã không còn gì nên sẽ kết thúc cycle của **`Case 2.2.1.1`** tại đây. Các case còn lại sẽ được xử lý dự trên method `addCheckInData` nên mình sẽ nhường các bạn tự hoàn thiện hoặc tải về từ source code. 

## 4. API lấy dữ liệu check-in
Chúng ta đã tách dữ liệu check-in ra nên khi cần truy vấn dữ liệu các tháng trước đó chúng ta cần có API mới. Dữ liệu check-in sẽ được lấy theo 2 cách:
* month: trả về check-in data của tháng đó.
* year: trả về tất cả check-in data của các tháng trong năm đó.
    
> Các bạn có thể thêm vào lấy theo quý cũng được, mình sẽ để dành cho bài viết về S.O.L.I.D

Mình sẽ triển khai **Cycle** để implement cho API này ở trường hợp lấy dữ liệu theo năm
```typescript:src/modules/users/test/users.service.spec.ts
import { PERIOD_TYPE } from '@modules/daily-check-in/dto/get-daily-check-in.dto';
...
...
describe('UserService', () => {
    ...
    describe('Get check-in data by', () => {
		describe('Year', () => {
			it('should return list check-in data of all given year', async () => {
				// Arrange
				const user = createUserStub();
				const check_in_data = [
					{
						_id: '64b17877dae2badc43e47366',
						user: user._id,
						month_year: '5-2023',
						check_in_data: [
							{
								checked_date: new Date('2023-05-20'),
								access_amount: 4,
								eligible_for_reward: false,
								reward_days_count: 1,
								_id: '64b17877dae2badc43e47367',
							},
						],
					},
					{
						_id: '64b17877dae2badc43e47369',
						user: user._id,
						month_year: '6-2023',
						check_in_data: [
							{
								checked_date: new Date('2023-06-07'),
								access_amount: 2,
								eligible_for_reward: false,
								reward_days_count: 1,
								_id: '64b17877dae2badc43e47370',
							},
							{
								checked_date: new Date('2023-06-14'),
								access_amount: 2,
								eligible_for_reward: false,
								reward_days_count: 2,
								_id: '64b17877dae2badc43e47371',
							},
						],
					},
				];
				(
					daily_check_in_service.findAllByPeriod as jest.Mock
				).mockResolvedValueOnce({
					count: check_in_data.length,
					items: check_in_data,
				});

				// Act
				const result = await users_service.getCheckInData(user._id.toString(), {
					type: PERIOD_TYPE.YEAR,
					year: '2023',
				});

				// Assert
				expect(daily_check_in_service.findAllByPeriod).toBeCalledWith({
					user_id: user._id,
					year: '2023',
					type: PERIOD_TYPE.YEAR,
				});
				expect(result).toEqual({
					count: check_in_data.length,
					items: check_in_data,
				});
			});
		});
	});
```

![image.png](https://images.viblo.asia/c813ac43-9c18-4f9a-b61f-6288fe48a4ef.png)

Vẫn sẽ là kết quả failed do chúng ta chưa define method, và ở đây chúng ta cần method `findAllByPeriod` ở `DailyCheckInService` nên **Cycle** mới sẽ được tạo tương tự như các case trước đó.

```typescript:src/modules/daily-check-in/test/daily-check-in.service.spec.ts
...
describe('DailyCheckInService', () => {
    ...
    describe('findAllByPeriod', () => {
		it('should get check in data by call to repository', async () => {
			// Arrange
			const filter = {
				year: '2023',
				type: PERIOD_TYPE.YEAR,
			};
			const user = createUserStub();

			// Act
			await daily_check_in_service.findAllByPeriod({
				user_id: user._id,
				...filter,
			});

			// Assert
			expect(daily_check_in_repository.findAllByPeriod).toBeCalledWith({
				user_id: user._id,
				...filter,
			});
		});
	});
```

Tiếp theo sẽ là **Cycle** của `DailyCheckInRepository` để gọi đến MongoDB model. **Phase 1:** thêm vào test case lấy dữ liệu theo năm:
```typescript:src/repositories/test/daily-check-in.repository.spec.ts
...
describe('DailyCheckInRepository', () => {
    ...
    describe('findAllByPeriod', () => {
		describe('Year', () => {
			it('should be call to model to return check in data base on filter', async () => {
				// Arrange
				const filter = {
					year: '2023',
					type: PERIOD_TYPE.YEAR,
					user_id: createUserStub()._id,
				};
				jest.spyOn(daily_check_in_model, 'find');
				// Act
				await daily_check_in_repository.findAllByPeriod(filter);

				// Assert
				expect(daily_check_in_model.find).toBeCalledWith({
					user: filter.user_id,
					month_year: {
						$regex: filter.year,
						$options: 'i',
					},
				});
			});
		});
	});
```

Tiếp tục triển khai **Phase 2** như bên dưới

```typescript:src/repositories/daily-check-in.repository.ts
...
    async findAllByPeriod(filter: {
		user_id: string;
		type: string;
		year: string;
	}): Promise<DailyCheckIn[]> {
		try {
			switch (filter.type) {
				case PERIOD_TYPE.YEAR:
					const check_in_data_year = await this.daily_check_in_model.find({
						month_year: {
							$regex: filter.year,
							$options: 'i',
						},
						user: filter.user_id,
					});
					return check_in_data_year;
				default:
					break;
			}
		} catch (error) {
			throw error;
		}
	}
```

Sau khi test đã pass chúng ta quay về với **Phase 2** ở `DailyCheckInService`
```typescript:src/modules/daily-check-in/daily-check-in.service.ts
    ...
    async findAllByPeriod(filter: findAllByPeriodDto): Promise<DailyCheckIn> {
		try {
			return await this.daily_check_in_repository.findAllByPeriod(filter);
		} catch (error) {
			throw error;
		}
	}
```

Phần này cũng không có gì đặc sắc nên chúng ta quay về luôn **Phase 2** của `UsersService`:
```typescript:src/modules/users/users.service.ts
    ...
    async getCheckInData(
		id: string,
		filter: findAllByPeriodDto,
	): Promise<DailyCheckIn[] | DailyCheckIn> {
		try {
			return await this.daily_check_in_service.findAllByPeriod({
				user_id: id,
				...filter,
			});
		} catch (error) {
			throw error;
		}
	}
```

Phase 2 kết thúc cũng khép lại **Cycle** của chúng ta về API lấy check-in data. Bài viết đã hơi dài nên trường hợp còn lại mình sẽ push lên git, khi các bạn tự triển khai nếu có vấn đề có thể tham khảo.

Các bạn lưu ý ở trên mỗi trường hợp mình chỉ lấy 1 test case làm ví dụ, trong thực tế thì mỗi trường hợp có thể có thêm các test case khác ở trong nên các bạn cần phải bao quát hết các khía cạnh của nó. Ví dụ như API lấy check-in data, nếu user gửi dữ liệu không hợp lệ cho month hoặc year thì chúng ta phải có các test case báo lỗi cho user. 

Một điều nữa là ở trên mình không viết về Cycle cho controller để tiết kiệm thời gian, nên khi các bạn triển khai nhớ đừng bỏ qua bước đó. Nội dung sẽ như bên dưới:

```typescript:
import { Test, TestingModule } from '@nestjs/testing';

// INNER
import { UsersController } from '../users.controller';
import { UsersService } from '../users.service';
import { createUserStub } from './stubs/user.stub';
import { RequestWithUser } from 'src/types/requests.type';

// OUTER
import { PERIOD_TYPE } from '@modules/daily-check-in/dto/get-daily-check-in.dto';
import { JwtAccessTokenGuard } from '@modules/auth/guards/jwt-access-token.guard';
import { isGuarded } from 'src/shared/test/utils';

jest.mock('../users.service.ts');
describe('UsersController', () => {
	let users_controller: UsersController;
	let users_service: UsersService;

	beforeEach(async () => {
		const module: TestingModule = await Test.createTestingModule({
			controllers: [UsersController],
			providers: [UsersService],
		}).compile();

		users_controller = module.get<UsersController>(UsersController);
		users_service = module.get<UsersService>(UsersService);
	});

	it('should be defined', () => {
		expect(users_controller).toBeDefined();
	});

	describe('updateDailyCheckIn', () => {
		it('should be protected with JwtAuthGuard', () => {
			expect(
				isGuarded(
					UsersController.prototype.updateDailyCheckIn,
					JwtAccessTokenGuard,
				),
			);
		});
		it('should add check-in and return user with new check-in data', async () => {
			// Arrange
			const request = {
				user: createUserStub(),
			} as RequestWithUser;

			// Act
			await users_controller.updateDailyCheckIn(request);

			// Assert
			expect(users_service.updateDailyCheckIn).toHaveBeenCalledWith(
				request.user,
				expect.any(Date),
			);
		});
	});

	describe('getCheckInData', () => {
		it('should be protected with JwtAuthGuard', () => {
			expect(
				isGuarded(
					UsersController.prototype.getCheckInData,
					JwtAccessTokenGuard,
				),
			);
		});
		it('should return check-in data base on filter', async () => {
			// Arrange
			const request = {
				user: createUserStub(),
			} as RequestWithUser;
			const filter = {
				type: PERIOD_TYPE.YEAR,
				year: '2023',
			};

			// Act
			await users_controller.getCheckInData(request, filter.type, filter.year);

			// Assert
			expect(users_service.getCheckInData).toHaveBeenCalledWith(
				request.user._id,
				filter,
			);
		});
	});
});
```

# Kết luận
Vậy là chúng ta đã hoàn thành xong chức năng check-in nhận reward vào cuối tháng, đáp ứng được các yêu cầu về tối ưu được nêu ra ở đầu bài viết. Thông qua các ví dụ trên hy vọng sẽ giúp các bạn có cái nhìn tổng quan cũng như lợi ích về cách thiết kế với Bucket Pattern và triển khai với TDD khi mới bắt đầu dự án hoặc khi dự án có yêu cầu thay đổi.

Hẹn gặp lại các bạn ở các bài viết tiếp theo, nếu thấy hay hãy cho mình 1 upvote và follow. Cảm ơn các bạn đã giành thời gian đọc bài viết.
# Tài liệu tham khảo
* Coupal, D. (2021) Building with patterns: The bucket pattern: Mongodb blog, MongoDB. Available at: https://www.mongodb.com/blog/post/building-with-patterns-the-bucket-pattern (Accessed: 16 July 2023). 
* The guide to test-driven development (TDD) (no date) PractiTest. Available at: https://www.practitest.com/resources/tdd-guide (Accessed: 17 July 2023). 
* Anonystick (no date) Bucket pattern Mongodb - Một MÔ hình paging (phân Trang) rất hiệu quả nhưng Khi Nào Nên sử dụng?, AnonyStick. Available at: https://anonystick.com/blog-developer/bucket-pattern-mongodb-mot-mo-hinh-paging-phan-trang-rat-hieu-qua-nhung-khi-nao-nen-su-dung-2022060471961127 (Accessed: 16 July 2023).