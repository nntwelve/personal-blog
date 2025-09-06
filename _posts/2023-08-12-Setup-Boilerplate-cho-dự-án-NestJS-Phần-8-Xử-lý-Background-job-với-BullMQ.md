Đây là bài viết nằm trong Series NestJS thực chiến 🎬, các bạn có thể xem toàn bộ bài viết ở link 🔗: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

----
# Đặt vấn đề 📜

![image.png](https://images.viblo.asia/722feef0-1d05-493c-8344-f7aecef60794.png)

Khi nhắc đến background job trong NestJS không ít lần chúng ta đều nghe tới package **Bull** , nó được dùng kết hợp với package **@nestjs/bull** để ứng dụng background job vào source code Nest.

Tuy nhiên theo sự thay đổi từ phía team phát triển mà hiện tại **Bull** chỉ còn ở maintain mode và không còn được phát triển thêm tính năng mới. 
> "Bull is currently in maintainance mode, we are only fixing bugs. For new features check BullMQ, a modern rewritten implementation in Typescript. You are still very welcome to use Bull if it suits your needs, which is a safe, battle tested library". Xem chi tiết [ở đây](https://www.npmjs.com/package/bull).

![image.png](https://images.viblo.asia/9916363b-68f6-4989-bbfc-71a0709deb13.png)

Ở thời điểm hiện tại của bài viết (12/08/2023) document của Nest chưa có hướng dẫn sử dụng với **BullMQ** và các tài liệu trên internet cũng chỉ ở mức cơ bản. Do đó ở phần này chúng ta sẽ cùng tìm hiểu và sử dụng package **BullMQ**, để sau này nếu dự án các bạn tham gia có cần sử dụng có thể tham khảo lại mà không cần phải mất thời gian tìm kiếm.

# Thông tin package 📦️
* "bullmq": "^3.15.8"
* "@nestjs/bullmq": "^10.0.0"

Source code của phần này sẽ nằm ở [branch part-8-background-job-with-bullmq](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-8-background-job-with-bullmq) các bán có thể tải về để sử dụng.
## 1. Cài đặt 🛠
Tiến hành cài đặt với lệnh: 

`npm i --save @nestjs/bullmq bullmq`

## 2. Cấu hình 🔩🔧
Tương tự với **Bull**, **BullMQ** cũng sử dụng **Redis** nên chúng ta cần start lên mới có thể sử dụng. Thêm vào service trong file `docker-compose`. Mình sẽ dùng thêm `redis-commander` để monitoring cũng như thao tác với dữ liệu trên Redis dễ dàng hơn. 
```docker-compose.yml
...
    flash_cards_redis:
        container_name: flash_cards_redis
        image: redis:alpine
        expose:
          - 6379
        ports:
          - 6379:6379 # Public port để lát nữa test multiple worker
        restart: unless-stopped
    flash_cards_redis_commander:
        container_name: flash_cards_redis_commander
        image: rediscommander/redis-commander:latest
        environment:
          - REDIS_HOSTS=local:flash_cards_redis:6379
        ports:
          - '8088:8081'
        depends_on:
          - flash_cards_redis
  ...
```

```shell:.env
...
REDIS_PORT=6379
REDIS_HOST=flash_cards_redis
...
```

Tiến hành start 2 service trên: `docker compose up -d`

Sau khi đã có **Redis** chúng ta sẽ tiến hành cấu hình **BullMQ** trong `AppModule`

```typescript:src/app.module.ts
import { BullModule } from '@nestjs/bullmq';
    
@Module({
	imports: [
        ...
        BullModule.forRootAsync({
			inject: [ConfigService],
			useFactory: async (configService: ConfigService) => ({
				connection: {
					host: configService.get('REDIS_HOST'),
					port: configService.get('REDIS_PORT'),
				},
			}),
		}),
    ...
```

Đây là **khác biệt đầu tiên so với Bull**, chúng ta dùng `connection` thay vì `redis` như trong doc của Nest:

![image.png](https://images.viblo.asia/921b09ba-88b7-4c23-b4e5-6d17c49baeaf.png)

Việc cấu hình chỉ đơn giản vậy thôi, **nếu** bạn nào **gặp lỗi thì** khả năng cao là do kết nối tới redis chưa chính xác. **Kiểm tra lại biến môi trường** cũng như **xem redis đã start thành công hay chưa**.  

## 3. Tìm hiểu về lifecycle 🧬
Để hiểu tường tận hơn về **BullMQ**, như thường lệ chúng ta sẽ tìm hiểu về **Lifecycle** của nó trong quá trình xử lý Job. 

![](https://images.viblo.asia/dca0315c-2c00-420b-b446-10e429df2bdd.png)


Chú thích:
* Khi một Job được thêm vào queue bởi method `add`, tùy theo các option mà sẽ thuộc một trong 3 state sau:
    *  **wait**: danh sách **các Job vừa được add** và đang chờ được chuyển sang trạng thái **active**. Các **Job bị lỗi** trong khi **active** và có **cấu hình retry** cũng sẽ được đưa về state này để chờ thử lại. 
    * **delayed**: đang chờ để được xử lý, sau khi chờ xong sẽ chuyển sang **wait**, thường thấy khi sử dụng option `delay`. Các **Job bị lỗi** trong khi **active** và có **cấu hình retry với back-off strategy** cũng sẽ được đưa trở về state này, chờ trước khi được chuyển sang **wait** để thử lại.
    * **prioritized**: biểu thị các **Job có priority cao** sẽ **được xử lý trước**, thường thấy khi sử dụng với option `priority`. State này theo mình tìm hiểu thì cơ chế sau khi add cũng sẽ tương tự như status **wait** và chỉ thêm key priority. Có lẽ đó là lý do tại sao giá trị đầu tiên truyền vào method `getJobs` không có value `priority` (hình dưới), thay vào đó chúng ta phải dùng value `wait` để lấy ra kết quả có nó. Một lưu ý khác là **priorities** bắt đầu **từ 0 đến 2^21** và **0 là ưu tiên cao nhất**.
![image.png](https://images.viblo.asia/15e72fff-e570-488c-b2ea-fd718e1693d6.png)
* Tiếp sau 3 state trên là **active** là tập hợp các job đang được xử lý. Các job ở đây sẽ thực thi đến khi nào hoàn thành hoặc gặp lỗi.
* Sau cùng sẽ là một trong 2 trạng thái **completed** hoặc **failed** tùy theo kết quả ở state **active**.

Dưới đây là quá trình xử lý các job ứng với các config khác nhau (xem trên redis-commander):

* ⛓️🚀 Quá trình (thứ tự) một Job được xử lý đơn giản hoặc dùng option **priority** như sau: 

    *added* ➡️ *waiting* ➡️ *active* ➡️ *completed* ➡️ *drained*

![image.png](https://images.viblo.asia/b6292229-3f64-4626-a34d-9871153233c4.png)

* ⛓️⏳ Quá trình một Job được xử lý với option **delay** như sau:

    *added* ➡️ ***delayed*** ➡️ *waiting* ➡️ *active* ➡️ *completed* ➡️ *drained*

![image.png](https://images.viblo.asia/6f0d28b7-10a8-4ab8-9fcf-d41144cde348.png)

.
* ⛓️🔁 Quá trình một Job có lỗi trong quá trình xử lý với Back-off strategy như sau:

    *added* ➡️ *waiting* ➡️ ***active*** ➡️ ***delayed*** ➡️ *waiting* ➡️ *active* ➡️ ***failed*** ➡️ *retries exhausted* ➡️ *drained*

![image.png](https://images.viblo.asia/823e724c-a90b-41dc-bd49-54a3c8cb2dc7.png)

## 4.Cách sử dụng cơ bản 💡
Phần này chúng ta sẽ lấy ví dụ với **module flash-card**, khi user tạo flash-card mới sẽ có property image làm hình minh họa, nếu chúng ta **lưu trực tiếp image** đó vào mà không xử lý thì sẽ **gây hao tốn tài nguyên** trong trường hợp những **image có dung lượng lớn**. Do đó để tối ưu chúng ta cần **tiền xử lý** để **giảm bớt dung lượng** của image trước khi lưu vào server, mình sẽ gọi là **optimize image** cho ngắn gọn.

### 4.1 Queue 📧📧📧
> Về mặt khái niệm thì queue chính là một waiting list (hàng chờ) chứa các job đang chờ để được xử lý. Khái niệm này tương tự với khái niệm queue các bạn thường thấy ở các message broker khác.

Để implement logic trên mình sẽ tạo ra 1 queue để lưu trữ các job dùng cho việc optimize image ở **FlashCardModule**:
```typescript:src/modules/flash-cards/flash-cards.module.ts
import { BullModule } from '@nestjs/bullmq';
...
@Module({
	imports: [
		...
		BullModule.registerQueue({
			name: 'image:optimize',
            prefix: 'flash-cards'
		})
    ]
    ...
export class FlashCardModule {}
```
Có nhiều option chúng ta có thể truyền vào queue, ở đây thường mình sẽ sử dụng:
* **name**: là tên của Queue, **bắt buộc phải có** để processor có thể nhận biết.
* **prefix**: dùng thêm vào tiền tố cho các job trong Queue này.
* **connection**: dùng chỉ định kết nối đến một redis cụ thể trong trường hợp đang kết nối đến nhiều redis cùng lúc. 
* **processor**: thường được dùng để chạy Sandbox processor, chúng ta sẽ nói để sau.

Sau khi đã register queue xong, chúng ta có thể vào để xem BullMQ cấu trúc nó như thế nào trong **redis** thông qua **redis-commander** chúng ta vừa cài đặt khi nảy. Mình chạy nó ở port 8088 nên sẽ truy cập http://localhost:8088. 

![image.png](https://images.viblo.asia/9f032f3f-9aac-486e-8478-122348be6b85.png)

Có thể thấy hình trên được biểu thị dưới dạng như cấu trúc tree folder. 
* Đầu tiên là flash-cards từ config `prefix`.
* Nhờ vào option `name` phía trên có sử dụng tính năng gọi là key namespace (syntax `:`) trong redis, giúp cho cấu trúc của chúng ta được phân chia rõ ràng hơn . Việc này giúp chúng ta có thể dễ dàng monitoring và debugging.

### 4.2 Job 📧
Khi đã có queue rồi thì chúng ta chỉ cần thao tác thêm job vào trong, để làm việc đó mình sẽ thêm logic vào trong **FlashCardService**. Khi nhận đc image thì chúng ta sẽ tiến hành add job vào queue.
```typescript:src/modules/flash-cards/flash-cards.service.ts
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
...

@Injectable()
export class FlashCardsService extends BaseServiceAbstract<FlashCard> {
	constructor(
        ...
		@InjectQueue('image:optimize')
		private readonly image_optimize_queue: Queue,
	) {
		super(flash_cards_repository);
	}

	async createFlashCard(
		create_dto: CreateFlashCardDto,
		file: Express.Multer.File,
	): Promise<FlashCard> {
		const flash_card = await this.flash_cards_repository.create(create_dto);
        // 🔽 Thêm ở đây
		await this.image_optimize_queue.add('optimize-size', {
			file,
			id: flash_card._id,
		});
		return flash_card;
	}
}
```
Giải thích:
* Như các bạn thấy chúng ta dùng `InjectQueue` với giá trị được truyền vào là value của option `name` ở bước trước đó để inject queue vừa register ở module. **Lưu ý** rằng dù chúng ta có **config `prefix`** ở **bước register** nhưng option đó **không liên quan** đến phần này.
*  Trong method của service chúng ta sẽ dùng **method `add`** để thêm job vào Queue. Method này nhận vào **3 tham số**:
    *  Tham số đầu tiên là **name** của job, thường dùng khi xử lý logic trong 1 queue có nhiều trường hợp xảy ra. 
    *  Tham số thứ 2 là **data** chứa các data liên quan dùng cho xử lý job, ở đây chúng ta sẽ truyền vào image cần optimize và id của flash card để lưu image vào sau khi đã xử lý
    *  Tham số cuối cùng là **opts** hay **job options**, sẽ dùng để cấu hình các option như: `delay`, `priority`, `back-off`,... mà chúng ta đã nói đến ở phần trên.

Các bạn nhớ chỉnh sửa lại controller để gọi đến method mới của service. Sau đó tiến hành gọi thử API tạo flash-card để kiểm tra xem job đã được thêm vào chưa. 

![image.png](https://images.viblo.asia/5f6a77fa-f63b-4054-9b09-370ac38913ba.png)

Có thể thấy bên trong đang có một job đang ở trạng thái **wait** với giá trị là 1 tương ứng với job có id 1. Các bạn có thể bấm vào **events** để chi tiết những gì đang diễn ra.

![image.png](https://images.viblo.asia/60758971-8b4b-4e49-af9c-39ee6dadea7f.png)

Vậy là chúng ta đã thêm được *job* vào *queue*, việc tiếp theo cần làm là cấu hình *worker* để có thể lấy job từ queue ra xử lý.

### 4.3 Worker 👷‍♂️ 👷‍♀️
Để tạo *worker* hay *processor* chúng ta sẽ dùng decorator `@Processor`, tiến hành tạo **ImageOptimizationProcessor**
```typescript:src/modules/flash-cards/queues/flash-cards.processor.ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';

@Processor('image:optimize')
export class ImageOptimizationProcessor extends WorkerHost {
	private logger = new Logger();
	async process(job: Job<any, any, string>, token?: string): Promise<any> {
		switch (job.name) {
			case 'optimize-size':
				const optimzied_image = await this.optimizeImage(job.data);
				console.log({ optimzied_image });
				return optimzied_image;

			default:
				throw new Error('No job name match');
		}
	}

	async optimizeImage(image: unknown) {
		this.logger.log('Processing image....');
		return await new Promise((resolve) =>
			setTimeout(() => resolve(image), 30000),
		);
	}
}
```
Giải thích:
* Phạm vi bài viết này mình sẽ không implement logic xử lý image thay vào đó dùng setTimeout để giả lập thời gian xử lý. 
* Để *worker* có thể xử lý đúng *queue* chúng ta cần truyền vào đúng *queue name* cho `@Processor`.
* **Khác với Bull**, **BullMQ** sẽ **không** còn **dùng decorator `@Process`** để xử lý cho các *job* (mặc định hoặc dựa vào *job name*) mà **thay vào đó** chúng ta phải **extends `abstract class WorkerHost`** sau đó **implement logic** cho **method `process`**. Vì thế cũng sẽ không còn các method xử lý riêng theo từng job name như đã từng dùng với decorator `@Process`. Thay vào đó BullMQ đưa ra 2 hướng giải quyết sau:
    * Dùng **switch case** bên trong method `process` để xử lý tương ứng với từng *job name*. 
    * Tạo thêm các *queue* ứng với các *job name* mà chúng ta cần xử lý (cách này được khuyên dùng hơn).
 
> Giải thích thêm thì theo như team dev của Bull thì việc dùng `@Process` ở Bull sẽ gây ra nhiều confusion do đó họ đã loại bỏ ở BullMQ và về bản chất thì logic bên trong `@Process` cũng chỉ là dùng switch case. Tham khảo thêm từ [docs ở đây](https://docs.bullmq.io/patterns/named-processor).

Để *woker* có thể lấy *job* từ *queue* ra để xử lý chúng ta cần thêm nó vào provider vì hiện tại chúng ta chỉ vừa tạo file chứ chưa kết nối vào bất kỳ đâu. Đây là **bước quan trọng**, nếu **thiếu bước này thì *worker* sẽ không hoạt động.**
```typescript:src/modules/flash-cards/flash-cards.module.ts

import { ImageOptimizationProcessor } from './queues/flash-cards.processor';
...
@Module({
	...
    providers: [
		...
        ImageOptimizationProcessor
	],
})
export class FlashCardsModule {}
```

Save lại và kiểm tra kết quả ở console, có thể thấy job đã được lấy ra xử lý và sau 30 giây trả về kết quả.

![image.png](https://images.viblo.asia/f4d1d7fa-a89f-482d-a88e-9020260d47a1.png)

Trong quá trình job được xử lý thì sẽ được chuyển vào **active** như hình dưới
![image.png](https://images.viblo.asia/eced3914-c240-43cb-95e4-69834ff5afdf.png)

Sau khi job đã xử lý thành công thì sẽ nằm trong **completed**

![image.png](https://images.viblo.asia/41ff9262-2e8d-4429-885b-596c2b2988f7.png)

### 4.4 Concurrency và Multiple Worker
Mặc định mỗi *worker* chỉ xử lý 1 job trong cùng một khoảng thời gian, các bạn có thể thử gọi 2 lần sẽ thấy 1 job đang trong **active** và 1 job đang trong **wait**.

![image.png](https://images.viblo.asia/14c79fb9-e0a8-41f8-aac0-a3a222db14bb.png)

Trong một số trường hợp chúng ta mong muốn tăng số lượng job chạy đồng thời để đáp ứng các yêu cầu nhất định của dự án. Để làm được việc đó chúng ta có thể dùng các cách sau:
* Dùng option **concurrency để cho phép worker xử lý song song nhiều job**. Tuy nhiên phải cẩn thận, nếu quá nhiều job cùng chạy mà cấu hình server không đáp ứng được có thể phát sinh vấn đề.
* **Tăng thêm số lượng worker** cho queue, Bull/BullMQ được thiết kế theo cấu trúc phân tán nên các thành phần của nó có thể nằm riêng biệt với nhau, giúp tăng khả năng mở rộng theo chiều ngang. Ổn hơn cách trên nhưng sẽ tốn thêm chi phí ứng với các worker.
* Kết hợp cả 2 cách trên **vừa tăng worker vừa dùng option concurrency**. 
* Sử dụng **Sandboxed processors**, chúng ta sẽ nói đến ở phần tiếp theo.

> Mỗi cách đều có ưu nhước điểm riêng nên không có cách nào là tối ưu nhất, tùy thuộc theo nhu cầu dự án mà các bạn chọn cách thích hợp nhất.

Chúng ta sẽ lần lượt đi qua các ví dụ, đầu tiên là dùng option **concurrency**
```typescript:src/modules/flash-cards/queues/flash-cards.processor.ts
...
@Processor('image:optimize', {
	concurrency: 2, // <=== Thêm vào đây
})
export class ImageOptimizationProcessor extends WorkerHost { ... }
```
Chỉ đơn giản vậy thôi, chúng ta sẽ thử gọi API 3 lần xem chuyện gì sẽ xảy ra. Như hình bên dưới có thể thấy được có 2 job đang active và 1 job đang wait, đúng như những gì chúng ta muốn. 

![image.png](https://images.viblo.asia/952d19bf-de66-4719-97cc-8ca790db27f5.png)

Ví dụ tiếp theo là tăng số lượng worker kết hợp với concurrency. Mình sẽ tạo một dự án Nest khác làm worker để minh họa cho tính phân tán của Bull/BullMQ. Các bạn có thể tải về source code của [worker ở đây](https://github.com/nntwelve/BullMQ-Worker). Đầu tiên như ở phần cấu hình chúng ta tạo connect  ở AppModule sau đó register vào queue cần cho worker chạy. Sau đó chúng ta cũng tạo file **flash-cards.processor.ts** và copy toàn bộ nội dung của class **ImageOptimizationProcessor** vào.
> Để cho nhanh thì code ở worker mình không tách ra module flash-card mà viết thẳng bên ngoài luôn.
```typescript:bullmq-worker/src/app.module.ts
import { ImageOptimizationProcessor } from './flash-cards.processor';
...
@Module({
  imports: [
    BullModule.forRoot({
      connection: {
        host: 'localhost',
        port: 6379,
      },
    }),
    BullModule.registerQueue({
      name: 'flash-cards',
      prefix: 'flash-cards',
    }),
  ],
  controllers: [AppController],
  providers: [AppService, ImageOptimizationProcessor],
})
export class AppModule {}
```

Tiến hành start worker với lệnh `npm run start:dev`. Vì mỗi worker hiện có option concurrency là 2 nên chúng ta sẽ gọi API 5 lần để xem kết quả tường minh hơn.

![image.png](https://images.viblo.asia/e7682281-9a87-4f4a-9a64-9e4a4ff73d08.png)

Từ hình trên có thể thấy được có 4 job đang active và 1 job đang wait như chúng ta dự định. Chúng ta có thể thay đổi option **concurrency** ở các *worker* để cấu hình mỗi *worker* có thể chạy số lượng job riêng.

Tuy nhiên ở đây có một vấn đề, khi chúng ta xem console ở các worker không biết được job nào đang được xử lý cũng như job nào vừa hoàn thành hoặc đang gặp vấn đề dẫn đến lỗi. Chúng ta cần một thứ gì đó giúp monitor quá trình xử lý (lifecyle) *job*, trên môi trường production tùy theo dự án có thể sẽ có rất nhiều job nên nếu chỉ dựa vào **redis-commander** thì không đủ. Chúng ta sẽ đến với phần tiếp theo để giải quyết vấn đề trên.

### 4.5 Event Listener 👂💬
Hiểu một cách đơn giản thì mỗi khi 1 job chuyển sang trạng thái thì nó sẽ emit ra một event, chúng ta có thể sử dụng các Event Listeners để lắng nghe các sự kiện đó.

Tương tự với Bull thì BullMQ cũng sẽ có 2 loại:
* **Local event listeners**: lắng nghe các sự kiện xảy ra trong lifecycle của job của riêng instance đó. Ví dụ cụ thể trong **worker 1** của chúng ta, khi 1 job được nó xử lý thì các event của job đó chỉ có thể listen trong **worker 1**, **worker 2** sẽ không thể listen được   
* **Global event listeners**: lắng nghe các sự kiện xảy ra trong lifecycle của toàn bộ instance trong queue. Ngược lại với ở trên chúng ta có thể lắng nghe sự kiện của cả 2 worker. 

Để trực quan hơn chúng ta sẽ cùng đi đến phần ví dụ minh họa bên dưới.

#### **Local event listeners** 🏡
```typescript:src/modules/flash-cards/queues/flash-cards.processor.ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';

@Processor('image:optimize', {
	concurrency: 2
})
export class ImageOptimizationProcessor extends WorkerHost {
	@OnWorkerEvent('active')
	onQueueActive(job: Job) {
		this.logger.log(`Job has been started: ${job.id}`);
	}

	@OnWorkerEvent('completed')
	onQueueComplete(job: Job, result: any) {
		this.logger.log(`Job has been finished: ${job.id}`);
	}

	@OnWorkerEvent('failed')
	onQueueFailed(job: Job, err: any) {
		this.logger.log(`Job has been failed: ${job.id}`);
		this.logger.log({ err });
	}

	@OnWorkerEvent('error')
	onQueueError(err: any) {
		this.logger.log(`Job has got error: `);
		this.logger.log({ err });
	}
    ...
```
Giải thích:
* **Đây cũng là một cải tiến khác so với Bull**, thay vì mỗi trạng thái ứng với 1 decorator, BullMQ gôm lại thành 1 decorator duy nhất là  `OnWorkerEvent` và chúng ta chỉ cần truyền vào trạng thái cần listen.
* Ứng với mỗi state thì sẽ có param khác nhau để chúng ta có thể thao tác. Còn nhiều state khác mà mình không thể liệt kê hết, các bạn có thể xem thêm [ở đây](https://api.docs.bullmq.io/interfaces/WorkerListener.html).

Tiến hành gọi API để kiểm tra kết quả, kết quả ở console của worker 1 như bên dưới, có thể thấy khi xử lý đến từng trạng thái của job thì các method tương ứng mà chúng ta khai báo sẽ được gọi.

![image.png](https://images.viblo.asia/14f9c879-1c13-46fd-8aa3-09ef70bf0abf.png)

Vì là local event nên bên worker 2 sẽ không bị ảnh hưởng, vẫn hiển thị log như lúc đầu. Việc này giúp tách biệt được log giữa các worker giúp chúng ta dễ debug hơn.

![image.png](https://images.viblo.asia/be02fcb4-da06-4531-96b5-7be6eb5a690b.png)

#### Global event listeners 🌐
Trong trường hợp chúng ta cần một listener lắng nghe sự kiện của tất cả worker thì chúng ta sẽ dùng decorator `OnQueueEvent` (ở Bull là `OnGlobalQueue*`). Tuy nhiên theo như document, để sử dụng `OnQueueEvent` chúng ta cần đặt nó trong class được decorate với decorator `QueueEventsListener`. Ở worker 2 mình sẽ tạo file **app.listener.ts** để lấy ví dụ:
```typescript:bullmq-worker/src/app.listener.ts
import {
  OnQueueEvent,
  QueueEventsHost,
  QueueEventsListener,
} from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';

@QueueEventsListener('image:optimize')
export class AppListener extends QueueEventsHost {
  private logger = new Logger(AppListener.name);
  @OnQueueEvent('active')
  onQueueEventActive(job: any) {
    this.logger.log(`Job has been started!`);
    this.logger.log(job);
  }

  @OnQueueEvent('completed')
  onQueueEventCompleted(job: Job, result: any) {
    this.logger.log(`Job has been completed!!`);
  }
}
```
Tương tự processor, để có thể sử dụng listener chúng ta cần thêm nó vào provider
```typescript:bullmq-worker/src/app.module.ts
...
import { AppListener } from './app.listener';

@Module({
  ...
  providers: [
    AppService, ImageOptimizationProcessor, AppListener,
  ],
})
export class AppModule {}
```
Thử gọi API 3 lần và xem kết quả ở console của worker 2, từ hình dưới có thể thấy worker 1 xử lý 1 job, worker 2 xử lý 1 job và AppListener của chúng ta đề lắng nghe được hết.

![image.png](https://images.viblo.asia/f3c8c42c-94ff-429e-9033-f355fb200d79.png)

Dựa vào 2 ví dụ trên thì các bạn có thể phát triển tùy theo nhu cầu của dự án để tối ưu trong việc monitor quá trình các job được xử lý.

> Chúng ta hoàn toàn có thể config một Global event listeners riêng chỉ để lắng nghe các sự kiện, hạn chế được các vấn đề khi dùng chung với worker.

### 4.6 Rate Limiting 🚧
Đối với một số loại dự án mà trong logic xử lý job có gọi đến service của khách hàng thì chúng ta cần chú ý về tần suất gọi đi. 

**Ví dụ:** Worker của chúng ta có cấu hình **concurrency là 200** và thời gian để mỗi job **xử lý khoảng 0.5-1s**, bình thường trong **mỗi 60 giây** chúng ta chỉ có **khoảng 100 request** gọi đến service của họ  (**tối đa trong 60 giây sẽ xử lý được 200 * 2 * 60 = 24000**). Đột nhiên một ngày lượng truy cập của người dùng **tăng đột biến**, giả sử server của chúng ta thiết kế tối ưu nên vẫn có thể xử lý ổn thỏa ✅, có điều số lượng job **xử lý trong 60 giây** sẽ **tăng tối đa lên 24000 request** ⚠️. Trong trường hợp này không ít thì nhiều cũng sẽ ảnh hưởng đến service của khách hàng nếu như họ không thiết kế tốt, tệ hơn là sẽ gây sập service của khách hàng ❌. 

Từ ví dụ trên có thể thấy chúng ta cần phải giới hạn lại các job xử lý trong một khoảng thời gian để hạn chế vấn đề trên. Giải pháp cho vấn đề này là **Rate Limiting**, chức năng này cho phép Bull/BullMQ chỉ xử lý một số lượng job nhất định trong một khoảng thời gian. Cách sử dụng thì chúng ta sẽ thêm vào worker như sau, mình lấy `ImageOptimizationProcessor` làm ví dụ: **có thể xử lý đồng thời 200 job nhưng tối đa trong 60s chỉ xử lý 200 job**.

```typescript:src/modules/flash-cards/queues/flash-cards.processor.ts
...
@Processor('image:optimize', {
	concurrency: 200,
	limiter: {
		max: 200,
		duration: 60000,
	},
})
export class ImageOptimizationProcessor extends WorkerHost { ... }
```

    > **Lưu ý** theo như tài liệu của BullMQ `The rate limiter is global, so if you have for example 10 workers for one queue with the above settings, still only 10 jobs will be processed by second.`: có nghĩa là **rate limiter sẽ áp dụng trên tất cả worker** chứ không phải cho riêng worker nào **mặc dù BullMQ lại yêu cầu chúng ta config theo worker** chứ** không phải queue** nên khá là khó hiểu. Điều chúng ta cần lưu ý là các worker có thể có config rate limiter khác nhau. Ví dụ worker 1 config max là 200 nhưng worker 2 là 250 thì sẽ lấy tối đa là 250, không quan trọng thứ tự start worker.
### 4.7 Sandboxed Processors
Trong quá trình sử dụng Bull/BullMQ sẽ có lúc các bạn gặp lỗi: **`Missing lock for job queue...`** sau đó job tự restart lại và chạy lại từ đầu (double process).

Lỗi này có 2 nguyên nhân:
* Node process đang xử lý job bị terminate (nguyên nhân này ít gặp)
* Do job chúng ta cần xử lý quá nặng, dẫn đến chiếm dụng rất nhiều CPU làm cho **Event Loop** của NodeJS bị **block**, khi đó Bull/BullMQ không thể renew job lock. Khi không thể renew thì job sẽ bị coi là **stalled** và tự động restart lại.
> Giải thích thêm về job lock thì mỗi job khi được worker lấy ra khỏi queue sẽ được lock lại để xử lý, tránh các worker khác lại lấy nó ra. Việc lock job được xử lý internal bởi **lockDuration** (mặc định 30 giây) và **lockRenewTime**, nhưng ở trên do việc xử lý tốn quá nhiều thời gian và chiếm dụng Event Loop dẫn đến Bull/BullMQ không thể renew job lock nên bị đưa vào **stalled**.

Để giải quyết lỗi trên chúng ta có thể dùng các cách sau:
* Chỉnh sửa lại logic code để tránh block Event Loop quá lâu, chia thành các phần nhỏ để xử lý.
* Điều chỉnh giá trị của **lockDuration** và **lockRenewTime**, theo mình các bạn chỉ nên dùng cách này khi biết chính xác khoảng thời gian cần xử lý của phần logic đã gây ra lỗi để điều chỉnh phù hợp. Vi dụ logic lấy dữ liệu và xử lý cần 40 giây xử lý xong, chúng ta tăng **lockDuration** lên 45 giây thì chạy ổn, nhưng đến lúc nào đó dữ liệu tăng và xử lý cần 50 giây mới xong thì lại phải chỉnh sửa.
* Dùng **Sandboxed processors**, theo như tài liệu thì các worker sẽ chạy processor trên các process riêng nên sẽ không ảnh hưởng đến process xử lý job lock.

Để lấy ví dụ mình sẽ chỉnh sửa lại `ImageOptimizationProcessor`, cho method `optimizeImage` thực hiện loop và in ra giá trị liên tục để gây ra lỗi:
```typescript:src/modules/flash-cards/queues/flash-cards.processor.ts
@Processor('image:optimize', {
	// concurrency: 1, // Tạm thời tắt concurrency
	lockDuration: 3000, // Giảm lockDuration về 3s thay vì 30s mặc định
})
export class ImageOptimizationProcessor extends WorkerHost {
	...
	async optimizeImage(image: unknown) {
        // Sẽ blocking Event Loop ⭕️
		for (let index = 0; index < 10e5; index++) {
			const progress = ((index * 100) / 10e5).toFixed(2);
			this.logger.log(`${progress}%`);
		}
		return await new Promise((resolve) =>
			setTimeout(() => resolve(image), 1000),
		);
	}
}
```

Thử gọi API các bạn sẽ thấy khi chạy thì thấy job chạy xong nhưng vẫn báo lỗi và tự động restart (tùy theo trường hợp có thể chạy được một phần thì báo lỗi và restart luôn), sau vài lần thì sẽ chuyển thành failed như hình dưới:

![image.png](https://images.viblo.asia/ee03c377-7b36-4bbd-a2d8-fa4d8363c04a.png)

Để giải quyết được vấn đề trên mình sẽ dùng **Sandboxed Processors** để xử lý theo như [tài liệu từ BullMQ](https://docs.bullmq.io/guide/workers/sandboxed-processors), tạo file **flash-cards.sandbox-processor.ts**. Logic sẽ không có gì thay đổi so với `ImageOptimizationProcessor`, chỉ có thay decorator `@Processor` thành `export default`.
```typescript:src/modules/flash-cards/queues/flash-cards.sandbox-processor.ts
import { Logger } from '@nestjs/common';
import { SandboxedJob } from 'bullmq';

export default async function (job: SandboxedJob) {
	const logger = new Logger('Flash Card Processor');
	switch (job.name) {
		case 'optimize-size':
			const optimzied_image = await optimizeImage(job.data);
            logger.log('DONE');
			return optimzied_image;

		default:
			throw new Error('No job name match');
	}

	async function optimizeImage(image: unknown) {
		for (let index = 0; index < 10e5; index++) {
			const progress = ((index * 100) / 10e5).toFixed(2);
			logger.log(`${progress}%`);
		}
		return await new Promise((resolve) =>
			setTimeout(() => resolve(image), 1000),
		);
	}
}
```

Cập nhật lại `FlashCardsModule` thay thế `ImageOptimizationProcessor` thành processor chúng ta vừa tạo

```typescript:src/modules/flash-cards/flash-cards.module.ts
import { join } from 'path';
...

@Module({
	imports: [
		...
		BullModule.registerQueue({
			...
			processors: [ // ⏪️ Thêm vào đây ⏬
				{
					concurrency: 1, // Các bạn có thể cấu hình processor chạy đồng thời
					path: join(__dirname, 'queues/flash-cards.sandbox-processor.js'),
				},
			],
		}),
	],
	providers: [
		...
		// ImageOptimizationProcessor, // <=== Comment để loại bỏ
	],
    ...
})
export class FlashCardsModule {}
```

Thử gọi lại và xem kết quả, có thể thấy job đã được chạy thành công. 

![image.png](https://images.viblo.asia/f2e05f5b-8183-43ed-b10f-0c553c5d67b8.png)

> Lưu ý: khi sử dụng **Sandboxed processors** chúng ta sẽ **không thể** sử dụng **Dependency Injection (và IoC container)** vì đang chạy trên forked process. Vì thế nếu các bạn phải tự khởi tạo các dependency nếu cần sử dụng.

### 4.8 Flows

![image.png](https://images.viblo.asia/45fdf87d-97d3-4d23-b445-7cd0eb19a81a.png)

Đây là một **tính năng hay** và **chỉ có trên BullMQ**, chức năng này giúp chúng ta xử lý các job theo **parent - child relationships**, có nghĩa là các job được định nghĩa là **parent** sẽ ở trạng thái chờ (**waiting-children** state) và nó chỉ được xử lý khi và chỉ khi **tất các job children** được xử lý **thành công**.  

![image.png](https://images.viblo.asia/bd984254-219f-4886-b830-2afd53bed60a.png)

Để lấy ví dụ chúng ta sẽ thêm vào logic xử lý image ở trên: *kiểm tra image có vi phạm chính sách và tiêu chuẩn cộng đồng hay không*. Sau khi quá trình kiểm tra và optimize image thành công thì sẽ lưu vào storage service và cập nhật database. 

Theo như tài liệu, chúng ta sẽ bắt đầu bằng việc tạo **FlowProvider**, đầu tiên tạm thời loại bỏ code liên quan đến queue khi nảy và register lại với `registerFlowProducer` ở `FlashCardsModule`

```typescript:src/modules/flash-cards/flash-cards.module.ts
import { BullModule } from '@nestjs/bullmq';
...

@Module({
	imports: [
		MongooseModule.forFeature([...]),
        // 🔄 đổi registerQueue thành registerFlowProducer
		BullModule.registerFlowProducer({
			name: 'image:upload',
			prefix: 'flash-cards',
		}),
	],
	controllers: [FlashCardsController],
	providers: [
		FlashCardsService,
		{
			provide: 'FlashCardsRepositoryInterface',
			useClass: FlashCardsRepository,
		},
	],
})
export class FlashCardsModule {}
```

Tương tự ở `FlashCardsService`, chúng ta sẽ thay `@InjectQueue` bằng `@InjectFlowProducer`, sau đó thay đổi logic add *job* vào *queue* thành *flows*

```typescript:src/modules/flash-cards/flash-cards.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { InjectFlowProducer } from '@nestjs/bullmq';
import { FlowProducer } from 'bullmq';
import { BaseServiceAbstract } from 'src/services/base/base.abstract.service';
import { FlashCard } from './entities/flash-card.entity';
import { FlashCardsRepositoryInterface } from './interfaces/flash-cards.interface';
import { CreateFlashCardDto } from './dto/create-flash-card.dto';

@Injectable()
export class FlashCardsService extends BaseServiceAbstract<FlashCard> {
	constructor(
		@Inject('FlashCardsRepositoryInterface')
		private readonly flash_cards_repository: FlashCardsRepositoryInterface,
        // 🔄
		@InjectFlowProducer('image:upload')
		private readonly image_upload_flow: FlowProducer,
	) {
		super(flash_cards_repository);
	}

	async createFlashCard(
		create_dto: CreateFlashCardDto,
		file: Express.Multer.File,
	): Promise<FlashCard> {
		const flash_card = await this.flash_cards_repository.create(create_dto);

		await this.image_upload_flow.add({
			name: 'uploading-image',
			queueName: 'image:upload',
			data: { id: flash_card._id },
			children: [
				{
					name: 'optimize-size',
					data: { file },
					queueName: 'image:optimize',
                    opts: {
						delay: 1000,
					},
				},
				{
					name: 'check-term',
					data: { file },
					queueName: 'image:check-valid',
				},
				{
					name: 'check-policy',
					data: { file },
					queueName: 'image:check-valid',
				},
			],
		});
		return flash_card;
	}
}
```
Giải thích:
* Cấu trúc của *flows* sẽ khác với lúc đầu, chúng ta sẽ chỉ định *queue* của **parent**  trong lúc `add`. 
* **name** và **data** thì tương tự, vẫn là tên *job* và dữ liệu truyền vào.
* **children** chính là danh sách các **children job** cần xử lý. Nó được thiết kế các element bên trong có cấu trúc sẽ tương tự như **parent**, vì thế chúng ta **có thể có các children bên trong children** để thực hiện các job nhỏ hơn.
* **opts** bao gồm các option tương tự khi dùng với job queue thông thường.

Tiến hành gọi lại API tạo flash-card và xem kết quả ở redis-commander:

![image.png](https://images.viblo.asia/c98ab32a-7db0-42d1-8326-ac8cef4f1d94.png)

Có thể thấy ở hình trên, có 4 job được tạo ra tương ứng với những gì chúng ta truyền vào method `add` của *flows*: 2 job check policy, 1 job optimize đang ở trạng thái **wait** và parent job upload đang ở trạng thái **waiting-children**. 

Do lúc nảy chúng ta đã xóa hết worker nên giờ sẽ lần lượt thêm lại để cho chúng pick job ra xử lý:
```typescript:src/modules/flash-cards/queues/image-verification.processtor.ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';

@Processor('image:check-valid') // Các bạn vẫn có thể cấu hình concurrency và rate limiter như bình thường
export class ImageVerificationProcessor extends WorkerHost {
	...
	async process(job: Job<any, any, string>, token?: string): Promise<any> {
		switch (job.name) {
			case 'check-term':
				return await this.checkTerm(job.data);
			case 'check-policy':
				return await this.checkPolicy(job.data);

			default:
				throw new Error('No job name match');
		}
	}

	async checkTerm(image: unknown) {
		// Do something with the job here
		this.logger.log('Start checking term...');
		return await new Promise((resolve) =>
			setTimeout(() => resolve(true), 5000),
		);
	}

	async checkPolicy(image: unknown) {
		this.logger.log('Start checking policy...');
		return await new Promise((resolve) =>
			setTimeout(() => resolve(true), 8000),
		);
	}
}
```
```typescript:src/modules/flash-cards/queues/image-optimization.processor.ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';

@Processor('image:optimize')
export class ImageOptimizationProcessor extends WorkerHost {
	...
	async process(job: Job<any, any, string>, token?: string): Promise<any> {
		switch (job.name) {
			case 'optimize-size':
				const optimzied_image = await this.optimizeImage(job.data.file);
				return optimzied_image;
			default:
				throw new Error('No job name match');
		}
	}

	async optimizeImage(image: unknown) {
		return await new Promise((resolve) =>
			setTimeout(() => resolve(image), 10000),
		);
	}
}
```

Sau khi có 2 worker cho children job, để worker có thể lấy job ra xử lý chúng ta sẽ làm tương tự như ở phần trước: register queue và thêm vào provider.
```typescript:src/modules/flash-cards/flash-cards.module.ts
import { ImageOptimizationProcessor } from './queues/image-optimization.processor';
import { ImageVerificationProcessor } from './queues/image-verification.processtor';
...
@Module({
	imports: [
        ...
        BullModule.registerQueue({
			name: 'image:optimize',
			prefix: 'flash-cards',
		}),
		BullModule.registerQueue({
			name: 'image:check-valid',
			prefix: 'flash-cards',
		}),
    ],
    ...
	providers: [
		...
        ImageOptimizationProcessor,
		ImageVerificationProcessor,
	],
...
```
Save lại và kiểm tra kết quả:
![image.png](https://images.viblo.asia/f39220ec-4e56-482a-9cd8-244d01fc6e10.png)

Tại thời điểm tất cả các children job completed, parent job sẽ chuyển trạng thái từ **waiting-children** sang **wait**. Vậy tại sao lại là **wait** mà không phải là **active**? Nguyên nhân đơn giản là do parent job vẫn là một job thông thường nên chúng ta vẫn cần worker để xử lý. Tiến hành tạo thêm worker để xử lý image uploading
```typescript:src/modules/flash-cards/queues/image-uploading.processor.ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { FlashCardsService } from '../flash-cards.service';

@Processor('image:upload')
export class ImageUploadingProcessor extends WorkerHost {
	constructor(private readonly flash_cards_service: FlashCardsService) {
		super();
	}
	...
	async process(job: Job<any, any, string>, token?: string): Promise<any> {
		switch (job.name) {
			case 'uploading-image':
				const children_results = await job.getChildrenValues();
				let optimzied_image;

				for (const property in children_results) {
					if (property.includes('image:optimize')) {
						optimzied_image = children_results[property];
						break;
					}
				}

				const uploaded_image = await this.uploadingImageToS3(optimzied_image);
				const uploaded = await this.flash_cards_service.update(job.data.id, {
					image: `image_url_from_s3_${uploaded_image.originalname}`,
				});
				return uploaded;
			default:
				throw new Error('No job name match');
		}
	}

	async uploadingImageToS3(
		image: Express.Multer.File,
	): Promise<Express.Multer.File> {
		this.logger.log('Start uploading image to S3...');
		return await new Promise((resolve) =>
			setTimeout(() => resolve(image), 5000),
		);
	}
}
```
Giải thích: để có thể lấy được kết quả từ các **children job** chúng ta có thể sử dụng method `getChildrenValues`, tuy nhiên vì kết quả có dạng object value nên khi truy xuất giá trị mình search theo queue name.

Tương tự như trên chúng ta thêm vào `FlashCardsModule`
```typescript:src/modules/flash-cards/flash-cards.module.ts
import { ImageOptimizationProcessor } from './queues/image-optimization.processor';
...
@Module({
	imports: [
        ...
       BullModule.registerQueue({
			name: 'image:upload',
			prefix: 'flash-cards',
		}),
    ],
    ...
	providers: [
		...
        ImageOptimizationProcessor
	],
...
```

Ngay sau khi thêm vào và save lại thì job được lấy xử lý, các bạn có thể kiểm tra log ở console hoặc ở redis-commander. 

![image.png](https://images.viblo.asia/6b5ece2a-81cd-47da-84f3-388367222d4a.png)

Vậy là chúng ta đã xong quá trình xây dụng một quy trình xử lý các job có quan hệ với nhau bằng *flows*. Có một vài lưu ý các bạn nên để tâm khi xóa job:
* Cách xóa job:
```typescript
await job.remove();
// or
await queue.remove(job.id);
```
* Khi parent job bị remove thì các children cũng sẽ bị remove
* Nếu một children job bị remove thì parent sẽ bỏ qua nó, và nếu nó là job cuối cùng thì parent job sẽ chuyển sang trạng thái tiếp theo.
* Nếu job bất kỳ sẽ bị xóa bị khóa, thì sẽ không bị xóa và throw ra exception.

### 4.9 Pause ⏸️ và Resume ▶️

Trong một vài trường hợp đặc biệt, ví dụ như chúng ta sử dụng service notification của bên thứ 3, nhưng hôm đó bất ngờ service đó có vấn đề hoặc đang bảo trì nâng cấp dẫn đến không thể thực hiện ngay lập tức. Khi đó nếu job đó vẫn chạy sẽ dẫn đến failed hàng loạt, vì thế chúng ta muốn tạm dừng xử lý queue đó để tránh các monitor cảnh báo liên tục.

> **Lưu ý:** job khi đã được **active** thì sẽ **không có cách nào** để **pause** hoặc **remove**.

Ví dụ mình tạo API để tạm dừng queue optimize:image
```typescript:src/modules/flash-cards/flash-cards.service.ts
import { InjectFlowProducer, InjectQueue } from '@nestjs/bullmq';
import { FlowProducer, Queue } from 'bullmq';
...
@Injectable()
export class FlashCardsService extends BaseServiceAbstract<FlashCard> {
	constructor(
		...
		@InjectQueue('image:optimize')
		private readonly image_optimize_queue: Queue,
	) {
		super(flash_cards_repository);
	}
	async pauseOrResumeQueue(state: string) {
		if (state !== 'RESUME') {
			return await this.image_optimize_queue.pause();
		}
		return await this.image_optimize_queue.resume();
	}
    ...
}
```
```typescript:src/modules/flash-cards/flash-cards.controller.ts
...
export class FlashCardsController {
	...
	@Patch('queue/state')
	@ApiQuery({
		name: 'state',
		enum: ['PAUSE', 'RESUME'],
	})
	pauseOrResumeQueue(@Query('state') state: string) {
		return this.flash_cards_service.pauseOrResumeQueue(state);
	}
}
```

Thử gọi API trên với state là PAUSE, sau đó gọi API tạo flash-card để kiểm tra kết quả:

![image.png](https://images.viblo.asia/87a68d5b-6d7d-4dce-a188-32a2932d8f03.png)

Có thể thấy được queue optimize:image xuất hiện key *pause*, queue image:check-valid thì vẫn hoạt động bình thường và do optimize:image đã pause nên parent job vẫn sẽ tiếp tục chờ ở waiting-children. Gọi lại API với state RESUME và xem kết quả:

![image.png](https://images.viblo.asia/94dbd97d-d0da-4553-9959-c74e7a77c046.png)

Khi queue được resume thì ngay lập tức job bên trong sẽ được lấy ra và trở lại flow xử lý như bình thường. 

> Khi pause queue các bạn nhớ cân nhắc xem có cần rate limiting không, vì khi resume thì sẽ có hàng loạt job được chạy có thể dẫn đến vấn đề như mình đề cập ở phần trên
# Kết luận 📝
Vậy là chúng ta đã tìm hiểu xong về cách triển khai một background job với BullMQ từ các tính năng cơ bản đến các tính năng nâng cao. Chúng ta cũng biết được một vài sự khác nhau giữa BullMQ và tiền thân của nó là Bull. Hy vọng bài viết có thể giúp ích cho các bạn trong tương lai. Cảm ơn các bạn đã giành thời gian đọc bài viết và hẹn gặp lại ở các bài tiếp theo.

# Tài liệu tham khảo 🔍
* What is bullmq (no date) BullMQ. Available at: https://docs.bullmq.io/ (Accessed: 12 August 2023). 
* Astudilllo, M. (2022) Dividing heavy jobs using BullMQ flows in nodejs, Taskforce.sh Blog. Available at: https://blog.taskforce.sh/splitting-heavy-jobs-using-bullmq-flows/ (Accessed: 12 August 2023). 
* Documentation: Nestjs - a progressive node.js framework (no date) NestJS. Available at: https://docs.nestjs.com/techniques/queues (Accessed: 12 August 2023). 

# Change log 📓
* August 12, 2023: Init document