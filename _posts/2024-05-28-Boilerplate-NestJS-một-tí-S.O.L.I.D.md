Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---
# Đặt vấn đề 📜
**S.O.L.I.D** thì mọi người đã nghe nhắc đến quá nhiều trong quá trình lập trình, nó giúp code trở nên sạch sẽ, gọn gàng, dễ mở rộng và maintain hơn. Công dụng là vậy nhưng không ít lần khi có người hỏi và nhờ giải thích thì mặc dù hiểu nhưng chúng ta không thể diễn tả hết được bằng lời - kể cả mình cũng vậy 🫠. Thôi thì đã không thể diễn tả được bằng lời thì hôm nay chúng ta cùng diễn tả bằng bài viết này kèm với các ví dụ đơn giản và sâu hơn là ví dụ trong dự án chúng ta đang làm nhé 😁. 
# Thông tin package 📦️
- Source code phần S và I sẽ nằm ở branch [part-11-upload-file-aws-s3-client](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-11-upload-file-aws-s3-client)
- Source code ví dụ L nằm [ở đây](https://replit.com/@NgocNguyen16/Liskov-Substitution-Principal#isp.ts)


# Nội dung 💡
## S: Single Responsability Principle - SRP
![](https://images.viblo.asia/d784fa2b-23ef-4b06-9fa3-b4f6eb9c8e0c.jpg)
> **Each class must have one, and only one, reason to change** - 
> Một class/module chỉ nên mang một nhiệm vụ duy nhất. 
### Vấn đề ⚠️
Để hiểu rõ hơn chúng ta cùng đến với ví dụ sau:
```typescript
class Topic {
    createTopic(data: CreateTopicDto) {...}
    updateTopic(id: string, data: UpdateTopicDto) {...}
    deleteTopic(id: string) {...}
    getAllTopics(filter: GetAllFilter) {...}
    
    sendNotificationToSubscribers(id: string, content: string) {...} // Vi phạm SRP
    addNewSubscriber(id: string, user_id: string) {...} // Vi phạm SRP
}
```

Ở trên nhờ chú thích, chúng ta thấy được 2 method `sendNotificationToSubscribers` và  `addNewSubscriber` đang vi phạm nguyên tắc **S** vì nó đang làm các logic mặc dù vẫn nằm trong phạm vi Topic nhưng lại nằm ngoài trách nhiệm quản lý Topic của class. 

Chúng ta dễ nhầm lẫn vì nghĩ là 2 chức năng thêm subscriber và thông báo là của Topic, tuy nhiên vấn đề sẽ phát sinh sau này khi cần thay đổi logic gửi thông báo hoặc khi thêm subscriber, chúng ta đều phải vào chỉnh sửa class Topic, trong khi đó không phải là trách nhiệm của nó 😤.

Tuy nhiên, khi nói tới đây có thể một số bạn sẽ có suy nghĩ:
> "Tại sao phải phức tạp như vậy, cứ viết chung thôi, rồi khi cần chỉnh sửa chỉ cần mở file đó thôi có gì đâu? Chả thấy có vấn đề gì cả!"

Đó chính xác là những gì mình từng nghĩ tới khi lần đầu tiên đọc về nguyên tắc S, và sau khoảng thời gian suy ngẫm cũng như va chạm với kha khá dự án, đặc biệt là các dự án lớn thì mình nhận ra một số vấn đề có thể xảy ra nếu chúng ta bỏ qua nguyên tắc **S**:
- Nếu viết toàn bộ chức năng liên quan vào một file thì số lượng dòng code trong file sẽ rất lớn.
- Khi cần chỉnh sửa logic chúng ta phải mất thời gian tìm kiếm trong file cả ngàn dòng để chỉnh sửa.
- Khó khăn trong việc tách các chức năng ra module riêng, ví dụ chúng ta muốn tách notification ra riêng để phục vụ cho các entity khác.
- Khi một chức năng bị lỗi có thể ảnh hưởng đến các chức năng khác không liên quan trong class.
- Làm cho việc viết test cũng trở nên phiền phức hơn. Thử nghĩ đến việc chúng ta phải mock hàng tá thứ cho một file test là đã thấy ngán rồi.  
- ... (các bạn giúp mình bổ sung thêm ở phần bình luận nha)

### Giải pháp 💡
Chúng ta nên chỉnh sửa lại như sau:
```typescript
class Topic {
    createTopic(data: CreateTopicDto) {...}
    updateTopic(id: string, data: UpdateTopicDto) {...}
    deleteTopic(id: string) {...}
    getAllTopics(filter: GetAllFilter) {...}
}

class Subscriber {
    addNewSubscriber(id: string, user_id: string) {...}
}

class Notifier {
    sendNotificationToSubscribers(id: string, content: string) {...}
}
```
Sau khi refactor lại chúng ta có thể thấy mỗi class đã đảm nhiệm một logic riêng:
- Topic: chịu trách nhiệm quản lí topic
- Subscriber: chịu trách nhiệm quản lí các subscriber
- Notifier: chịu trách nhiệm gửi thông báo 

Từ đó giúp cho code chúng ta mang tính modular hơn 📦️, dễ đọc và dễ chỉnh sửa bảo trì.

Đến với ví dụ trong source code của chúng ta, các bạn có thể thấy ở file `image-uploading.processor.ts` thay vì viết toàn bộ logic upload file vào `ImageUploadingProcessor`, chúng ta đã tách riêng ra class `UploadFileServiceS3`:

```typescript:image-uploading.processor.ts
...
export class ImageUploadingProcessor extends WorkerHost {
	constructor(
		private readonly flash_cards_service: FlashCardsService,
		private readonly upload_file_service: UploadFileServiceAbstract,
	) { super(); }
	...
	async process(job: Job<any, any, string>, token?: string): Promise<any> {
		switch (job.name) {
			case 'uploading-image':
				...
				const uploaded_result = // UploadFileServiceS3 implement UploadFileServiceAbstract
					await this.upload_file_service.uploadFileToPublicBucket( 
						`flash-card/${job.data.full_name}-${job.data.user_id}`,
						{ file: optimized_image, file_name: job.data.file_name },
					);
				...
		}
	}
}
```
### Mục tiêu hướng tới 🌅
> **Giúp tách biệt các behaviors với nhau, để khi có lỗi phát sinh trong quá trình chỉnh sửa 1 behavior sẽ không ảnh hưởng các behavior không liên quan**
## O: Open/Closed Principle - OCP
![](https://images.viblo.asia/1fcfab3f-124b-4ebe-af34-024e08186cc3.jpg)

> **Software entities (classes, modules, functions, and so on) should be open for extension but closed for modification**

Theo mình thì đây là nguyên lý dễ hiểu nhất trong 5 principal, thiết kế code của chúng ta như thế nào để dễ dàng trong việc mở rộng hơn là chỉnh sửa code có sẵn. Có nghĩa là khi có feature mới hoặc cần chỉnh sửa behavior có sẵn chúng ta sẽ kế thừa hoặc tạo class mới thay vì chỉnh sửa vào code có sẵn.  
### Vấn đề ⚠️
Chúng ta thường thấy nguyên lý này khi áp dụng các logic có nhiều điều kiện `if-else` hoặc `switch-case` . Cùng xét ví dụ giả sử chúng ta có chức năng payment hỗ trợ các phương thức thanh toán như Card, ApplePay, ShopeePay và Cash:
```typescript
class PaymentService {
    public processPayment(method: PaymentMethod, orderData: any) {
    ...
        switch (method){
            case "Card":
                // Pay by Card
            case "ApplePay":
                // Pay by Apple Pay
            case "ShopeePay":
                // Pay by Shopee Pay
            case "Cash":
                // Pay by Cash
        }
    }
}
// Using
payment = new PaymentService()
payment.processPayment("ApplePay", ...)
```
Thoạt nhìn thì đoạn code trên không có vấn đề gì, và nếu như trong suốt vòng đời của ứng dụng chúng ta không thay đổi hoặc bổ sung thêm phương thức thanh toán thì nó **hoàn toàn ổn** 👌. Tuy nhiên với trường hợp chúng ta cần bổ sung thêm hoặc xóa bớt phương thức thanh toán và việc đó diễn ra thường xuyên thì nó mới phát sinh vấn đề 💣️. Chúng ta sẽ phải liên tục thêm/xóa các `case`, việc đó làm vi phạm nguyên tắc trên.

Tiếp tục là câu hỏi của mình ngày xưa:
> Sửa thì sửa thôi, thêm có mấy dòng code, nhìn cũng clean mà 🙄!

Chúng ta cùng xét ví dụ sau để biết clean hay không nha. Giả sử chúng ta đang code 1 app bán hàng, ban đầu có 3 actor mua hàng:
- User thông thường và Agent: cho phép thanh toán *Cash on Delivery* và Credit card, Apple Pay, ShopeePay.
- User guest: chỉ cho phép thanh toán với Credit card, Apple Pay, ShopeePay.

```typescript
enum PaymentMethod {
    CoD = "CoDPayment"
    ApplePay = "ApplePayPayment"
    Card = "CardPayment"
    ShopeePay = "ShopeePayPayment"
}
class OrderService {
    private paymentService = new PaymentService()
    createOrderForGuest(paymentMethod: PaymentMethod, orderData: any) {
        if (paymentMethod === PaymentMethod.CoD) throw new Error('This method is not allow');
        paymentService.processPayment(paymentMethod, orderData)
    }

    createOrderForUser(user: User, paymentMethod: PaymentMethod, orderData: any) {
        paymentService.processPayment(paymentMethod, orderData)
    }
}
```
> Giả sử ở trên chúng ta chia ra 2 method, 1 cái dùng cho JwtAuth 1 cái không cần Auth

Mọi thứ hoạt động ổn và đến 1 ngày nọ chúng ta muốn support Agent nên bổ sung thêm 1 payment method nữa là PayLater - cho phép Agent có thể mua hàng và thanh toán sau. Lưu ý: Agent ở đây vẫn là user chỉ khác type. 

Vì một vài lý do khách quan nào đó như sắp đến giờ về mà sếp yêu cầu ở lại làm cho xong, hoặc tối đó đang chuẩn bị đi chơi với người yêu thì sếp nhờ OT làm giùm ngay lập tức 🫠. Khi đó chúng ta sẽ ở trong trạng thái muốn làm nhanh cho xong nên chúng ta phi ngay vào chỉnh sửa **PaymentService**:
```typescript
class PaymentService {
    public processPayment(method: PaymentMethod, orderData: any) {
    ...
        switch (method){
            case PaymentMethod.Card:
                // Pay by Card
            case PaymentMethod.ApplePay:
                // Pay by Apple Pay
            case PaymentMethod.ShopeePay:
                // Pay by Shopee Pay
            case PaymentMethod.CoD:
                // Pay by CoD
            case PaymentMethod.PayLater: // 🆕: bổ sung vô đây (nhớ thêm vào enum)
                // Pay by later
        }
    }
}
```
và chỉnh sửa thêm ở đây:
```typescript
class OrderService {
    createOrderForGuest(paymentMethod: PaymentMethod, orderData: any) {
        if (paymentMethod === PaymentMethod.CoD) throw new Error('This method is not allow');
        paymentService.processPayment(paymentMethod, orderData)
    }

    createOrderForUser(user: User, paymentMethod: PaymentMethod, orderData: any) {
        // 🆕 Thêm logic trả về lỗi nếu user không phải Agent mà vẫn thanh toán PayLater
        if (user.type !== "Agent" && paymentMethod === PaymentMethod.PayLater) { 
            throw new Error('This method is not allow');
        }
        paymentService.processPayment(paymentMethod, orderData)
    }
}
```

Quá nhanh gọn chúng ta commit code và đi về/đi chơi với người yêu 😎. Một thời gian sau sếp phát hiện doanh thu thất thoát và kiểm tra lại thì đùng 1 cái xuất hiện rất nhiều những Guest user thanh toán PayLater và họ trốn luôn không trả tiền. Khi nhìn lại các bạn đã thấy vấn đề ở đâu rồi phải không: 
```typescript
class OrderService {
    createOrderForGuest(paymentMethod: PaymentMethod, orderData: any) { 
        if (paymentMethod === PaymentMethod.CoD) throw new Error('This method is not allow');
        // 🚫 Chúng ta quên đặt logic để check Guest không được thanh toán PayLater
        paymentService.processPayment(paymentMethod, orderData)
    }
    ...
}
```
Ví dụ trên chính là lý do tại sao chúng ta nên áp dụng nguyên lý **O**
### Giải pháp 💡
Chúng ta có thể refactor lại như sau:
```typescript
interface Payment {
    processPayment(method: PaymentMethod, orderData: any)
}

class CardPayment implements Payment {
    processPayment(method: PaymentMethod, orderData: any) { /* Pay by Card */ }
}
class ApplePayPayment implements Payment {
    processPayment(method: PaymentMethod, orderData: any) { /* Pay by ApplePay */ }
}
class ShopeePayPayment implements Payment {
    processPayment(method: PaymentMethod, orderData: any) { /* Pay by ShopeePay */ }
}
class CoDPayment implements Payment {
    processPayment(method: PaymentMethod, orderData: any) { /* Pay by CoD */ }
}
class PayLaterPayment implements Payment {
    processPayment(method: PaymentMethod, orderData: any) { /* Pay by later */ }
}

class PaymentService {
  // Danh sách các phương thức thanh toán được đăng ký
  private payments: Record<string, Payment> = {};
  // Dùng đăng ký thêm các phương thức thanh toán
  public registerPaymentMethod(payment: Payment) {
    this.payments[payment.name] = payment;
  }
  constructor(...paymentMethods: Payment[]) { 
      for (const paymentMethod of paymentMethods) {
          this.registerPaymentMethod(paymentMethod.name, paymentMethod)
      }
  }

  public async processPayment(paymentMethod: PAYMENT_METHOD, rest: any) {
    const payment = this.payments[paymentMethod];
    if (!payment) throw new Error('This method is not allow');
    await payment.processPayment(rest);
  }
}
```

```typescript
class OrderService {
    createOrderForGuest(paymentMethod: PaymentMethod, orderData: any) {
        paymentService = new PaymentService(new CardPayment(),
                                            new ApplePayPayment(),
                                            new ShopeePay())
        paymentService.processPayment(paymentMethod, orderData)
    }

    createOrderForUser(user: User, paymentMethod: PaymentMethod, orderData: any) {
        paymentService = new PaymentService(new CardPayment(),
                                            new ApplePayPayment(),
                                            new ShopeePayPayment(),
                                            new CodPayment())
        if (user.type === "Agent") { 
            paymentService.registerPaymentMethod(new PayLaterPayment())
        }
        paymentService.processPayment(paymentMethod, orderData)
    }
}
```

> Ở đây mình chỉ minh họa ngắn gọn để mọi người dễ hiểu, khi dùng trong thực tế thì không nên tạo PaymentService trong method, dẫn đến mọi lần gọi hàm lại tạo 1 instance thì không hay.

Có thể thấy sau này khi có thêm phương thức thanh toán mới, chúng ta chỉ cần tạo class implements interface `Payment` và khai báo lúc tạo `PaymentService` hoặc gọi `registerPaymentMethod` là xong. Việc này giúp chúng ta tránh được các lỗi tiềm tàng có thể xảy ra khi chỉnh sửa các class trong trường hợp đầu.
### Mục tiêu hướng tới 🌅
> **Tránh được các lỗi phát sinh không mong muốn khi chúng ta chỉnh sửa code có sẵn.**

## L: Liskov Substitution Principle - LSP
![](https://images.viblo.asia/a889f616-db98-4968-a643-86fabbb0dff8.jpg)

> Any instance of a subclass or derived class should be substitutable for an instance of its base class without affecting the correctness of the program.

Nguyên lý này theo mình là phức tạp nhất, nó biểu thị rằng các class con phải có khả năng thay thế được toàn bộ behavior của class mà nó kế thừa. 

### Vấn đề ⚠️
Chúng ta sẽ cùng xét ví dụ sau, giả sử chúng ta có class `Vehicle` được thiết kế như sau:

```typescript
export abstract class Vehicle {
  isEngineRunning = false;
  speed = 0;
  turnOnEngine(): void {
    this.isEngineRunning = true;
  }
  abstract accelerate(): void;
}
```
và có 2 class sẽ *extends* :
```typescript
export class Sedan extends Vehicle {
  accelerate(): void {
    this.speed += 80;
  }
}

export class Bicycle extends Vehicle {
  accelerate(): void {
    this.speed += 5;
  }
  turnOnEngine(): void {
    throw new Error("Bicycles don't have engines!");
  }
}
```

Có thể thấy class `Bicycle` không thể thực hiện được chức năng của method `turnOnEngine` ở class `Vehicle` và thế là chúng ta vi phạm nguyên tắc **L**. 

Vậy nó gây ra tác hại gì 🤔, các bạn có thể thấy đoạn code trên dường như vô hại đúng không ❎️. Tiếp tục với ví dụ, giả sử chúng ta đi du lịch bằng function sau: 

```typescript
function goTravelling(vehicle: Vehicle) {
  vehicle.turnOnEngine();
  vehicle.accelerate();
  console.log(`Goingggg with speed ${vehicle.speed}`);
}
```

Cách sử dụng là chúng ta sẽ khởi tạo class instance của class `Vehicle` và truyền vào function để giả sử cho quá trình trên đường đi du lịch.

```typescript
const sedan = new Sedan();
goTravelling(sedan) // ✅️ Chạy bình thường

const bicycle = new Bicycle();
goTravelling(bicycle) // ❌️ Báo lỗi: Bicycles don't have engines!
```

Rõ ràng là trong thực tế chúng ta có thể đi du lịch bằng xe đạp, nhưng ở ví dụ trên chúng ta lại gặp lỗi, đó là một unexpected behavior (một hành vi không mong đợi).

### Giải pháp 💡

Thông thường lỗi này xảy ra là do quá trình thiết kế các class không được hiệu quả, dẫn đến việc làm cho nó mang nhiều logic hơn mức cần thiết. Ví dụ như class `Vehicle` ở trên, chúng ta cần chỉnh lại như sau:
```typescript
abstract class Vehicle {
  speed = 0;
  abstract accelerate(): void;
  // Loại bỏ method `turnOnEngine` khỏi class Vehicle
}

abstract class Car extends Vehicle{
  isEngineRunning = false;
}

export class Sedan extends Car {
  accelerate(): void {
    this.speed += 40;
  }
  turnOnEngine(): void {
    this.isEngineRunning = true;
  }
}

export class Bicycle extends Vehicle {
  accelerate(): void {
    this.speed += 5;
  }
}
```

Có thể thấy chúng ta đã tách biệt logic khởi động engine ra khỏi `Vehicle` và từ đó việc `Bicycle` extends `Vehicle` mà không cần quan tâm đến việc có động cơ hay không. Chúng ta sẽ điều chỉnh lại function `goTravelling` để đáp ứng với thay đổi trên:
```typescript
function goTravelling(vehicle: Vehicle) {
  vehicle instanceof Car && vehicle.turnOnEngine();
  vehicle.accelerate();
  console.log(`Goingggg with speed ${vehicle.speed}`);
}
```

### Mục tiêu hướng tới 🌅
> Nguyên tắc này giúp đảm bảo tính nhất quán giữa class cha và các class kế thừa nó, đồng thời cũng giúp chúng ta có thể đoán trước được behavior của các class đó. Nếu chúng ta vi phạm nguyên tắc này có thể dẫn đến lỗi không mong muốn như ví dụ trên, quá trình maintain cũng trở nên khó khăn hơn. 
## I: Interface Segregation Principle - ISP
![](https://images.viblo.asia/b141aedd-b6b1-4979-aefb-3c22daa85b47.jpg)

> A class should not be forced to implement interfaces and methods that will not be used.

Nguyên lý này cũng không quá khó hiểu: một class không nên implement các interface và method mà nó không dùng tới. 

### Vấn đề ⚠️
Lấy ví dụ tương tự về vấn đề ở nguyên lý **L**:
```typescript
export interface Vehicle {
  turnOnEngine(): void
  accelerate(): void;
}

export class Bicycle implements Vehicle {
  accelerate(): void {
    this.speed += 5;
  }
  // ❌️ Đang phải implement một method mà nó không dùng tới
  turnOnEngine(): void {
    throw new Error("Bicycles don't have engines!");
  }
}
```
và cũng tương tự nó gây ra lỗi khi sử dụng với function `goTravelling`,  khi sử dụng cũng sẽ gặp lỗi `"Bicycles don't have engines!"`

### Giải pháp 💡

Để giải quyết chúng ta cần tách riêng nó ra interface có chức năng khởi động động cơ riêng như sau:
```typescript
export interface Vehicle {
  accelerate(): void;
}

export interface Car {
    turnOnEngine(): void;
}

export class Bicycle implements Vehicle {
  accelerate(): void {
    this.speed += 5;
  }
}
```
Bằng cách tách ra như vậy chúng ta đã tránh được lỗi và cũng không cần phải mất thời gian viết thêm method `turnOnEngine` cho class `Bicycle` để làm gì trong khi không dùng tới.

Một ví dụ khác trong source code Flash card mà các bạn hay thấy đó là ở file `base.interface.service.ts` mình có khai báo 2 interface để tượng trưng cho 2 action là Write và Read:
```typescript:src/services/base/base.interface.service.ts
export interface Write<T> {
	create(item: T | any): Promise<T>;
	update(id: string, item: Partial<T>): Promise<T>;
	remove(id: string): Promise<boolean>;
}

export interface Read<T> {
	findAll(filter?: object, options?: object): Promise<FindAllResponse<T>>;
	findOne(id: string): Promise<T>;
	findOneByCondition(filter: Partial<T>): Promise<T>;
}
```

Với các module thông thường có đầy đủ action Write và Read thì chúng ta sẽ implement cả 2 interface này:
```typescript:src/services/base/base.interface.service.ts
...
export interface BaseServiceInterface<T> extends Write<T>, Read<T> {}
```
Còn trong trường hợp chỉ cần Read hoặc Write thôi thì chúng ta sẽ chỉ đơn giản implement 1 trong 2 là được. Ví dụ như chúng ta có `LogService` dùng để đọc các log từ file log trên server hoặc bên thứ 3 nào đó:
```typescript
export class LogService implements Read<any> {
    // Implement findAll, findOne, findOneByCondition
}
```

### Mục tiêu hướng tới 🌅
> Nguyên tắc này giúp code chúng ta trở nên flexible và modularity hơn bằng cách tách các action ra thành những interface riêng biệt. Bên cạnh đó code cũng trở nên readable từ đó dễ maintain hơn.

## D: Dependency Inversion Principle - DIP
![](https://images.viblo.asia/9f14727e-9c15-45fa-99b1-406d279234cd.jpg)

> High-level modules should not depend on low-level modules. Both should depend on the abstraction.

Nguyên lý này thì quá quen thuộc với anh em sử dụng NestJS rồi 😁: các module cấp cao không nên phụ thuộc vào các module cấp thấp mà chỉ nên phụ thuộc vào sự trừu tượng.

### Vấn đề ⚠️
Chúng ta cùng ôn lại kiến thức về một trong những lý do vì sao nên sử dụng NestJS, giả sử chúng ta có Topic module như sau:
```typescript
interface Topic {
  id: string;
  name: string;
  description: string;
}

class TopicRepository {
  constructor() {}
  create(topic: Topic) {}
}

class TopicService {
  private topicRepository: TopicRepository;

  constructor() {
    this.topicRepository = new TopicRepository();
  }

  create(topic: Topic) {
    this.topicRepository.create(topic);
  }
}
```
Rõ ràng từ ví dụ trên chúng ta thấy `TopicService` bị phụ thuộc và các method của `TopicRepository`, nếu chỉnh sửa tên method `create` thành `save` thì ngay lập tức `TopicService` sẽ bị lỗi. Bên cạnh đó việc test `TopicService` một cách độc lập cũng rất là khó do chúng ta phải tạo instance của `TopicRepository` và truyền vào `TopicService` 🤒.

### Giải pháp 💡
Chúng ta cần tạo ra các abstract để cả `TopicService` và `TopicRepository` cùng implement:
```typescript
...
interface TopicRepositoryInterface {
    create(topic: Topic): Topic {}
}

class TopicRepository implements TopicRepositoryInterface {
  constructor() {}
  create(topic: Topic): topic {}
}

class TopicService {
  private topicRepository: TopicRepositoryInterface;

  constructor(repository: TopicRepositoryInterface) {
    this.topicRepository = repository
  }

  create(topic: Topic): Topic {
    return this.topicRepository.create(topic);
  }
}
```
Với cách dùng trên chúng ta có thể thực hiện những thay đổi ở `TopicRepository` nhưng vẫn tuân theo các quy tắc ở `TopicRepositoryInterface`, và khi test chúng ta chỉ cần tạo mock cho `TopicRepository` là xong. 

Ở đoạn trên nếu mình chỉnh lại như bên dưới thì các bạn có thấy quen không:
```typescript
class TopicService {
  constructor(
      @Inject()
      private readonly repository: TopicRepositoryInterface
  ) {}
  ...
}
```
Đó chỉnh xác là **NestJS** mà chúng ta hay dùng 😆
### Mục tiêu hướng tới 🌅
> Nguyên tắc này giúp code chúng ta loose coupling từ đó giúp tăng tính modularity, dễ test, bảo trì và mở rộng.
# Kết luận 📝
Vậy là chúng ta đã cùng nhau tìm hiểu qua về một trong các nguyên lý quan trọng trong lập trình. Hy vọng bài viết này sẽ giúp các bạn hiểu hơn cũng như là vận dụng được **S.O.L.I.D** hiệu quả trong quá trình lập trình.

Cảm ơn các bạn đã giành thời gian đọc bài viết, hẹn gặp lại vào các bài viết tiếp theo 🎉

# Tài liệu tham khảo 🔍
- Uehara, K.T. (2024) Solid - the simple way to understand, DEV Community. Available at: https://dev.to/kevin-uehara/solid-the-simple-way-to-understand-47im?ref=dailydev (Accessed: 24 May 2024). 
- Thelma, U. (2020) The S.O.L.I.D principles in pictures, Medium. Available at: https://medium.com/backticks-tildes/the-s-o-l-i-d-principles-in-pictures-b34ce2f1e898 (Accessed: 24 May 2024). 
- Malik, S.K. (2023) Liskov substitution principle in software design, LinkedIn. Available at: https://www.linkedin.com/pulse/liskov-substitution-principle-software-design-sanjoy-kumar-malik/ (Accessed: 24 May 2024). 
- Krishna, A. (2023) What is solid? principles for better software design, freeCodeCamp.org. Available at: https://www.freecodecamp.org/news/solid-principles-for-better-software-design/ (Accessed: 24 May 2024). 
# Change log 📓
- May 24, 2024: Init document.
- May 28, 2024: Update document and publish.