# Đặt vấn đề 📜
Chủ đề về Event Loop thì đã hiện hữu rất nhiều trên Internet, tuy nhiên khi đọc và tìm hiểu mình thấy đa phần vẫn chưa thật sự dễ hiểu và tiếp thu, ngay cả tài liệu trên trang chính thức của NodeJS cũng không ngoại lệ 😅. 

Đợt vừa rồi mình đọc được bài viết của tác giả **VISHWAS GOPINATH** trên trang builder.io thấy rất hay và dễ hiểu nên mình đã tham khảo và tổng hợp lại với kiến thức cá nhân để giúp mọi người có thể tiếp cận dễ dàng hơn. Link các bài viết của tác giả mình có để ở phần Tài liệu tham khảo 🔍, các bạn có thể tham khảo.

Mình cũng có làm một vài gif mô phỏng lại quá trình thực thi, hy vọng sẽ giúp mọi người dễ hình dung hơn 🌟. 
# Thông tin package 📦️
Các bạn có thể tải về các ví dụ [ở đây](https://github.com/nntwelve/nodejs-event-loop)
# Lý thuyết 📃

> User written synchronous Javascript code takes priority over async code that the runtime would like to execute. Only after the call stack is empty, the event loop comes into the picture.

Giải thích một cách dễ hiểu thì code mà hiện tại đang chạy nếu đang ở đoạn synchronous thì callback của các code asynchronous vừa thực thi sẽ không được gọi cho đến khi thực thi hết tất cả các code synchronous (hay callstack empty). 

![image.png](https://images.viblo.asia/edf53fca-ed05-4f25-9ed5-eb995647edfd.png)

Event loop trong NodeJS được chia ra làm 4 phrase chính:
- **Timer**: chứa các hàm liên quan tới thời gian như setTimeout, setInterval.
- **I/O Polling**: thực hiện các tác vụ liên quan tới I/O. 
- **Check**: gọi các callback trong setImmediate.
- **Close**: các các callback của các close handler.

Giữa các phrase chính đều sẽ gọi tới **microtask queue**, bên trong nó có thêm 2 phrase sau:
- **nextTick**: thực hiện các callback được khai báo trong *process.nextTick*
- **Promise**: gọi callback khi promise resolve.

> ***Timer*, *I/O*, *check*, *close*** **là thành phần của libuv**, ***microtask queue* không phải là thành phần của libuv**. Chúng ta sẽ cùng nhau tìm hiểu về **libuv** ở một bài  viết khác.

![image.png](https://images.viblo.asia/3b8e0e55-ea0e-4f87-b6f0-c1d4cbe1cb90.png)

Hình trên biểu diễn thứ tự thực thi của các phrase trong Event loop như sau:
- 1. Các callbacks trong *microtask queue* sẽ được thực thi. Callbacks trong *nextTick queue* sẽ được thực thi trước, sau đó đến callbacks trong *promise queue*.
- 2. Tất cả các callbacks trong setTimeout, setInterval sẽ được thực thi.
- 3. Tiếp tục quay trở lại thực thi các callbacks trong *microtask queue* nếu có. Thứ tự sẽ luôn là *nextTick queue* sau đó tới *promise queue*.
- 4. Thực thi tất cả callbacks trong I/O queue.
- 5. Thực thi toàn bộ các callbacks trong *microtask queue*.
- 6. Tất cả các callbacks trong *check queue* được thêm bởi setImmediate được thực thi.
- 7. Tiếp tục thực thi các callbacks trong *microtask queue*.
- 8. Các callbacks trong *close queue* được thực thi.
- 9. Tiếp tục thực thi các callbacks trong *microtask queue*.

Sau khi thực hiện xong các bước trên và vẫn còn các callbacks mới được thêm vào thì Event loop sẽ tiếp tục thực hiện thêm một loop nữa và các bước trên sẽ được lặp lại. Ngược lại nếu không còn đoạn code nào thực thi thì Event loop sẽ exit. 

## Microtask queue 🔬
### nextTick queue ✔️
Các callbacks của process.nextTick sẽ được thêm vào nextTick queue.
> Các callbacks trong nextTick queue sẽ **thực thi** cho **đến khi nào không còn callbacks nào trong nextTick queue**. 
 
Có nghĩa là nếu chúng ta **tiếp tục thêm callback vào** trong **nextTick queue** bằng process.nextTick thì **callback đó sẽ được thực thi** (theo thứ tự FIFO) trong phrase đó luôn chứ **không cần phải đợi thực thi các phrase sau** xong mới quay lại xử lý. Để chứng thực chúng ta cùng xét ví dụ sau:

```js:6-phrase/example-0-process-next-tick.js
1   process.nextTick(() => {
2     console.log('Next tick 1');
3     process.nextTick(() => console.log('Next tick 1.1'));
4   });
5
6   setTimeout(() => console.log('Set timeout 1'));
```
Chạy thử và kiểm tra kết quả bên dưới, các bạn thử đoán xem kết quả có giống như bên dưới không:

![image.png](https://images.viblo.asia/c556126b-d6b1-4632-bb0c-3ff45c11f270.png)

Chúng ta có thể thấy được mặc dù ở line 3  chúng ta chỉ mới thêm vào callback 'Next tick 1.1' nhưng nó lại thực thi trước callback setTimeout ở line 6. Để hiểu rõ hơn các bạn có thể xem minh họa quá trình thực thi như bên dưới:

![](https://images.viblo.asia/18f63ce2-917e-46ed-8086-b4c89b073b45.gif)


### Promise queue 🤝
Các *callback* khi **Promise resolve** hoặc được truyền vào **queueMicrotask** sẽ được thêm vào **Promise queue**.

Tương tự như **nextTick queue,** nếu chúng ta thêm **Promise** hoặc **queueMicrotask** thì nó sẽ thực thi ngay trong phrase của **microtask queue**.

Chúng ta sẽ cùng xem xét ví dụ đầy đủ các thành để chứng minh các điều trên:

```typescript:6-phrase/example-1-microtask-queue.js
1   process.nextTick(() => console.log('nextTick: 1'));
2   process.nextTick(() => {
3     console.log('nextTick: 2');
4     Promise.resolve().then(() => console.log('nextTick-Promise: 2.1'));
5     process.nextTick(() => console.log('nextTick-nextTick: 2.2'));
6   });
7   process.nextTick(() => console.log('nextTick: 3'));
8
9   Promise.resolve().then(() => console.log('Promise: 1'));
10  Promise.resolve().then(() => {
11    console.log('Promise: 2');
12    Promise.resolve().then(() => console.log('Promise-promise: 2.1'));
13    process.nextTick(() =>  console.log('Promise-nextTick: 2.2'));
14  });
15  Promise.resolve().then(() => console.log('Promise: 3'));
16
17  setTimeout(() => {
18    console.log('Timeout: 1');
19  }, 0);
20  console.log('Console 1');
```

Kết quả khi run sẽ như sau:
![image.png](https://images.viblo.asia/0f58310c-5c1c-42e2-b01c-26aea0c67027.png)

- Đầu tiên khi chúng ta khởi chạy, các callbacks **nextTick-1,2,3 (line 1, 2-6, và 7)** và **Promise-1,2,3 (line 9, 10,15)** được đưa vào **microtask queue** do là *asynchronous code (async)*. Sẽ **chưa có gì thực thi** vào thời điểm này. 
- Callback của **setTimeout (line 17)** cũng là *async* nên sẽ được đưa vào **Timer queue** do chúng ta set là 0. Lưu ý nó sẽ phải vào NodeAPI rồi sau đó mới đến **Timer queue**, ở phần sau về Timer queue chúng ta sẽ tìm hiểu rõ hơn. Vẫn **chưa có gì thực thi** vào thời điểm này. 
- Do `console.log` (line 20) là *synchronous code* nên sẽ **được thực thi** ngay. `Console 1` in ra màn hình.
- Vì không còn code để thực thi nên control sẽ chuyển đến **Event loop** và **bắt đầu thực thi** các *async code*. Quá trình diễn ra như sau:
    - **Kiểm tra callbacks trong nextTick queue**, ở đây chúng ta đang có 3 callbacks (**line 1, 2-6, 7**) nên sẽ lấy ra **thực thi lần lượt theo thứ tự FIFO**:
        - Callback line 1: `nextTick: 1` in ra màn hình
        - Callback line 2-6: `nextTick: 2` in ra màn hình. Callback của **Promise (line 4)** được thêm vào **Promise queue**. Callback trong **process.nextTick (line 5)** tiếp tục được thêm vào **nextTick queue**.
        - Callback line 7: `nextTick: 3` in ra màn hình. 
        - Lúc này do **nextTick queue vẫn chưa rỗng (callback của line 5 vừa thêm vào) nên sẽ tiếp tục lấy callback ra xử lý**. Kết quả **`nextTick: 2.2` (line 5)** được in ra màn hình. Đến đây thì đã **không còn callback trong nextTick queue nên sẽ chuyển đến Promise queue**.
    - **Kiểm tra callbacks trong Promise queue**, hiện tại đang có 4 callbacks bên trong (sắp xếp theo thứ tự được thêm vào từ **line 9,10-14,15 và 4**), tiến hành thực thi theo thứ tự FIFO tương tự như ở **nextTick queue**: 
        - **Callback line 9**: `Promise: 1` in ra màn hình.
        - **Callback line 10-14**`Promise: 2` (line 11) in ra màn hình. Callback của **Promise (line 12)** được thêm vào **Promise queue** (số lượng callbacks còn lại trong **Promise queue** hiện tại là 3: **line 15, 4 và 12**). Callback trong process.nextTick (line 13) được thêm vào nextTick queue. 
        - **Callback line 15**:`Promise: 3` in ra màn hình.
        - **Callback line 4**:`nextTick-Promise: 2.1` in ra màn hình.
        - **Callback line 12**:`Promise-promise: 2.1` in ra màn hình. 
     - Sau khi thực thi xong callback **line 12** thì **Promise queue đã rỗng, vì thế lại quay về kiểm tra nextTick queue và xử lý callbacks chứ không chuyển đến Timer queue**.
    - **Kiểm tra callbacks trong nextTick queue**, vẫn còn 1 callback vừa được thêm vào (**line 13**) nên nó sẽ được lấy ra thực thi. Kết quả `Promise-nextTick: 2.2` được in ra màn hình. Giờ thì Promise queue và nextTick queue đã rỗng, control được chuyển đến phrase tiếp theo, ở đây là **Timer queue**.
    - **Kiểm tra callbacks trong Timer queue**, lấy callback ở line 17 ra thực thi và in ra màn hình `Timeout: 1`.

> Từ trình tự thực thi trên chúng ta đã chúng minh được: **Nếu các callbacks mới liên tục được thêm vào nextTick và Promise queue thì nó sẽ mắc kẹt trong microtask queue và control không thể chuyển đến xử lý các phrase tiếp theo 😵**.

Cùng xem lại quá trình thực thi thông qua gif bên dưới, không hiểu sao mấy hôm nay Viblo không up hình gif lên được nên các bạn tham khảo link dưới giúp mình, khi nào upload lại được mình sẽ cập nhật lại:

https://github.com/nntwelve/nodejs-event-loop/blob/main/example-1.gif

Do đó không khuyến khích chúng ta sử dụng process.nextTick, thông thường chỉ dùng cho 2 lý do như bên dưới:
![image.png](https://images.viblo.asia/270dd27f-8bdd-4dee-a1ff-cf3e3596d78d.png)

## Timer queue 🕰️
Chúng ta cùng đến với queue đầu tiên trong libuv, các callbacks được khai báo trong setTimeout và setInterval sẽ được enqueue vào Timer queue. Tuy nhiên cần lưu ý một điều, callback của 2 function trên chỉ chuyển vào Timer queue khi và chỉ khi hết thời gian được khai báo. Cùng xét ví dụ sau:
```typescript:6-phrase/example-2-timer-queue.js
1  setTimeout(() => console.log('Timeout: 1'), 1000);
2  setTimeout(() => console.log('Timeout: 2'), 0);
3  setTimeout(() => console.log('Timeout: 3'), 0);
4
5  process.nextTick(() => console.log('nextTick: 1'));
6  Promise.resolve().then(() => console.log('Promise: 1'));
```
Kết quả thu được:

![image.png](https://images.viblo.asia/972c4fc2-d28b-4bfb-97a5-f256f91e0e0e.png)

Từ hình trên chúng ta có thể hình dung quy trình thực thi diễn ra như sau:
- Ở line 1 setTimeout có time là 1000 nên được đưa vào NodeAPI, lúc này Timer queue vẫn đang rỗng.
- Line 2 và 3 do có time là 0 nên khi đưa vào NodeAPI ngay lập tức trả về kết quả và đẩy vào Timer queue.  Lúc này Timer queue sẽ chứa 2 callbacks.
> Lưu ý, quá trình đưa từ NodeAPI về Timer queue có thể bị delay do CPU đang busy, kết quả làm cho callbacks bị đẩy qua loop tiếp theo. Khi đến phần I/O queue chúng ta sẽ thấy rõ hơn vấn đề này.
- Line 5, 6 tương tự như ví dụ trước của chúng ta, lần lượt được đưa vào nextTick queue và Promise queue.
- Sau khi đến the end of file, control sẽ đưa về cho Event loop.
    - `nextTick: 1`, `Promise: 1`  được in ra màn hình chứng minh cho chúng ta thấy được Timer queue có thứ tự thấp hơn Microtask queue.
    - `Timeout: 2`, `Timeout: 3` được in ra theo thứ tự FIFO. Đến đây thì Timer queue đã rỗng và khoảng 1 giây sau thì setTimeout ở line 1 hoàn thành, callback được gửi đến Timer queue.
    - `Timeout: 1` được in ra màn hình.

Ở ví dụ trên chúng ta đã thấy được priority của Timer queue so với Microtask queue. Tiếp theo chúng ta sẽ chỉnh sửa lại ví dụ đó một tí để nó trở nên thú vị hơn:
```typescript:6-phrase/example-2-timer-queue.js
1   setTimeout(() => console.log('Timeout: 1'), 1000);
2   setTimeout(() => {
3    console.log('Timeout: 2');
4    Promise.resolve().then(() => console.log('Timeout-promise:2.1'));
5    process.nextTick(() => console.log('Timeout-nextick: 2.2'));
6   }, 0);
7   setTimeout(() => console.log('Timeout: 3'), 0);
8
9   process.nextTick(() => console.log('nextTick: 1'));
10  Promise.resolve().then(() => console.log('Promise: 1'));
```
Dưới đây là kết quả khi chạy:

![image.png](https://images.viblo.asia/583bfd1f-c1e3-418f-b0bc-f42c8d10ff9a.png)

Ở giữa `Timeout: 2` và `Timeout: 3` xuất hiện thêm 2 log được khai báo trong callback của setTimeout line 2. Điều đó cho chúng ta thấy được rằng **mỗi lần thực thi xong 1 callback** trong Timer queue, Event Loop sẽ **kiểm tra callbacks trong Microtask queue** và **lấy ra thực thi** nếu có. 
Sau khi đã thực thi hết callbacks (line 4 và 5) trong Microtask queue thì mới xử lý callback (line 7) tiếp theo. 

Mọi người xem gif minh họa ở link sau để hiểu rõ hơn:

https://github.com/nntwelve/nodejs-event-loop/blob/main/example-2.gif

## IO queue 🟰
Hầu hết các async methods trong build-in modules sẽ đẩy các callback của nó vào I/O queue. Như sơ đồ ở trên chúng ta cũng biết được **I/O queue có độ ưu tiên thấp hơn Timer queue**. Tuy nhiên khi vận dụng cần lưu ý, ví dụ:
```typescript:6-phrase/example-3-io-queue.js
1  const fs = require('fs');
2
3  fs.readFile(__filename, () =>  console.log('Readfile 1'));
4
5  setTimeout(() => console.log('Timeout: 1'), 1000);
6  setTimeout(() => console.log('Timeout: 2'), 0);
7  setTimeout(() => console.log('Timeout: 3'), 0);
```

Chạy vài lần liên tục và kiểm tra log ở console:
![image.png](https://images.viblo.asia/f1d4fcaf-4ea6-4118-9c26-2bae2cb57d11.png)

Có thể thấy từ hình trên, thứ tự hiển thị log giữa `Timeout: 2,3` và `Readfile 1` không được bảo đảm. **Callbacks chỉ được gửi vào I/O queue khi và chỉ khi nó completed**. Lý giải cho vấn đề này là do khi chúng ta dùng setTimeout với giá trị là 0 thì chưa chắc nó sẽ thực thi ngay lập tức. Dường như cơ chế của nó giống với **DOMTimer** trong code C++, khoảng thời gian để setTimeout đẩy callback vào Timer queue sẽ giới hạn trong khoảng 1 millisecond hoặc 1 millisecond nhân với thời gian chúng ta truyền vào tham số thứ 2 của setTimeout. Điều đó có nghĩa là nếu chúng ta nhập vào 0 thì khoảng thời gian đó sẽ là max(1,0), chính là 1. 

Vậy 1 millisecond đó đã ảnh hưởng để log như thế nào? Khi control đến Event Loop, NodeJS cần kiểm tra xem thời gian 1ms đã trôi qua hay chưa, dẫn đến xảy ra 2 trường hợp sau:
- Khi control đến trong **khoảng > 0 và < 1ms**, trong khoảng thời gian đó callback của setTimeout vẫn chưa được đẩy vào Timer queue, control sẽ đến I/O queue => 'Readfile 1' in ra trước
- Khi control đến **> 1ms** do CPU busy, lúc này callback của setTimeout đã được thêm vào Timer queue, do đó 'Timeout: 1' và 'Timeout: 2' được in ra trước, sau đó control đến I/O queue và in ra 'Readfile 1'

Do quá trình trên phụ thuộc vào CPU busy nên chúng ta sẽ không đảm bảo được thứ tự thực thi của setTimeout với tham số là 0 nếu không có sự can thiệp như bên dưới:

```typescript:example-3-io-queue.js
const fs = require('fs');

fs.readFile(__filename, (err, data) => {  console.log('Readfile 1') });

setTimeout(() => console.log('Timeout: 1'), 1000);
setTimeout(() => console.log('Timeout: 2'), 0);
setTimeout(() => console.log('Timeout: 3'), 0);

for (let index = 0; index < 20000000; index++) {}
```
Với việc thêm vòng lặp vào làm cho CPU busy và control không đến được Event loop sớm nên thứ tự đã được đảm bảo
![image.png](https://images.viblo.asia/a895cc60-8fcf-4fe0-8b18-1d6a7655a3b8.png)

Vậy là chúng ta đã chứng minh được I/O queue sẽ thực thi sau Timer queue trong điều kiện lý tưởng. Tuy nhiên chúng ta cần lưu ý 1 điều với I/O queue, đó chính là giai đoạn I/O Polling.
### I/O Polling

Quay lại hình trên ở giữa I/O queue và Check queue có một giai đoạn là I/O Polling, đó là nơi epoll, kqueue, ... kiểm tra xem các async task đã hoàn thành hay chưa và nếu hoàn thành sẽ đẩy callback vào I/O queue. Xét ví dụ bên dưới để thấy rõ hơn:

```typescript:example-4-io-polling.js
1 const fs = require('fs');
2
3 setTimeout(() => console.log('Timeout: 1'), 0);
4
5 fs.readFile(__filename, () => console.log('Readfile: 1'));
6
7 setImmediate(() => console.log('Immediate: 1'));
8
9 for (let index = 0; index < 20000000; index++) {}
```

Theo như sơ đồ ở trên kèm với vòng lặp `for` để làm CPU busy chúng ta có thể dễ dàng đoán ra rằng kết quả sẽ là: `Timeout: 1` => `Readfile: 1` => `Immediate: 1`. Tuy nhiên mọi chuyện không đơn giản như chúng ta đã nghĩ, cùng check kết quả bên dưới:

![image.png](https://images.viblo.asia/2853c965-6142-4367-9795-ad00faa2b642.png)

`Readfile: 1` in ra cuối cùng 🤯, rõ ràng có gì đó mờ ám ở đây. Đó là do chúng ta chưa nói đến phần **I/O Polling**:
- Khi control đến **I/O Phrase** lúc này không có callback nào bên trong **I/O queue**, nó tiếp tục thực hiện quá trình polling, và nó sẽ kiểm tra `fs.readFile` của line 5 đã hoàn thành chưa, `readFile` báo rằng đã hoàn thành nên callback của nó được Event loop đẩy vào **I/O queue**. 
- Tuy nhiên lúc này control đã đi qua I/O queue nên nó không thể lấy callback đó ra thực thi ngay mà chuyển sang **Check queue** để thực hiện callback line 7. 
- Sau khi thực thi xong Check queue control tiếp tục, và do trong Event Loop vẫn còn callback nên nó tiếp tục vòng loop thứ 2, control chuyển dần đến I/O queue lần nữa, lúc này callback của line 5 mới được thực thi và in `Readfile: 1` ra màn hình.

Đó là lý do tại sao chúng ta có kết quả ở trên.

## Check queue ✅️
Như đã đề cập ở phần trên, Check queue sẽ xử lý các callback được khai báo trong `setImmediate`, và tương tự các queue trước đó, giữa quá trình thực thi mỗi callback đều sẽ thực thi các callback ở Microtask queue trước:
```typescript:example-5-check-queue.js
1  const fs = require('fs');
2 
3  fs.readFile(__filename, (err, data) => {
4  console.log('Readfile 1');
5  setImmediate(() => {
6    console.log('Immediate 1');
7    process.nextTick(() => console.log('nextTick 1'));
8  });
9  setImmediate(() => console.log('Immediate 2'));
10 });
11
12 setTimeout(() => console.log('Timeout: 1'), 0);
13
14 for (let index = 0; index < 10e7; index++) {}
```

![image.png](https://images.viblo.asia/826dadf7-0d91-45a7-8802-0e1ff965af31.png)

Kết quả trên chúng ta cũng thấy được mặc dù callback ở line 7 được thêm vào sau callback ở line 9 nhưng nó vẫn được thực thi trước.
## Close queue 📴
Đây là queue cuối cùng trong Event loop giúp xử lý các callback liên quan đến quá trình close event của các async task. Ví dụ bên dưới về `createReadStream` sẽ cho chúng ta thấy rõ khả năng của Close queue:

```typescript:example-6-close-queue.js
const fs = require('fs');

const readableStream = fs.createReadStream(__filename);
readableStream.close();

readableStream.on('close', () => {
  console.log('readableStream close event callback');
});

setImmediate(() => console.log('Immediate: 1'));

setTimeout(() => console.log('Timeout: 1'), 0);

for (let index = 0; index < 10e7; index++) {}
```
Lần này kết quả sẽ như những gì chúng ta nghĩ đến trong đầu:

![image.png](https://images.viblo.asia/06a58000-31e9-4e9d-a22f-7613c085cf75.png)

# Kết luận 📝 
Chúng ta đã cùng tìm hiểu qua về các kiến thức lý thuyết về Event Loop trong NodeJS, các kiến thức này hơi hàn lâm nên đến phần sau chúng ta sẽ cùng tìm các ví dụ để vận dụng vào thực tế để xem lợi ích nó mang lại. 

Cảm ơn các bạn đã giành thời gian đọc bài viết, hẹn gặp lại 🎉

# Tài liệu tham khảo 🔍
* Node.js - the node.js event loop (no date) Node.js -. Available at: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick (Accessed: 30 May 2024). 
* Gopinath, V. (2023) Visualizing nexttick and promise queues in node.js event loop, Builder.io. Available at: https://www.builder.io/blog/NodeJS-visualizing-nextTick-and-promise-queues (Accessed: 30 May 2024). 
* Gopinath, V. (2023b) Visualizing the timer queue in node.js event loop, Builder.io. Available at: https://www.builder.io/blog/visualizing-nodejs-timer-queue (Accessed: 30 May 2024). 
* Gopinath, V. (2023b) Visualizing the I/O queue in the node.js event loop, Builder.io. Available at: https://www.builder.io/blog/visualizing-nodejs-io-queue (Accessed: 30 May 2024). 
* Gopinath, V. (2023a) Visualizing I/O polling in the node.js event loop, Builder.io. Available at: https://www.builder.io/blog/visualizing-nodejs-io-polling (Accessed: 30 May 2024). 
* Gopinath, V. (2023c) Visualizing the check queue in the node.js event loop, Builder.io. Available at: https://www.builder.io/blog/visualizing-nodejs-check-queue (Accessed: 30 May 2024). 
* Gopinath, V. (2023) Visualizing the close queue in the node.js event loop, Builder.io. Available at: https://www.builder.io/blog/visualizing-nodejs-close-queue (Accessed: 30 May 2024). 
* Keshishyan, A. (2019) Effective way understanding node.js event loop deeper, Medium. Available at: https://medium.com/preezma/node-js-event-loop-architecture-go-deeper-node-core-c96b4cec7aa4 (Accessed: 30 May 2024). 
* Introduction to libuv - the node.js event loop (2022) CodeAhoy. Available at: https://codeahoy.com/learn/libuv/ch1/ (Accessed: 30 May 2024). 
# Change log 📓
* May, 30, 2024 - Init document