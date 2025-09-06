Đây là bài viết nằm trong Series NestJS thực chiến, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA
# Đặt vấn đề 📜  
Xin chào mọi người, Design Pattern trước giờ luôn là một chủ đề rất hot trong giới lập trình của chúng ta, nó không nhất thiết là điều kiện cần nhưng nó lại là điều kiện đủ để chúng ta đạt được mức lương mong muốn trong hành trình theo đuổi sự nghiệp 😇. 

Vì thế song song với cá bài về các công nghệ trong Series NestJS hiện tại, mình sẽ bổ sung thêm các bài viết về Design Pattern cho có thêm chút gia vị để mọi người đỡ ngán 🍡. Hôm nay chủ đề của chúng ta sẽ là **State Design Pattern**. Chúng ta sẽ nói qua về khai niệm sau đó bắt đầu với ví dụ về code OOP cơ bản và cuối cùng là apply vô dự án NestJS thực tế. 
# Lý thuyết 📃
> Disclaimer: bài viết này dựa theo kiến thức của mình nên nếu có sai sót mọi người góp ý giúp mình nhé. 
## Khái niệm 📖
> State is a behavioral design pattern that lets an object alter its behavior when its internal state changes. It appears as if the object changed its class.
> 
> Object thay đổi hành vi của nó khi state bên trong nó thay đổi. Như thể nó biến thành 1 class khác 🤔

Nói cho dễ hiểu thì ứng với mỗi state mà object mang, nó sẽ có các hành vi (method) khác nhau tương ứng (nghe cũng giống giống như đa nhân cách 🤕). Hiện giờ nhìn vào hình sẽ rồi lắm nên mọi người cứ lướt qua lát nữa hãy quay lại xem kỹ.

