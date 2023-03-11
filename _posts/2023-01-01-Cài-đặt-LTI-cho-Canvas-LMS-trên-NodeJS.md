---
Hiện tại mình thấy có nhiều trường Đại Học đã tích hợp các **LMS ( Learning Management System ) Platform** vào việc giảng dạy online hoặc bổ trợ cho giảng dạy tại lớp. **LMS** mang đến nhiều tiện lợi cho trong việc quản lí của nhà trường và giảng viên trở nên dễ dàng hơn. 

Bên cạnh các chức năng có sẵn của **LMS**, đôi khi chúng ta cần bổ sung thêm các tính năng theo nhu cầu, **LTI** sẽ là công cụ giúp chúng ta giải quyết các nhu cầu đó . Trong bài viết này chúng ta sẽ cùng tìm hiểu sơ qua về **LTI** và cách cài đặt **LTI** vào **Canvas LMS** để các bạn có thể dễ dàng ứng dụng vào các dự án về **LTI** trong **LMS** sau này.

Các bạn có thể tải về source code [ở đây](https://github.com/nntwelve/LTI-with-nodejs).
# LTI là gì?
**LTI ( Learning Tools Interoperability )** là một tiêu chuẩn tích hợp cho các công cụ của bên thứ ba và được phát triển bởi **IMS Global Learning Consortium**. **Canvas** (hoặc các nền tảng **LMS** khác) hỗ trợ **LTI**, cho phép dễ dàng tích hợp và sử dụng các công cụ của bên thứ ba.

Có rất nhiều **LTI** có sẵn trên [App Center](https://www.eduappcenter.com/) để chúng ta sử dụng hoặc có thể tự tạo **LTI** riêng để phát triển những chức năng phù hợp với nhu cầu của dự án. Ở đây mình sẽ hướng dẫn các bạn tự tạo một **LTI** riêng có chức năng hiển thị role của account hiện tại đang đăng nhập trong **LMS**, sau đó kết nối với **Canvas LMS**.
# LTI hoạt động như thế nào?
![image.png](https://images.viblo.asia/ccb55460-0780-47b5-950a-1ad4e45f4936.png)
# Cài đặt LTI trên Canvas
Để tạo **LTI** mình sẽ sử dụng package [ltijs](https://www.npmjs.com/package/ltijs) trên NPM. Bước đầu tiên chúng ta cần tạo Developer key trên Canvas để khởi tạo các config cần thiết, các giá trị này giúp xác thực giữa ltijs và Canvas. Quy trình tạo Developer key bao gồm các bước sau:

1. Login vào Canvas, sau đó vào **Admin** ➡️  **Root account** ( ở đây có thể khác nhau về tên tùy theo config của các bạn, thông thường url sẽ là https://your-canvas-domain/accounts/1 ) 
2. Bấm chọn **Developer keys** ở giao diện vừa hiển thị.
3. Ở giao diện **Developer keys**, bấm chọn button **|+ Developer Key|** và sau đó chọn **|+ LTI Key|** như hình bên dưới:
![image.png](https://images.viblo.asia/14e1c4eb-02bd-4898-bd0e-4600db00ed2c.png)
4. Giao diện tạo **LTI Key** xuất hiện, vì phần giao diện config nay hơi dài nên mình sẽ chia làm 2 phần.
***Phần 1:*** chúng ta sẽ điền vào các trường như hình bên dưới: 
![image.png](https://images.viblo.asia/78f88206-feef-44aa-a32b-5a6b46aa41f4.png)
   **Lưu ý**: 
   * Các trường có dấu "*" thì bắt buộc phải điền. 
   * Port *3333* trong http://localhost:3333 chính là port server của **LTI** lát nữa chúng ta sẽ cấu hình ở phần sau, các bạn có thể chọn port tùy ý. 
   * Phần *JWK Method* chuyển về *Public JWK URL* ( mặc định *Public JWK* )

   ***Phần 2:*** khi kéo xuống sẽ thấy giao diện tiếp theo
![image.png](https://images.viblo.asia/f6bb0d6d-50b2-4b3a-9896-1d990fe1a421.png)
    * Ở mục **LTI Advantage Services** sẽ giúp phân quyền cụ thể hơn các dữ liệu từ Canvas sang LTI, tùy theo yêu cầu dự án sẽ bật các option khác nhau. Nếu các bạn chưa quen có thể bật hết để dễ thao tác.
    * Phần **Additional Settings** mình sẽ sử dụng trường **Custom fields** để gửi kèm role của account hiện tại từ Canvas sang LTI, các giá trị này sẽ được gửi kèm theo response trong kết nối giữa Canvas và LTI. Tham khảo thêm các biến có thể gửi đi ở [đây](https://canvas.instructure.com/doc/api/file.tools_variable_substitutions.html).
    
5. Sau khi ấn button **Save** chúng ta sẽ được 1 Developer Key mới như hình dưới, ấn **ON** ở cột *State*. Dãy số dài là *Client ID*, bấm vào *Show key* để hiển thị *Secret key*.
 ![image.png](https://images.viblo.asia/59f4eb3f-fca2-46da-b30b-0057d478b700.png)
6. Để hiển thị LTI lên Canvas chúng ta cần thêm 1 bước cuối cùng là tạo External App, bấm vào **Setting** ➡️ **App** như hình dưới, sau đó chọn **+ App**:
![image.png](https://images.viblo.asia/f3ea3749-ff8e-4a47-8038-b18ce770cfd9.png)
7. Popup mới hiện lên, ở phần **Configuration Type** chọn *By Client ID*, sau đó nhập vào *Client ID* vừa tạo ở bước 5, bấm **Submit** ➡️ **Install** :
![image.png](https://images.viblo.asia/59f4c2f9-6035-44a6-af94-7ee202dec836.png) 
8. Refresh lại trang các bạn sẽ thấy LTI đã được hiển thị trên Canvas ( Phía dưới ***Developer keys*** ở sidebar - vì khi nảy ở Placement trong phần config Developer key chúng ta có để *Account Navigation*, tương tự khi vào Course cũng sẽ hiển thị do giá trị của Placement có chứa *Course Navigation*):
![image.png](https://images.viblo.asia/61dbb0ca-539d-4f8e-a7d6-e41bd35b77e0.png)

Đó là toàn bộ các bước cơ bản để tạo External App sử dụng LTI trên Canvas, chúng ta sẽ đến với phần tiếp theo là cấu hình trên NodeJS.
# Cài đặt LTI trên NodeJS
Chúng ta sẽ tiến hành cài đặt package [ltijs](https://www.npmjs.com/package/ltijs), thời điểm hiện tại của bài viết mình đang dùng **ubuntu**:*24.04*, **nodejs**:*v19.3.0* và package  **ltijs**:*v5.9.0*.
1. Tạo thư mục và khởi tạo project nodejs:

     `mkdir lti-account-role && cd lti-account-role`
     
     `npm init -y`
     
 2. Cài đặt các package cần thiết:
    
    `npm install ltijs dotenv`
 3. Chạy MongoDB:
>  This package natively uses mongoDB by default to store and manage the server data, so you need to have it installed, see link bellow for further instructions.


   Dựa vào document từ phía nhà phát triển, chúng ta nên sử dụng **MongoDB** để thống nhất với **ltijs**. Tuy nhiên mọi người có thể sử dụng các Database Plugin khác như [Firestore Plugin](https://github.com/examind-ai/ltijs-firestore) hoặc [Sequelize Plugin](https://github.com/Cvmcosta/ltijs-sequelize)

  3. Tạo cấu trúc thư mục, vì đây là bài hướng dẫn cài đặt **LTI** nên mình sẽ tạo cấu trúc ngắn gọn để các bạn có thể triển khai một cách nhanh gọn và dễ dàng nhất
 
  ![image.png](https://images.viblo.asia/3bead5db-a3f0-489f-b8f2-7992f05ab5fa.png)
 Nội dung và ý nghĩa của các file như sau:

    ```shell:.env
    DATABASE_NAME=lti_acocunt_role
    DATABASE_USERNAME=admin
    DATABASE_PASSWORD=admin
    DATABASE_PORT=27022
    DATABASE_URI=mongodb://localhost:27022

    PORT=3333 // PORT để Canvas kết nối vào LTI

    LTI_HOST=https://your-canvas-domain.com // Các bạn đổi thành domain Canvas các bạn đang dùng nhé
    LTI_CLIENT_ID=10000000000022 //  Đây là Client ID từ Developer key chúng ta tạo khi nảy
    LTI_KEY=HkvZwP0DKtqWTUjX1qNjQdWiSBCZmGWNe7iRR73ke9MiosdVSrY583urVouN8mk5 // tương tự đây là Secret key từ Developer key
    LTI_NAME=LTI_ACCOUNT_ROLE 
    LTI_ISS=https://canvas.instructure.com // ISS phải trùng với ISS trong file security.yml trong config của Canvas
    ```

*   ***.env***: lưu các biến môi trường. MongoDB của mình đang chạy ở port 27022 các bạn đổi thành port các bạn đang sử dụng ( thông thường là 27017 ). Tương tự với **DATABASE_USERNAME** và **DATABASE_PASSWORD**. 

    ```javascript:lti.config.js
    import ltijs from 'ltijs';
    const lti = ltijs.Provider;
    // Setup
    const ltiSetupConfig = {
      LTI_KEY: process.env.LTI_KEY,
      DB_CONNECTION: {
        url:
          process.env.DATABASE_URI +
          '/' +
          process.env.DATABASE_NAME +
          '?authSource=admin',
        connection: {
          // user: process.env.DATABASE_USERNAME,
          // pass: process.env.DATABASE_PASSWORD,
        },
      },
      LTI_APP_CONFIG: {
        cookies: {
          // Set secure to true if the testing platform is in a different domain and https is being used
          secure: false,
          // Set sameSite to 'None' if the testing platform is in a different domain and https is being used
          sameSite: '',
        },
        // Set DevMode to true if the testing platform is in a different domain and https is not being used
        devMode: true,
      },
    };

    // Setup LTI application
    lti.setup(
      ltiSetupConfig.LTI_KEY,
      ltiSetupConfig.DB_CONNECTION,
      ltiSetupConfig.LTI_APP_CONFIG
    );

    // Whitelisting the main app route and /nolti to create a landing page
    lti.whitelist(
      {
        route: new RegExp(/^\/nolti$/),
        method: 'get',
      },
      {
        route: new RegExp(/^\/ping$/),
        method: 'get',
      }
    ); // Example Regex usage

    // When receiving successful LTI launch redirects to app, otherwise redirects to landing page
    lti.onConnect((token, req, res) => {
      if (token) {
        return res.json(res.locals.context.custom.role);
      } else {
        lti.redirect(res, '/nolti');
      } // Redirects to landing page
    });

    // Setup function
    lti.setUpPlatform = async () => {
      await lti.deploy({ port: process.env.PORT });

      const ltiPlatConfig = {
        url: process.env.LTI_ISS,
        name: process.env.LTI_NAME,
        clientId: process.env.LTI_CLIENT_ID,
        authenticationEndpoint: `${process.env.LTI_HOST}/api/lti/authorize_redirect`,
        accesstokenEndpoint: `${process.env.LTI_HOST}/login/oauth2/token`,
        authConfig: {
          method: 'JWK_SET',
          key: `${process.env.LTI_HOST}/api/lti/security/jwks`,
        },
      };

      await lti.registerPlatform(ltiPlatConfig);
    };

    export { lti };
    ```

*   ***lti.config.js***: Chúng ta sẽ thực hiện config **ltijs** ở file này. 
    *   Function **lti.setup** giúp chúng ta config secret key và database connection. 
    *   Function **lti.whitelist** sẽ whitelist các route không cần xác thực mà vẫn có thể truy cập bên ngoài Canvas. Nếu các bạn không cần whitelist route nào thì có thể xóa phần config này.
    *   Function **lti.onConnect** khi khởi chạy LTI thành công từ Canvas function này sẽ được gọi. Mục đích của chúng ta là trả về account role nên mình sẽ lấy nó ra từ token `res.locals.context.custom.role`. Khi triển khai dự án thực tế chúng ta sẽ có giao diện từ Front-end nên phần `return res.json(res.locals.context.custom.role)` sẽ được thay thế thành `return res.sendFile()` hoặc `res.redirect()` tùy theo cách dùng của mỗi người.
    *   Function **lti.setUpPlatform** dùng để config port chạy server ltijs và config các biến từ Developer key tạo khi nảy.

    ```javascript:app.config.js
    import dotenv from 'dotenv';
    dotenv.config();
    import bodyParser from 'body-parser';

    export class Application {
      constructor(lti) {
        this.lti = lti;

        this.initializeMiddlewares();
        this.initializeRoutes();
        this.initializeErrorHandling();
      }

      initializeErrorHandling() {}

      initializeMiddlewares() {
        this.lti.app.use(bodyParser.json());
      }

      initializeRoutes() {
        this.lti.app.get('/ping', async (_req, res) => {
          res.send({
            message: 'pong',
          });
        });
      }

      listen() {
        this.lti.setUpPlatform();
      }
    }
    ```

* **app.config.js**: dùng để khởi tạo application, ở file lti.config mình cho whitelist route /ping để test **ltijs** đã chạy chính xác hay chưa. Thông thường khi server chạy ltijs sẽ chỉ được gọi từ Canvas, nếu chúng ta cố gắng truy cập trực tiếp **ltijs** sẽ trả về **NO_LTIK_OR_IDTOKEN_FOUND**. Mình có tạo  một số method initialize các bạn không dùng có thể xóa.

    ```javascript:main.js
    import { Application } from './configs/app.config.js';
    import { lti } from './configs/lti.config.js';

    const app = new Application(lti);
    app.listen();

    export { app };
    ```

* **main.js**: Khởi chạy server

4. Chỉnh sửa **package.json** thêm script:

    `"start": "node src/main.js"`

5. Chạy server bằng lệnh:

    `npm start`

* Nếu phản hồi như hình dưới thì server đã khởi chạy thành công.
![](https://images.viblo.asia/532f385c-88a1-4ec1-b90c-2821034f1f55.png)

* Truy cập http://localhost:3333/ping để kiểm tra. Nếu message trả về là "*pong*" thì đã set up thành công. 


![](https://images.viblo.asia/6aed2d94-4d03-4e3e-b2ce-e0fb0de5ad57.png)

6. Quay lại **Canvas** và mở **LTI** để xem kết quả, như hình dưới thì chúng ta đã thành công trả về role của account đang đăng nhập **Canvas** - mình đang đăng nhập bằng account *Admin* nên trong response sẽ có *Administrator* đối với account *Student* thì sẽ chỉ hiển thị *Student* . Do hiện tại là môi trường dev nên server **LTI** không có https sẽ hiển thị 1 số *warning* từ **Canvas**. Khi triển khai production cùng https các warning đó sẽ biến mất.

![image.png](https://images.viblo.asia/46d7b3aa-e8cc-4dad-a89b-2f1356da2de0.png)

# Các lỗi thường gặp
1. ***UNREGISTERED_PLATFORM***: thông thường lỗi này do **LTI_ISS** trong file **.env** của các bạn không trùng với **ISS** trong security config của Canvas LMS. Cập nhật lại để 2 bên trùng khớp, nếu lỗi vẫn còn xuất hiện các bạn truy cập vào table **platforms** trong database xóa record chứa **LTI_ISS** cũ và restart server.
2. ***TOKEN_TOO_OLD***: lỗi này thường do network chậm, các bạn refresh hoặc hard refresh một vài lần là có thể vào lại được.
# Kết luận
Vậy là chúng ta đã tạo xong 1 custom LTI, các bạn có thể dựa vào đó mà phát triển tiếp các chức năng tùy theo yêu cầu của dự án mà mình tham gia. 
Hiện tại chúng ta đang sử dụng server của **ltijs** - được chạy bằng **ExpressJS** - 
để chạy LTI, nó sẽ có các mặt thuận tiện và bất tiện riêng. Ở phần 2 mình sẽ hướng dẫn các bạn tích hợp **ltijs** vào **NestJS Framework** để chạy LTI như 1 route riêng bằng *serverless option*. Phần cuối của series mình sẽ là tích hợp FE chạy bằng ReactJS vào để khởi chạy 1 LTI hoàn chỉnh.

Cảm ơn các bạn đã theo dõi. Nếu có câu hỏi gì các bạn hãy comment phía dưới hoặc inbox riêng cho mình. Chúc các bạn có 1 năm Quý Mão thật là vui vẻ và thành công.
# Tài liệu tham khảo
* https://ltiaas.com/docs/introduction/
* https://infocanvas.upenn.edu/lti-policy-for-canvas/
* https://www.npmjs.com/package/ltijs
---
