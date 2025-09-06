Đây là bài viết nằm trong Series **NestJS thực chiến**, các bạn có thể xem toàn bộ bài viết ở link: https://viblo.asia/s/nestjs-thuc-chien-MkNLr3kaVgA

---

# Đặt vấn đề 📜
Dạo gần đây một vài project cũ của mình sử dụng package `aws-sdk` từ AWS đang gặp một warning ⚠️ (02/08/2023):

![image.png](https://images.viblo.asia/5e3103e2-5c1f-4797-9b76-764bf6e8aa27.png)

Nguyên nhân như các bạn cũng đã thấy, *AWS SDK for Javascript version 2* đã không còn được phát triển nữa thay vào đó là *version 3*. Thay vì gôm lại vào chung 1 package thì AWS đã tách chúng ta thành các package nhỏ giúp chúng ta tiện lợi và gọn gàng hơn trong việc sử dụng.
> Khá là khó chịu khi không dưới 1 lần mình gõ ConfigService thì auto import tự động import từ `aws-sdk` thay vì `@nestjs/config` 😩

Vì thế hôm nay chúng ta sẽ cùng nhau *upload public file* lên AWS S3 sử dụng *AWS SDK for Javascript version 3* với package mang tên `@aws-sdk/client-s3`. Bên cạnh đó chúng ta cùng tìm hiểu về khái niệm và cách triển khai package `@aws-sdk/lib-storage` dùng **tối ưu** cho trường hợp **upload các file có dung lượng lớn**.
# Thông tin package 📦️

* @aws-sdk/client-s3: ^3.379.1
* @aws-sdk/lib-storage: ^3.379.1

Toàn bộ source code của phần này các bạn có thể tải về ở branch [part-11-upload-file-aws-s3-client](https://github.com/nntwelve/Boilerplate-NestJS/tree/part-11-upload-file-aws-s3-client).

# Lý thuyết  📚
*TL,DR*: **AWS S3 là nơi chúng ta gửi file lên đó lưu trữ và dùng cho các mục đích khác nhau**.

Dành cho bạn nào mới làm quen với upload file trên AWS S3, **mình sẽ nói qua về các khái niệm, bạn nào đã nắm được rồi thì có thể skip đến phần triển khai**. 

## AWS S3 là gì?
Amazon **Simple Storage Service** (S3) là một dịch vụ lưu trữ đám mây được cung cấp bởi Amazon Web Services (AWS). AWS S3 cho phép chúng ta **lưu trữ** và **truy xuất** các file và các thông tin khác thông qua Internet. Đây là một trong những dịch vụ lưu trữ đám mây phổ biến và mạnh mẽ nhất, được sử dụng rộng rãi bởi các tổ chức và doanh nghiệp trên khắp thế giới.

Các đối tượng được lưu trữ trong AWS S3 được gọi là "**buckets**" (thùng). Mỗi bucket có thể chứa không giới hạn số lượng các đối tượng và **được định danh bằng một tên duy nhất trong không gian tên toàn cầu** của AWS - có nghĩa là **tên bucket là duy nhất trong toàn bộ hệ thống AWS**.

Một vài đặc điểm nổi bậc của AWS S3 có thể kể đến như:

* **Reliability**: AWS S3 đảm bảo độ tin cậy cao và có khả năng lưu trữ dữ liệu theo nguyên tắc đồng bộ hóa đệ quy. Dữ liệu được lưu trữ trên nhiều máy chủ và vị trí vật lý khác nhau để đảm bảo tính toàn vẹn và khả năng truy xuất.

* **Scalability**: AWS S3 cho phép mở rộng lưu trữ một cách linh hoạt, giúp đáp ứng nhu cầu của doanh nghiệp trong mọi tình huống, từ những ứng dụng nhỏ cho đến các ứng dụng lớn với khối lượng dữ liệu lớn.
* **Security**: S3 cung cấp một số tính năng bảo mật bao gồm *bucket policies*, *access control lists* (ACLs) và integration với AWS *Identity and Access Management* (IAM) để kiểm soát truy cập chi tiết đến các object của chúng ta.
* **Versioning**: S3 hỗ trợ versioning, cho phép chúng ta giữ nhiều version của một object trong cùng một bucket, providing additional protection against accidental data loss or overwrite. Cung cấp thêm lớp bảo vệ có khả năng chống mất mát hoặc ghi đè dữ liệu ngẫu nhiên.
* **Static Website Hosting**: Chúng ta có thể dùng S3 để static websites, giúp đơn giản hóa quá trình hosting các static content như HTML, CSS, và JavaScript files và vì thế giúp chúng ta tiết kiệm được chi phí.
* **Event Notifications**: S3 có thể trigger events dựa vào các action cụ thể như tạo hoặc xóa, có thể dùng để tự động hóa quy trình làm việc và tích hợp với các AWS services khác.

## `@aws-sdk/client-s3` là gì?

`@aws-sdk/client-s3` là một phần của **AWS SDK for JavaScript** (hay còn gọi là AWS SDK for Node.js) và là một thư viện JavaScript dùng để **tương tác** với dịch vụ **Amazon Simple Storage Service (S3)**. AWS SDK for JavaScript cho phép chúng ta sử dụng JavaScript để tạo, đọc, cập nhật và xóa các đối tượng và file trong Amazon S3.

Package này sử dụng AWS SDK for JavaScript **version 3**, được tách ra khỏi package `aws-sdk` AWS SDK for JavaScript **version 2**, chúng ta đã từng dùng theo dạng:
```shell:typescript:
// Old 👴: version 2️⃣
import { S3 } from 'aws-sdk';

// New 🧒: version 3️⃣
import { S3Client } from '@aws-sdk/client-s3';
```

# Triển khai ⚙️
Vì là một phần của hệ sinh thái AWS nên việc đầu tiên chúng ta cần làm là tạo IAM User trên AWS. 
## Tạo IAM User 👱
Để có thể kết nối vào AWS S3 chúng ta cần tạo IAM User để lấy **Access Key** và **Secret Access Key**. Các bạn truy cập vào [Identity And Access Management (IAM)](https://console.aws.amazon.com/iamv2/home#/users) sau đó chọn **Add users**. 

![image.png](https://images.viblo.asia/d667118f-68e8-4f29-acce-fadb78a3d515.png)

Điền vào bất kỳ user name nào các bạn muốn và bấm chọn **Next**

![image.png](https://images.viblo.asia/d788918c-ae3f-441c-a175-6b3695771026.png)

Tiếp theo chúng ta tích vào **Attach policies directly** để gán quyền trực tiếp (AWS recommend chúng ta tạo group sau đó add user vào, bạn nào muốn làm kĩ thì nên cân nhắc, còn mình làm ngắn gọn nên sẽ attach trực tiếp). Ở mục **Permissions policies**, chúng ta sẽ search như hình và tích chọn vào kết quả để grant full quyền của S3 cho user được tạo.  Cuối cùng là bấm **Next** và **Create user**.

Sau khi đã tạo xong, các bạn bấm vào user *s3-management* vừa tạo và bấm qua tab **Security credentials**, kéo xuống **Access keys** section và chọn **Create access key**

![image.png](https://images.viblo.asia/2e768494-feb0-4786-9715-32f146392872.png)

Giao diện tạo key sẽ xuất hiện, các bạn chọn **Other** sau đó bấm **Next**

![image.png](https://images.viblo.asia/1d9e6063-2b0c-48b9-9d6f-7cbfe7ba83cf.png)

Bước **Set description** là optional, các bạn muốn fill hay không cũng được. Sau đó bấm **Create access key** để hoàn thành.

![image.png](https://images.viblo.asia/f8ddcbe5-9157-442b-9c94-50b1e53f09c2.png)

Hình trên là chúng ta đã tạo thành công, các bạn cập nhật 2 giá trị trên vào file **.env**

```shell:.env
AWS_S3_PUBLIC_BUCKET=flash-card-project-public-bucket // Tên bucket lát nữa chúng ta dùng lưu trữ public file
AWS_S3_REGION=ap-southeast-1
AWS_S3_ACCESS_KEY_ID=YOUR_ACCESS_KEY // Đổi thành access key của các bạn
AWS_S3_SECRET_ACCESS_KEY=YOUR_SECRET_ACCESS // Đổi thành secret access key của các bạn
```

Vậy là chúng ta đã cấu hình xong user dùng để upload lên AWS S3, tiếp theo chúng ta sẽ **tiến hành tạo bucket làm nơi lưu trữ các file** chúng ta upload.

## Tạo bucket 🪣
Bucket các bạn có thể hiểu nó theo nghĩa đen là 1 cái xô chứa các file chúng ta gửi lên và chúng ta sẽ có các xô khác nhau để lưu chữ tùy theo mục đích khác nhau. Chúng được chia ra làm 2 loại:
- **Public Bucket**: là các file sau khi upload có thể truy cập trực tiếp bằng url. Thường được dùng cho các file không cần độ bảo mật cao.
- **Private Bucket**: là các file chỉ có thể truy cập bằng cách gọi lên AWS để tạo url, và url sẽ chỉ tồn tại trong một khoảng thời gian nhất định, sau đó nếu muốn truy cập sẽ lặp lại bước gọi lên AWS. Private bucket được dùng cho các file mang tính cá nhân và bảo mật cao.
### Tạo Public Bucket 🌐🪣
Tiến hành tạo **bucket** thông qua các bước sau:

* Truy cập [AWS S3 Bucket](https://console.aws.amazon.com/s3/buckets) và chọn **Create bucket**. Lưu ý tên bucket phải trùng với file `.env` ở trên là `flash-card-project-public-bucket`.

![](https://images.viblo.asia/10c31ee4-d927-40c7-bf7e-a5297c82105a.png)

 * Ở **Access Control List (ACL)** mình sẽ cho *enable* để có thể control ở trong code:

![](https://images.viblo.asia/0bd9ec84-1931-481d-988e-c59e95ad8e77.png)

* Chúng ta sẽ lưu trữ các file như *avatar*, *flash card illustration image* theo dạng public nên sẽ bỏ chọn **Block all public access** và chọn ở warning kèm theo:

![image.png](https://images.viblo.asia/2bdd6d92-3aad-40f2-a8e5-95cfd9b5ed65.png)

* Các bước còn lại chúng ta sẽ để mặc định và chọn **Create bucket**. Khi tạo thành công sẽ được như bên dưới.

![image.png](https://images.viblo.asia/939861c2-5031-4304-9a0a-78f08fffeea4.png)

### Tạo Private Bucket 🔐🪣
Phần này chúng ta sẽ cùng tìm hiểu ở bài viết tiếp theo.
## Cài đặt package 🛠
Việc tiếp theo chúng ta cần làm là cài đặt `@aws-sdk/client-s3`:

`npm install @aws-sdk/client-s3`

## Viết logic upload file 🆙📄
Chúng ta sẽ tiến hành viết service dùng để upload file, nhưng trước tiên sẽ áp dụng quy tắc **D** (Dependency Inversion) trong **S.O.L.I.D** tạo abstract cho service này trước khi bắt tay vào implement logic.

```typescript:src/services/files/upload-file.abstract.service.ts
export abstract class UploadFileServiceAbstract {
	abstract uploadFileToPublicBucket(
		path: string,
		{ file, file_name }: { file: Express.Multer.File; file_name: string },
	): Promise<{ url: string; key: string }>;
}
```

Khi đã có abtract chúng ta mới tiến hành implement logic như sau:

```typescript:src/services/files/upload-file.service.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { UploadFileServiceAbstract } from './upload-file.abstract.service';

@Injectable()
export class UploadFileServiceS3 implements UploadFileServiceAbstract {
	private s3_client: S3Client;
	constructor(private readonly config_service: ConfigService) {
		this.s3_client = new S3Client({
			region: config_service.get('AWS_S3_REGION'),
			credentials: {
				accessKeyId: config_service.get('AWS_S3_ACCESS_KEY_ID'),
				secretAccessKey: config_service.get('AWS_S3_SECRET_ACCESS_KEY'),
			},
		});
	}
	async uploadFileToPublicBucket(
		path: string,
		{ file, file_name }: { file: Express.Multer.File; file_name: string },
	) {
		const bucket_name = config_service.get('AWS_S3_PUBLIC_BUCKET');
		const key = `${path}/${Date.now().toString()}-${file_name}`;
		await this.s3_client.send(
			new PutObjectCommand({
				Bucket: bucket_name,
				Key: key,
				Body: file.buffer,
                ContentType: file.mimetype,
				ACL: 'public-read',
				ContentLength: file.size, // calculate length of buffer
			}),
		);

		return `https://${bucket_name}.s3.amazonaws.com/${key}`;
	}
}
```
Giải thích:
* Nếu ở version 2 chúng ta phải config AWS ở bên ngoài file **main.ts** thì ở version 3 chúng ta sẽ config thẳng ở trong service dùng nó. 
* Biến `key` dùng để tạo path cho file trên S3, được cấu thành từ:
    * `path`: folder lưu trữ file. Chúng ta sẽ có các folder như: profile-picture, flash-card, collection, topic... để tiện cho việc quản lý. 
    * `Date.now` kết hợp với `file_name` dùng để lưu trữ tên file cùng với timestamp.
* Method `upload` ở version 2 sẽ được thay đổi thành `send` kết hợp với `PutObjectCommand` và không cần phải chain `.promise()` ở đằng sau nữa. Các tham số truyền vào như sau:
    * Bucket: tên bucket vừa tạo ở bước trên.
    * Key: tên file.
    * Body: file cần upload. Có thể là *string, Buffer, Uint8Array, Readable, ReadableStream, Blob*
    * ContentType: loại file như png, jpeg, mp4,...
    * Bổ sung `ACL: 'public-read'` biểu thị rằng file này được truy cập public. Nếu sau này chúng ta cần thay đổi ACL không cho public nữa có thể chỉnh ở đây hoặc trên giao diện S3.
    * Bổ sung `ContentLength: file.size` là kích thước của file truyền vào `Body`, giúp ích cho trường hợp không thể tự động xác định kích thước.
* Response sau khi upload sẽ không bao gồm url của file nên chúng ta cần phải thủ công tạo ra nó. Rất may là url của file được lưu trên S3 có pattern là `https://${bucket_name}.s3.amazonaws.com/${key}` nên chúng ta chỉ cần truyền vào các giá trị là có thể truy cập vào file.
* Trả về `key` để sau này dùng tương tác với file đã lưu trên S3, ví dụ như xóa. 

> Bạn nào gặp lỗi như hình dưới đa phần là do thiếu `ContentLength`
> 
> ![image.png](https://images.viblo.asia/25bf79dc-457c-4a76-938d-63450f980eea.png)

Sau khi đã tạo xong service chúng ta sẽ inject vào `FlashCardsModule` để tiến hành tích hợp vào job `uploading-image`.

```typescript:
import { UploadFileServiceAbstract } from 'src/services/files/upload-file.abstract.service';
import { UploadFileServiceS3 } from 'src/services/files/upload-file-s3.service';
...
@Module({
	...
	providers: [
		...
		{
			provide: UploadFileServiceAbstract,
			useClass: UploadFileServiceS3,
		},
	],
})
export class FlashCardsModule {}
```

> Có thể thấy được nhờ áp dụng **Dependency Inversion** chúng ta không bị hard code vào provider **AWS S3** mà có thể dễ dàng thay thế `UploadFileServiceS3` thành bất kỳ provider nào khác như **Google Clound**, **Cloudinary**,...

Tiếp theo là apply vào `ImageUploadingProcessor`, bạn nào chưa rõ về `ImageUploadingProcessor` có thể xem lại bài viết [Phần 8: Xử lý Background job với BullMQ](https://viblo.asia/p/setup-boilerplate-cho-du-an-nestjs-phan-8-xu-ly-background-job-voi-bullmq-yZjJY9DDJOE#_47-sandboxed-processors-12):

```typescript:src/modules/flash-cards/queues/image-uploading.processor.ts
import { UploadFileServiceAbstract } from 'src/services/files/upload-file.abstract.service';
...
@Processor('image:upload', {...})
export class ImageUploadingProcessor extends WorkerHost {
	constructor(
		private readonly flash_cards_service: FlashCardsService,
		private readonly upload_file_service: UploadFileServiceAbstract,
	) {	super() }
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

				this.logger.log('Start uploading image to S3...');
                // 📝 chỉnh sửa lại ở đây
				const uploaded_result =
					await this.upload_file_service.uploadFileToPublicBucket(
						'profile-picture',
						{ file: optimzied_image, file_name: job.data.id },
					);
				const uploaded = await this.flash_cards_service.update(job.data.id, {
					image: uploaded_result,
				});
				return uploaded;
			default:
				throw new Error('No job name match');
		}
	}
}
```
Giải thích:
* Chúng ta chỉ cần inject vào và sử dụng bình thường như các module khác.
* `profile-picture` sẽ là tên folder chứa file của chúng ta (giá trị này tốt nhất nên truyền từ flash card service chứ không phải ở processor, do mình làm ngắn gọn nên để ở đây).
* `file_name` mình sẽ chọn là id để dễ truy xuất sau này.

Ở phần `await this.flash_cards_service.update(job.data.id, { image: uploaded_result })` chúng ta sẽ lưu toàn bộ `url` và `key` nên cần thay đổi lại field `image` trong `FlashCard` model nếu không sẽ gặp lỗi. Chỉnh sửa lại như sau

```typescript:src/modules/flash-cards/entities/flash-card.entity.ts
import { PublicFile } from '@modules/shared/upload-files/public-files.entity';
...
export class FlashCard extends BaseEntity {
	...
	@Prop({
		type: PublicFile,
	})
	@Type(() => PublicFile)
	image: PublicFile;
    ...
```

```typescript:src/modules/shared/upload-files/public-files.entity.ts
import { Prop } from '@nestjs/mongoose';

export class PublicFile {
	@Prop({ required: true })
	url: string;

	@Prop({ required: true })
	key: string;
}
```

Chỉnh luôn cả ở `CreateFlashCardDto`

```typescript:src/modules/flash-cards/dto/create-flash-card.dto.ts
import { PublicFile } from '@modules/shared/upload-files/public-files.entity';

export class CreateFlashCardDto {
    ...
	image: PublicFile;
    ...
}
```

Mọi thứ đã hoàn tất, giờ chúng ta chỉ cần gọi API tạo flash card để kiểm tra kết quả:

![image.png](https://images.viblo.asia/005beb3e-fc90-4d83-b064-29749e637729.png)

Folder `profile-picture` đã tự động được tạo ra, thử mở ra xem bên trong xem đã lưu được file hay chưa:

![image.png](https://images.viblo.asia/a064c614-b806-4f32-94bc-ab00cc35665b.png)

> Có một điều khá là thú vị ở AWS S3, đó là mặc dù giao diện chia theo dạng folder nhưng thực chất tất cả các file dù nằm trong folder nào cũng là cùng một cấp thư mục. Bằng cách sử dụng ký tự `/`, AWS sắp xếp nó lại và hiển thị cho người dùng dưới dạng folder.

Có thể thấy file chúng ta vừa lưu đã được lưu trữ lên S3, các bạn có thể mở bấm vào file đó và chọn link
bên dưới Object Url để xem hình vừa up có bị lỗi không:

![image.png](https://images.viblo.asia/cd00964a-f10a-4329-af63-113dcc38ba89.png)

Sau khi đã chắc chắn file lưu thành công, chúng ta có thể đối chiếu lại ở trong database xem link có trùng khớp hay không:

![image.png](https://images.viblo.asia/1cd19ea4-f8b8-4c9b-ad39-5bc04e0d437d.png)

Vậy là chúng ta đã lưu thành công file lên AWS S3 sử dụng AWS SDK version 3. Việc tiếp theo là triển khai logic delete image.

## Viết logic delete file 📄🪓
Việc xóa file khỏi S3 khá là đơn giản, chúng ta chỉ cần dùng method `DeleteObjectCommand`. Nhưng trước tiên chúng ta cần cập nhật `UploadFileServiceAbstract`

```typescript:src/services/files/upload-file.abstract.service.ts
export abstract class UploadFileServiceAbstract {
	...
	abstract deleteFileFromPublicBucket(key: string): Promise<void>;
}
```

Sau đó implement logic, việc xử lý rất đơn giản, chúng ta chỉ cần truyền vào `key` được trả về từ bước upload file khi nảy:

```typescript:src/services/files/upload-file-s3.service.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
...
export class UploadFileServiceS3 implements UploadFileServiceAbstract {
	...
	async deleteFileFromPublicBucket(key: string): Promise<void> {
		await this.s3_client.send(
			new DeleteObjectCommand({
				Bucket: this.config_service.get('AWS_S3_PUBLIC_BUCKET'),
				Key: key,
			}),
		);
	}
}
```

## Cải thiện tốc độ upload với Multipart Upload 📈
Trong quá trình upload file, nếu **file có dung lượng** nhỏ thì không sao, nếu như có kích thước **lớn thì sẽ dẫn đến mất khá nhiều thời gian**. Chúng ta cần tìm cách nào đó để tăng tốc quá trình này, giúp cho trải nghiệm của người dùng được mượt mà hơn. Rất may là AWS có cung cấp cho chúng ta một tính năng giúp tăng tốc quá trình đó, nó được gọi là [Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html). 

>"Multipart upload allows you to upload a single object as a set of parts. Each part is a contiguous portion of the object's data. You can upload these object parts independently and in any order."

Nói ngắn gọn là thay vì gửi cả file lên thì nó sẽ được chia ra thành nhiều phần và gửi lên đồng loạt mà không phụ thuộc lẫn nhau. Khi nào tất cả các phần được gửi lên đầy đủ thì AWS S3 sẽ tổng hợp lại thành một file hoàn chỉnh.

Để triển khai Multiple Upload có 2 cách:
* Cách 1 dùng method [CreateMultipartUploadCommand](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/s3/command/CreateMultipartUploadCommand/) của package @aws-sdk/client-s3.
* Cách 2 đơn giản hơn là dùng library `@aws-sdk/lib-storage`. 

Cách 2 thì ngắn gọn và dễ dùng hơn nên chúng ta sẽ dùng trong bài viết này. Bạn nào thích sự thách thức có thể dùng cách 1 😁  

Chúng ta sẽ bắt đầu vận dụng sức mạnh của `@aws-sdk/lib-storage` bằng cách cài đặt nó:

`npm install @aws-sdk/lib-storage`

Cuối cùng là thay thế method `PutObjectCommand` bằng method `Upload` và thêm thắt một vài logic như sau:

```typescript:src/services/files/upload-file-s3.service.ts
import { Upload } from '@aws-sdk/lib-storage';

export class UploadFileServiceS3 implements UploadFileServiceAbstract {
	private s3_client: S3Client;
	constructor(private readonly config_service: ConfigService) {
		console.log(config_service.get('AWS_S3_REGION'));
		this.s3_client = new S3Client({
			region: config_service.get('AWS_S3_REGION'),
			credentials: {
				accessKeyId: config_service.get('AWS_S3_ACCESS_KEY_ID'),
				secretAccessKey: config_service.get('AWS_S3_SECRET_ACCESS_KEY'),
			},
		});
	}
	async uploadFileToPublicBucket(
		path: string,
		{ file, file_name }: { file: Express.Multer.File; file_name: string },
	) {
		const bucket_name = this.config_service.get('AWS_S3_PUBLIC_BUCKET');
		const key = `${path}/${Date.now().toString()}-${file_name}`;

		const parallelUploads3 = new Upload({
			client: this.s3_client,
			params: {
				Bucket: bucket_name,
				Key: key,
				Body: Buffer.from(file.buffer),
				ACL: 'public-read',
				ContentType: file.mimetype,
			},
			queueSize: 4, // optional concurrency configuration
			partSize: 1024 * 1024 * 5, // optional size of each part, in bytes, at least 5MB
			leavePartsOnError: false, // optional manually handle dropped parts
		});

		parallelUploads3.on('httpUploadProgress', (progress) => {
			console.log({ progress });
		});

		await parallelUploads3.done();

		return {
			url: `https://${bucket_name}.s3.amazonaws.com/${key}`,
			key,
		};
	}
    ...
}
```
Giải thích:
* Các option trong `params` sẽ tương tự như ở `PutObjectCommand` và không cần có `ContentLength`
* `queueSize`: số lượng tối đa các chunks được tải lên đồng thời. Ví dụ ở trên là 4 chunks cùng một lúc. 
* `partSize`: kích thước tối đa của mỗi chunk. Mặc định là 5MB như chúng ta truyền vào ở trên.
* `leavePartsOnError`: quá trình tải lên sẽ loại bỏ các chunk không thành công. Có thể set thành true nếu muốn xử lý lỗi thủ công. Khi xem console các bạn sẽ thấy quá trình upload diễn ra.

Chỉ cần cập nhật vậy thôi là chúng ta đã có thể sử dụng. Mình sẽ tiến hành test với file có dung lượng **43MB** với API khi nảy.

![image.png](https://images.viblo.asia/bbb3b32b-7e4c-4fe8-bbda-f2fd74b5b008.png)

Để kiểm tra xem việc sử dụng `@aws-sdk/lib-storage` có nhanh hơn hay không, mình đã thử nghiệm upload với file có dung lượng 43MB bằng cả 2 cách và cho ra kết quả như bên dưới.

| Số lần thử | Lần 1 | Lần 2 | Lần 3 | Lần 4 | Lần 5 |
| -------- | -------- | -------- | -------- | -------- |-------- |
| method `PutObjectCommand`   | 10.3     | 14.8     | 12.8    | 4.0     | 16.7     |
| method `Upload`     | 4.0     | 4.3     | 13.1    | 7.5     | 5.87     |

Có thể thấy từ table trên, tốc độ được cải thiện đáng kể như thế nào. (Ở một vài lần thử thì tốc độ đột nhiên nhanh hoặc chậm đi chắc là do mạng ở nhà mình có nhiều người dùng nên gặp vấn đề chỗ đó, còn lại nhìn chung là ổn).

## Multipart Upload con dao 2 lưỡi 🗡️

> "There's no rose without a thorn"

Đi kèm với những lợi ích không thể chối cãi trên, nếu như để ý các bạn sẽ thấy một vấn đề:

**Điều gì sẽ xảy ra nếu quá trình upload multipart gặp error hoặc chúng ta cancel, những part đã được upload thành công sẽ đi về đâu?**

Câu trả lời rất đơn giản: **chúng vẫn sẽ được lưu trữ trong S3** và như tất cả các file khác, chúng ta sẽ phải trả tiền cho số dung lượng để lưu trữ chúng. Khá là tệ nếu như khách hàng giao account cho chúng ta và đến tháng họ hỏi tại sao file lưu thì ít mà số tiền charge thì nhiều 🫣. Do các phần đó chưa hoàn chỉnh nên chúng ta sẽ không thể xem được chúng trên giao diện S3 thông thường mà phải dùng tới **AWS Storage Lens**.

Để khắc phục vấn đề trên chúng ta đi đến một khái niệm mới đó là [Amazon S3 Lifecycle 🌀](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)

>"An S3 Lifecycle configuration is a set of rules that define actions that Amazon S3 applies to a group of objects"

Một điều mình lưu ý trước là phạm vi sử dụng của **S3 Lifecycle** sẽ rộng hơn so vấn đề chúng ta vừa nêu nên và ở bài viết này mình chỉ dùng một phần công lực của nó để xử lý, các bạn có thể tự mình đọc docs để khám phá thêm các chức năng còn lại.

Ở giao diện S3 chọn tab [Management](https://console.aws.amazon.com/s3/buckets/flash-card-project-public-bucket?region=ap-southeast-1&tab=management):

![image.png](https://images.viblo.asia/34364dea-99e4-4b6e-850b-314176c80ae5.png)

Sau đó bấm vào **Create lifecycle rule**, *Rule name* thì các bạn đặt gì cũng được, gợi nhớ là được. *Rule scope* thì chúng ta sẽ *Apply to all...* để áp dụng cho toàn bộ object trong bucket - Nhớ tick chọn *I acknowledge that...*, hoặc nếu các bạn biết chắc các object nào dùng multipart upload thì có thể chọn *Limit the scope...* để dùng prefix filter.

![image.png](https://images.viblo.asia/5492bdb9-2cc9-4dc4-a12c-fe6fc5b245eb.png)

Cuối cùng là chọn *Delete expired object delete markers or incomplete multipart uploads* và tick vào ô *Delete imcomplete multipart uploads*, ở đây mình sẽ setup sau 7 ngày sẽ tự động xóa các part upload không thành công.

![image.png](https://images.viblo.asia/18532e6e-1019-4879-9216-b58b73ba06a1.png)

Bấm vào **Create rule** là xong, chúng ta sẽ được kết quả như bên dưới:

![image.png](https://images.viblo.asia/ee453a64-0f76-40e6-a425-d2c76fe0659f.png)

Vậy là chúng ta đã xử lý xong vấn đề các tài nguyên lưu trữ của các part upload không thành công trên S3, từ giờ các incomplete part sẽ bị xóa sau 7 ngày để tránh chiếm dụng tài nguyên vô ích. 

Còn một phần cũng khá là hay nữa đó là về phân tích và thống kê các file được lưu trên S3, khi nảy mình có nhắc đến là dùng **AWS S3 Storage Lens Dashboard**. Mình chưa sử dụng nhiều về phần này nên sẽ hẹn các bạn vào một bài viết khác khi mình đã gôm đủ kinh nghiệm.

# Kết luận 📝

Ở bài viết này chúng ta đã cùng nhau nâng cấp AWS SDK từ version 2 lên version 3 thành công. Đồng thời cũng tìm hiểu sơ qua về cách upload và delete file trong AWS S3. 

Bên cạnh đó chúng ta cũng tìm hiểu được cách tối ưu quá trình upload file có dung lượng lớn với Multipart Upload.

Cảm ơn các bạn đã giành thời gian đọc bài viết, nếu có bất kì thắc mắc nào các bạn có thể comment ở dưới để mọi người cùng thảo luận nhé. Hẹn gặp lại các bạn ở các bài viết tiếp theo.
# Tài liệu tham khảo 🔍
- (No date) @aws-SDK/client-S3. Available at: https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/ (Accessed: 05 August 2023). 
- Akinsiku, F. (2023) How to upload files to Amazon S3 with Node.js, freeCodeCamp.org. Available at: https://www.freecodecamp.org/news/how-to-upload-files-to-aws-s3-with-node/ (Accessed: 05 August 2023). 
- Wanago, M. (2020) API with nestjs #10. uploading public files to Amazon S3, Marcin Wanago Blog - JavaScript, both frontend and backend. Available at: https://wanago.io/2020/08/03/api-nestjs-uploading-public-files-to-amazon-s3/ (Accessed: 05 August 2023). 
# Change log 📓
- January 14, 2024: Init document