![image.png](https://images.viblo.asia/ea95351a-4548-4e73-b02c-d65ca250aa10.png)

Nguồn: https://refactoring.guru

Để ứng dụng được State Pattern chúng ta sẽ cần 3 thành phần chính:
- State: định nghĩa inteface chung cho tất cả các trạng thái. Chứa các phương thức mà ConcreteStates cần triển khai. Không chứa logic cụ thể của từng trạng thái.
- ConcreteStates: Triển khai hành vi cụ thể của từng trạng thái. Thay đổi trạng thái trong Context nếu cần.
- Context: Lưu trữ trạng thái hiện tại và cung cấp method để thay đổi trạng thái. Ủy quyền hành vi cho trạng thái hiện tại. Đảm bảo trạng thái thay đổi theo đúng quy tắc.

Để tìm hiểu kỹ hơn chúng ta sẽ đi vào phần sau với ví dụ cụ thể nhé. 
## Vấn đề ⚠️
Lấy ví dụ về dự án Flash card của Seri, giả sử trong tương lai chúng ta phân chia user theo nhóm để tăng cường tính gắn kết của người dùng cũng như tạo ra một hoạt động chung.  Trong đó khi một user trong nhóm upload Flash card lên cho các thành viên trong nhóm xem thì phải trải qua 1 lần duyệt bởi trưởng nhóm. 

Với yêu cầu trên đoạn code của chúng ta có thể điều chỉnh lại bổ sung thêm các method như bên dưới:
```typescript
enum FlashCardState {
    Draft    = "Draft",
    InReview = "InReview",
    Approved = "Approved",
    Rejected = "Rejected",
    Publish  = "Publish",
}
class FlashCard {
    state = FlashCardState.Draft;
    constructor(vocabulary: string, meaning: string, ...// Other property) {}
    
    review(user: User, status: "approve" | "reject" ) { 
        switch (flash_card.state) {
            case FlashCardState.Draft: 
                throw new Error("Invalid state")
            case FlashCardState.InReview: 
                if (user.Role !== "Team_Leader") throw new Error("No permission")
                if (status === "approve") {
                    this.state = FlashCardState.Approved;
                    return;
                }
                this.state = FlashCardState.Rejected;
                return
            case FlashCardState.Approved: 
            case FlashCardState.Rejected:
            case FlashCardState.Publish:
                throw new Error("Invalid state")
        }
    }
}
```
Chúng ta thấy đoạn code trên hoạt động ổn định và trông cũng gọn gàng nên rất hài lòng 🤩. Tuy nhiên sau 1 khoảng thời gian sử dụng, chúng ta phát hiện có 1 nhóm user (bao gồm cả trưởng nhóm) upload lên những Flash card vi phạm chính sách 🤥😡. Chúng ta muốn ngăn chặn nhưng hiện tại do chỉ cho phép trưởng nhóm duyệt nên không thể kiểm soát được tình trạng này. Vì thế chúng ta cần cập nhật logic, bắt buộc thêm 1 lần duyệt nữa bởi Admin, hàng loạt thay đổi áp dụng cho method `review`:
```typescript
enum FlashCardState {
    Draft       = "Draft",
    InReview    = "InReview",
    ApprovedOne = "ApprovedOne",
    RejectedOne = "RejectedOne",
    ApprovedTwo = "ApprovedTwo",
    RejectedTwo = "RejectedTwo",
    Publish     = "Publish",
}
class FlashCard {
    state = FlashCardState.Draft;
    constructor(vocabulary: string, meaning: string, ...// Other property) {}
    
    review(user: User, status: "approve" | "reject" ) {
        switch (flash_card.state) {
            case FlashCardState.Draft: 
                throw new Error("Invalid state")
            case FlashCardState.InReview: 
                if (user.Role !== "Team_Leader") throw new Error("No permission")
                if (status === "approve") {
                    this.state = FlashCardState.ApprovedOne;
                    return;
                }
                this.state = FlashCardState.RejectedOne;
                    return;
            case FlashCardState.ApprovedOne:  // <== ✅️ Bổ sung thêm 1 lần duyệt 2
                if (user.Role !== "Admin") throw new Error("No permission")
                if (status === "approve") {
                    this.state = FlashCardState.ApprovedTwo;
                    return;
                }
                this.state = FlashCardState.RejectedTwo;
                    return;
            case FlashCardState.RejectedOne:
            case FlashCardState.ApprovedTwo: 
            case FlashCardState.RejectedTwo:
            case FlashCardState.Publish:
                throw new Error("Invalid state")
        }
    }
}
```
Mặc dù đoạn code trên đã giải quyết được vấn đề chúng ta đặt ra, tuy nhiên nó mang lại rất nhiều weakness:
- Khi số lượng state tiếp tục tăng lên, chúng ta phải tăng thêm các case dẫn đến nhìn method dài dòng khó đọc.
- Khi giảm số lượng state hoặc điều chỉnh logic (ví dụ như cho phép Admin duyệt lần 1) thì chúng ta cũng phải vào chỉnh sửa.
- Các thay đổi của chúng ta áp dụng lên các code có sẵn, dẫn đến vi phạm nguyên lý **O**: Open/Closed trong S.O.L.I.D.

## Giải pháp 💡
Để khắc phục vấn đề trên, một trong số những cách tiếp cận đó là State Design Pattern. Đầu tiên chúng ta quy định State và các method của nó, ở đây là `review`:
```typescript
interface State {value: string
    value: string
    review(user: User, status: "approve" | "reject")
}
```
Sau khi có `State interface` thì chúng ta sẽ lần lượt tạo ra các `Concrete State` để implement các hành vi riêng cho từng State.
```typescript
class DraftState implements State {
    value = FlashCardState.Draft
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ❌️ Trạng thái Draft thì không thể review
        throw new Error("Invalid state")
    }
}
```
```typescript
class InReviewState implements State {
    value = FlashCardState.InReview
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ✅️ Trạng thái InReview chúng ta đem logic khi nảy vào đây
        if (user.role !== "Team_Leader") throw new Error("No permission")
        if (status === "approve") {
            this.flashCard.changeState(new ApprovedOneState(this.flashCard))
            return;
        }
        this.flashCard.changeState(new RejectedOneState(this.flashCard))
        return;
    }
}
```
```typescript
class ApprovedOneState implements State {
    value = FlashCardState.ApprovedOne
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ✅️ Trạng thái ApprovedOne cũng tương tự
        if (user.role !== "Admin") throw new Error("No permission")
        if (status === "approve") {
            this.flashCard.changeState(new ApprovedTwoState(this.flashCard))
            return;
        }
        this.flashCard.changeState(new RejectedTwoState(this.flashCard))
        return;
    }
}
```
```typescript
class RejectedOneState implements State {
    value = FlashCardState.RejectedOne
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ❌️ Trạng thái RejectedOne thì không thể review
         throw new Error("Invalid state")
    }
}
```
```typescript
class ApprovedTwoState implements State {
    value = FlashCardState.ApprovedTwo
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ❌️ Trạng thái ApprovedTwo thì không thể review
         throw new Error("Invalid state")
    }
}
```
```typescript
class RejectedTwoState implements State {
    value = FlashCardState.RejectedTwo
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ❌️ Trạng thái RejectedTwo thì không thể review
         throw new Error("Invalid state")
    }
}
```
```typescript
class PublicState implements State {
    value = FlashCardState.Publish
    constructor(private flashCard: FlashCard) {}
    review(user: User, status: "approve" | "reject") { // <== ❌️ Trạng thái Publish thì không thể review
         throw new Error("Invalid state")
    }
}
```

Có thể thấy với các `Concrete State` trên, chúng ta phân chia rõ ràng hành vi (logic) của từng State ra, từ đó việc bổ sung thêm State diễn ra rất dễ dàng và việc chỉnh sửa logic cũng vậy. Tiếp theo chúng ta sẽ đến với thành phần cuối cùng là `Context`
```typescript
class FlashCard { // 💡 Các bạn đặt là FlashCardContext cũng được
    state: State
    constructor(name: string, meaning: string) {
        this.state = new DraftState(this)
    }
    changeState(state: State) {
        this.state = state;
    }
    review(user: User, status: "approve" | "reject") {
         this.state.review(user, status)
    }
}
```
Nhiệm vụ của `Context`:
- Giúp lưu trữ các references đến một trong những `Concrete State` object: `this.state = new DraftState(this)`
- Uỷ quyền cho nó thực hiện các công việc cụ thể của State: `this.state.review(user, status)`. 
- Nó cũng cung cấp ra một hàm để cho phép chuyển đổi State: `changeState`

Mọi thứ đã chuẩn bị xong, giờ chúng ta chỉ cần gọi test thử để kiểm tra kết quả:
```typescript
const flash_card = new FlashCard("Hello", "Xin chào");
console.log(flashCard.state.value)
flashCard.changeState(new InReviewState(flashCard))
console.log(flashCard.state.value)
flashCard.review(new User("name", "Team_Leader"), "approve");
console.log(flashCard.state.value)
flashCard.review(new User("name", "Admin"), "approve");
console.log(flashCard.state.value)
```
Kết quả:
![](https://images.viblo.asia/a5f1a9c3-b5c5-4c2e-b442-f18b390d4925.png)


> Mọi người có thể vào link này để chạy thử code: https://replit.com/@NgocNguyen16/StateDesignPattern#flash-card.ts


Chúng ta có được gì với những thay đổi trên? Có thể các bạn sẽ thấy nó còn dài dòng hơn so với phiên bản đầu 😅, tuy nhiên nó mang lại những lợi ích thiết thực 😎:
- Code được mở rộng theo chiều ngang, với số lượng State tăng lên chúng ta chỉ cần tạo thêm class và extends state để implement method.
- Với những thay đổi logic, tuy không thể đảm bảo 100% tuân theo nguyên tắc **O** nhưng chúng ta thu gọn được phạm vi thay đổi hơn so với ban đầu.

💡 Nếu mọi người thấy ở trên còn hơi dài dòng vì method review ở một số State bị lặp lại, đừng lo chúng ta có thể khác phục bằng cách dùng `abstract class`, lúc trước có bạn cũng đã hỏi mình vì sao dùng `abstract class` thay cho `interface`:
```typescript
abstract class State {
    review(user: User, status: "approve" | "reject") {
         throw new Error("Invalid state")
    }
}
```
```typescript
class DraftState extends State { }

class InReviewState extends State {
    review(user: User, status: "approve" | "reject") {
        if (user.Role !== "Team_Leader") throw new Error("No permission")
        if (status === "approve") {
            this.flashCard.changeState(new ApprovedOneState(this.flashCard))
            return;
        }
        this.flashCard.changeState(new RejectedOneState(this.flashCard))
        return;
    }
}
class ApprovedOneState extends State {
    review(user: User, flash_card: FlashCard, status: "approve" | "reject") {
        if (user.Role !== "Admin") throw new Error("No permission")
        if (status === "approve") {
            this.flashCard.changeState(new ApprovedTwoState(this.flashCard))
            return;
        }
        this.flashCard.changeState(new RejectedTwoState(this.flashCard))
        return;
    }
}
class RejectedOneState extends State { }
class ApprovedTwoState extends State { }
class RejectedTwoState extends State { }
class PublicState extends State { }
```

Có thể thấy code đã ngắn gọn hơn nhiều 😍. Chúng ta sẽ thêm một ví dụ nữa cho trường hợp cập nhật thêm method, ở trên chúng ta chuyển trực tiếp status sang InReview bằng đoạn code `flashCard.changeState(new InReviewState(flashCard))`. Trong thực tế chúng ta không cho phép client thao tác chuyển trực tiếp như vậy mà phải thông qua 1 method, mình sẽ tạo thêm method `sendReview`.

```typescript
abstract class State {
  value = FlashCardState.Draft;
  sendReview() { // <=== 💡 Chỉ cần bổ sung ở đây
    throw new Error("Invalid state");
  }
  ...
}
```

```typescript
class DraftState extends State {
  value = FlashCardState.Draft;
  constructor(private flashCard: FlashCard) {
    super();
  }
  sendReview() { // <=== 💡 Ở đây
    this.flashCard.changeState(new InReviewState(this.flashCard));
    return;
  }
}
```

```typescript
class FlashCard {
  ...
  sendReview() { // <=== 💡 Và ở đây
    this.state.sendReview();
  }
}
```
Sau khi cập nhật thêm method ở 3 class trên chúng ta đã có thể sử dụng:
```typescript
const flashCard = new FlashCard("Hello", "Xin chào");
console.log("Current state: ", flashCard.state.value, ".Try to review!");
try {
  flashCard.review(new User("name", "Team_Leader"), "approve");
} catch (e) { console.log(e) }
console.log("Current state: ", flashCard.state.value, ".Send review");
flashCard.sendReview();
console.log("Current state: ", flashCard.state.value, ".Aprroving one");
flashCard.review(new User("name", "Team_Leader"), "approve");
console.log("Current state: ", flashCard.state.value, ".Aprroving two");
flashCard.review(new User("name", "Admin"), "approve");
console.log("Current state: ", flashCard.state.value);
```
Chúng ta cùng chạy thử và xem kết quả nhé:

![](https://images.viblo.asia/b7fb6037-2c6b-44ca-970a-eff5a6bc8447.png)

Đối với trường hợp chúng ta thêm state mới thì sao, cùng mình thử luôn nhé. Mình sẽ thêm state Blocked dùng cho trường hợp Admin muốn block một thẻ nào đó, state này có thể chuyển từ bất kỳ state nào sang ngoại trừ Draft.

```typescript
enum FlashCardState {
  ...
  Blocked = "Blocked", // <=== Thêm state mới
}
abstract class State {
  ...
  block(user: User) {
    if (user.role !== "Admin") throw new Error("No permission");
    this.value = FlashCardState.Blocked;
  }
}
```

```typescript
class DraftState extends State {
  value = FlashCardState.Draft;
  constructor(private flashCard: FlashCard) {
    super();
  }
  block(user: User) {
    throw new Error("Invalid state");
  }
}
```

```typescript
class FlashCard {
  ...
  block(user: User) {
    this.state.block(user);
  }
}
```

Có thể thấy được việc chúng ta sử dụng State Pattern thì code sẽ rõ ràng hơn và tập trung chính xác vào logic cần thay đổi. Từ đó làm cho việc đọc code, review và maintain cũng trở nên dễ dàng và hiệu quả.

# Áp dụng vào thực tế
Ở trên chúng ta đã đi qua quá trình từ vấn đề cho đến giải pháp, tuy nhiên nếu các bạn nhìn vào dự án NestJS thì chúng ta thấy một thực tế là chúng ta còn phải lưu trữ các object vào database chứ không đơn thuần là ví dụ về OOP. Do đó để giúp mọi người tiếp cận sâu hơn mình sẽ lấy ví dụ về dự án Flash card mà chúng ta đang làm.

Source code của phần này sẽ nằm ở [branch](https://github.com/nntwelve/Boilerplate-NestJS/tree/design-pattern-behavioral-state) `design-pattern-behavioral-state`

## 1. Cách dùng đơn giản 💡
Chúng ta sẽ bằng đầu với cách mà hầu hết mọi người đều dùng, cứ cần gì thì viết trực tiếp:
```typescript:src/modules/flash-cards/entities/flash-card.entity.ts
...
export enum FlashCardState {
	Draft = 'Draft',
	InReview = 'InReview',
	ApprovedOne = 'ApprovedOne',
	RejectedOne = 'RejectedOne',
	ApprovedTwo = 'ApprovedTwo',
	RejectedTwo = 'RejectedTwo',
	Publish = 'Publish',
}
@Schema({
	collection: 'flash-cards',
})
export class FlashCard extends BaseEntity {
	...
	@Prop({ default: FlashCardState.Draft })
	state?: FlashCardState; // <== ✅️ Thêm field state
}
```

```typescript:src/modules/flash-cards/flash-cards.service.ts
...
@Injectable()
export class FlashCardsService extends BaseServiceAbstract<FlashCard> {
    ...
	sendReview(user: User, flash_card: FlashCard) {
		if (flash_card.state !== FlashCardState.Draft) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_DRAFT,
				details: 'Flash card wrong status',
			});
		}
        if (flash_card.user.toString() !== user._id.toString()) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_OWNED,
				details: 'Flash card not owned',
			});
		}
		return this.flash_cards_repository.update(flash_card._id.toString(), {
			state: FlashCardState.InReview,
		});
	}

	review(user: User, flash_card: FlashCard, status: 'approve' | 'reject') {
		let state = flash_card.state;
		switch (flash_card.state) {
			case FlashCardState.InReview:
				if (user.role !== 'Team_Leader')
					throw new BadRequestException({
						message: ERRORS_DICTIONARY.FLASH_CARD_NOT_REVIEWED,
						details: 'You do not have permission to review this flash card',
					});
				if (status === 'approve') {
					state = FlashCardState.ApprovedOne;
					break;
				}
				state = FlashCardState.RejectedOne;
				break;
			case FlashCardState.ApprovedOne:
				if (user.role !== 'Admin')
					throw new BadRequestException({
						message: ERRORS_DICTIONARY.FLASH_CARD_NOT_REVIEWED,
						details: 'You do not have permission to review this flash card',
					});
				if (status === 'approve') {
					state = FlashCardState.ApprovedTwo;
					break;
				}
				state = FlashCardState.RejectedTwo;
				break;
			case FlashCardState.Draft:
			case FlashCardState.RejectedOne:
			case FlashCardState.ApprovedTwo:
			case FlashCardState.RejectedTwo:
			case FlashCardState.Publish:
				throw new BadRequestException({
					message: ERRORS_DICTIONARY.FLASH_CARD_NOT_REVIEWED,
					details: 'Flash card wrong status',
				});
		}
		return this.flash_cards_repository.update(flash_card._id.toString(), {
			state,
		});
	}
}
```
Giải thích: ở trên chúng ta thêm 2 method đơn giản cho việc gửi duyệt và duyệt:
   - `sendReview`: gửi duyệt Flash card, check điều kiện trạng thái phải là Draft và người gửi duyệt phải là người tạo
   - `review`: người có quyền duyệt sẽ vào đồng ý hoặc từ chối, check điều kiện role Team_Leader chỉ được duyệt 1, Admin chỉ được duyệt 2 (các bạn có thể config admin được duyệt 1 luôn cũng được, tuỳ yêu cầu dự án). Các trạng thái còn lại khi duyệt sẽ báo lỗi

Sau cùng chúng ta thêm controller nữa là xong:

```typescript:src/modules/flash-cards/flash-cards.controller.ts
...
@Controller('flash-cards')
@ApiTags('flash-cards')
export class FlashCardsController {
	constructor(private readonly flash_cards_service: FlashCardsService) {}

	@Post(':id/sending-review')
	@ApiBearerAuth('token')
	@ApiOperation({
		summary: 'User send review for their flash card',
	})
	@UseGuards(JwtAccessTokenGuard)
	async sendReview(
		@Req() request: RequestWithUser,
		@Param('id', ParseMongoIdPipe) id: string,
	) {
		const flash_card = await this.flash_cards_service.findOne(id);
		if (!flash_card) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_FOUND,
				details: 'Flash card not found',
			});
		}
		if (flash_card.user._id.toString() !== request.user._id.toString()) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_OWNED,
				details: 'Flash card not owned',
			});
		}
		return this.flash_cards_service.sendReview(
			request.user,
			flash_card,
		);
	}

	@Post(':id/reviewing')
	@ApiBearerAuth('token')
	@ApiOperation({
		summary: 'Admin/leader review user/member flash card',
	})
	@ApiQuery({
		name: 'status',
		enum: ['approve', 'reject'],
	})
	@UseGuards(JwtAccessTokenGuard)
	async review(
		@Req() request: RequestWithUser,
		@Param('id', ParseMongoIdPipe) id: string,
		@Query('status') status: 'approve' | 'reject',
	) {
		const flash_card = await this.flash_cards_service.findOne(id);
		if (!flash_card) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_FOUND,
				details: 'Flash card not found',
			});
		}
		return this.flash_cards_service.review(
			request.user,
			flash_card,
			status,
		);
	}
}
```
Vậy là xong chúng ta chỉ cần gọi các API tương ứng với từng yêu cầu là xong được chức năng xét duyệt cơ bản. Giờ chúng ta sẽ cùng xem nếu dùng State Design Pattern thì mọi thứ sẽ ra sau nhé. 
## 2. Cách dùng với State design pattern 💪
Trước tiên chúng ta khởi tạo `State` là abtract class:
```typescript:src/modules/flash-cards/state-dp/flash-cards.state.ts
import { User } from '@modules/users/entities/user.entity';
import { BadRequestException } from '@nestjs/common';
import { ERRORS_DICTIONARY } from 'src/constraints/error-dictionary.constraint';
import { FlashCardsRepositoryInterface } from '../interfaces/flash-cards.interface';

export abstract class FlashCardStateAbstract {
	constructor(
		protected readonly flash_cards_repository: FlashCardsRepositoryInterface,
	) {}
	sendReview(user: User) {
		throw new BadRequestException({
			message: ERRORS_DICTIONARY.FLASH_CARD_NOT_DRAFT,
			details: 'Flash card wrong status',
		});
	}
	review(user: User, status: 'approve' | 'reject') {
		throw new BadRequestException({
			message: ERRORS_DICTIONARY.FLASH_CARD_NOT_REVIEWED,
			details: 'Flash card wrong status',
		});
	}
}
```
Giải thích: thay vì dùng interface thì mình tận dụng abstract class để chúng ta đỡ phải viết lại những method mặc định cho các state kế thừa

Tương ứng với 7 state mà Flash card chúng ta có, mình sẽ tạo ra 7 `Concrete State` kế thừa abstract class trên:
```typescript:src/modules/flash-cards/state-dp/flash-cards.concrete.ts
import { BadRequestException } from '@nestjs/common';
import { FlashCard, FlashCardState } from '../entities/flash-card.entity';
import { FlashCardStateAbstract } from './flash-cards.state';
import { ERRORS_DICTIONARY } from 'src/constraints/error-dictionary.constraint';
import { User } from '@modules/users/entities/user.entity';
import { FlashCardsRepositoryInterface } from '../interfaces/flash-cards.interface';

export class FlashCardDraftState extends FlashCardStateAbstract {
	constructor(
		protected readonly flash_card: FlashCard,
		protected readonly flash_cards_repository: FlashCardsRepositoryInterface,
	) {
		super(flash_cards_repository);
		if (flash_card.state !== FlashCardState.Draft) { // <=== 🟢 Check trạng thái flash card
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_DRAFT,
				details: 'Flash card wrong status',
			});
		}
	}
	sendReview(user: User) {
		if (this.flash_card.user.toString() !== user._id.toString()) {
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_NOT_OWNED,
				details: 'Flash card not owned',
			});
		}
		return this.flash_cards_repository.update(this.flash_card._id.toString(), {
			state: FlashCardState.InReview,
		});
	}
}

export class FlashCardInReviewState extends FlashCardStateAbstract {
	constructor(
		protected readonly flash_card: FlashCard,
		protected readonly flash_cards_repository: FlashCardsRepositoryInterface,
	) {
		super(flash_cards_repository);
		if (flash_card.state !== FlashCardState.InReview) { // <=== 🟢 Check trạng thái flash card
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_WRONG_STATUS,
				details: 'Flash card wrong status',
			});
		}
	}
	review(user: User, status: 'approve' | 'reject') {
		let state = this.flash_card.state;
		if (user.role !== 'Team_Leader')
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_PERMISSION_DENIED,
				details: 'You do not have permission to review this flash card',
			});
		if (status === 'approve') {
			state = FlashCardState.ApprovedOne;
		}
		state = FlashCardState.RejectedOne;
		return this.flash_cards_repository.update(this.flash_card._id.toString(), {
			state,
		});
	}
}

export class FlashCardApprovedOneState extends FlashCardStateAbstract {
	constructor(
		protected readonly flash_card: FlashCard,
		protected readonly flash_cards_repository: FlashCardsRepositoryInterface,
	) {
		super(flash_cards_repository);
		if (flash_card.state !== FlashCardState.InReview) { // <=== 🟢 Check trạng thái flash card
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_WRONG_STATUS,
				details: 'Flash card wrong status',
			});
		}
	}
	review(user: User, status: 'approve' | 'reject') {
		let state = this.flash_card.state;
		if (user.role !== 'Admin')
			throw new BadRequestException({
				message: ERRORS_DICTIONARY.FLASH_CARD_PERMISSION_DENIED,
				details: 'You do not have permission to review this flash card',
			});
		if (status === 'approve') {
			state = FlashCardState.ApprovedTwo;
		}
		state = FlashCardState.RejectedTwo;
		return this.flash_cards_repository.update(this.flash_card._id.toString(), {
			state,
		});
	}
}

export class FlashCardApprovedTwoState extends FlashCardStateAbstract {}
export class FlashCardRejectedOneState extends FlashCardStateAbstract {}
export class FlashCardRejectedTwoState extends FlashCardStateAbstract {}
export class FlashCardPublishState extends FlashCardStateAbstract {}
```
Giải thích:
- Ở `constructor` của các hàm mình sẽ cho kiểm tra trạng thái Flash card ngay khi class được tạo để tách biệt logic với các hàm bên dưới
- Các class của state Draft, InReview và ApprovedOne sẽ tương tự logic với khi chúng ta dùng cơ bản, chỉ cần copy vào là xong
- Các class của các state còn lại chỉ cần kế thừa là được (nếu kỹ hơn các bạn thêm constructor để check status cũng được), nếu cố tình gọi thì method `sendReview` hoặc `review` ở Abstract class sẽ được gọi.

Thành phần cuối cùng của bộ State là `Context`, mình sẽ khởi tạo như sau:
```typescript:src/modules/flash-cards/state-dp/flash-cards.context.ts
import { User } from '@modules/users/entities/user.entity';
import { FlashCard, FlashCardState } from '../entities/flash-card.entity';
import { FlashCardStateAbstract } from './flash-cards.state';
import {
	FlashCardApprovedOneState,
	FlashCardApprovedTwoState,
	FlashCardDraftState,
	FlashCardInReviewState,
	FlashCardPublishState,
	FlashCardRejectedOneState,
	FlashCardRejectedTwoState,
} from './flash-cards.concrete';
import { FlashCardsRepositoryInterface } from '../interfaces/flash-cards.interface';

export class FlashCardContext {
	state: FlashCardStateAbstract;

	constructor(
		flash_card: FlashCard,
		private readonly flash_cards_repository: FlashCardsRepositoryInterface,
	) {
		this.setState(flash_card);
	}

	private setState(flash_card: FlashCard): void {
		switch (flash_card.state) {
			case FlashCardState.Draft:
				this.state = new FlashCardDraftState(
					flash_card,
					this.flash_cards_repository,
				);
				break;
			case FlashCardState.InReview:
				this.state = new FlashCardInReviewState(
					flash_card,
					this.flash_cards_repository,
				);
				break;
			case FlashCardState.ApprovedOne:
				this.state = new FlashCardApprovedOneState(
					flash_card,
					this.flash_cards_repository,
				);
				break;
			case FlashCardState.RejectedOne:
				this.state = new FlashCardRejectedOneState(this.flash_cards_repository);
				break;
			case FlashCardState.ApprovedTwo:
				this.state = new FlashCardApprovedTwoState(this.flash_cards_repository);
				break;
			case FlashCardState.RejectedTwo:
				this.state = new FlashCardRejectedTwoState(this.flash_cards_repository);
				break;
			case FlashCardState.Publish:
				this.state = new FlashCardPublishState(this.flash_cards_repository);
				break;
		}
	}

	changeState(state: FlashCardStateAbstract) {
		this.state = state;
	}

	sendReview(user: User) {
		return this.state.sendReview(user);
	}

	review(user: User, status: 'approve' | 'reject') {
		return this.state.review(user, status);
	}
}
```
Giải thích:
- Mỗi khi chúng ta gọi API sẽ khởi tạo 1 Context và truyền trạng thái hiện tại của Flash card vào. Tương ứng với state nào thì Context sẽ tạo ra Concrete class đó.
- Chúng ta cũng sẽ có 2 method `sendReview` và `review` ở đây để đóng vai trò trung gian khi service gọi đến.

Sau khi đã cấu hình xong chúng ta chỉ cần cập nhật lại ở FlashCardService là xong:
```typescript:src/modules/flash-cards/flash-cards.service.ts
...
@Injectable()
export class FlashCardsService extends BaseServiceAbstract<FlashCard> {
    ...
	sendReview(user: User, flash_card: FlashCard) {
		const flash_card_context = new FlashCardContext(flash_card, this.flash_cards_repository);
		return flash_card_context.sendReview(user);
	}

	review(user: User, flash_card: FlashCard, status: 'approve' | 'reject') {
		const flash_card_context = new FlashCardContext(flash_card, this.flash_cards_repository);
		return flash_card_context.review(user, status);
	}
}
```

Có thể thấy được với việc áp dụng State pattern code chúng ta trở nên rõ ràng và mạch lạc hơn. Giả sử phát sinh yêu cầu chỉnh sửa bổ sung trạng thái duyệt 3 thì chuyện gì sẽ xảy ra.
- Cách thông thường: phải cập nhật lại logic chỉnh sửa trực tiếp ở method `review` => Vi phạm nguyên lý O
- Với State: viết thêm Concrete class cho logic mới và bổ sung ở Context, các file còn lại hầu như không cần phải chỉnh sửa gì => Đảm bảo được nguyên lý O

Tương tự khi xoá một bước duyệt cũng vậy, việc chúng ta cần làm là xoá Concrete class và ở Context là xong.

> Vẫn không quên nhắc mọi người tránh lạm dụng gây phức tạp codebase của chúng ta, ví dụ như chúng ta biết chắc chắn dự án từ đầu đến cuối chỉ có số bước duyệt cố định thì không cần dùng State Pattern để làm gì cho dài dòng và mất thời gian cho người sau vào maintain 😁


# Kết luận 📝
Vậy là chúng ta đã đi qua về cách hoạt động cũng như cách vận dụng State Design Pattern vào dự án NestJS, các bạn hãy thử vận dụng vào các dự án sau này để kiểm chứng xem nó có thật sự mang lại lợi ích như mình đã đề cập không nhé 😉. Nếu có bất kỳ thắc mặc hoặc vấn đề gì cần góp ý thì mọi người có thể comment ngay bên dưới nhé.

Cảm ơn mọi người đã giành thời gian đọc bài viết và hẹn gặp lại.
# Tài liệu tham khảo 🔍
- State Refactoring.Guru. Available at: https://refactoring.guru/design-patterns/state (Accessed: 05 March 2025). 
- YouTube Viet Tran. Available at: https://www.youtube.com/watch?v=Oy1MNhq2y_k&list=PLOsM_3jFFQRmNCt68hxCdxi8i_fUx2wTZ&index=21&ab_channel=Vi%E1%BB%87tTr%E1%BA%A7n (Accessed: 05 March 2025). 
# Change log 📓
- February 07, 2025: Init document.
- March 05, 2025: Update document.
- May 05, 2025: Publish document.