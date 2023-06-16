---
layout: post
title: 'Cùng mình tạo Boilerplate cho dự án NextJS v12 - Phần 2: i18n và NextAuthjs'
date: 2023-03-09 17:27:12 -0000
categories: NextJS
---

<!--
Xin chào mọi người, ở phần trước chúng ta đã cùng nhau cấu hình dự án với **axios-hooks** để tăng tính linh hoạt và thuận tiện trong lúc phát triển cũng như bảo trì và cũng cài đặt MUI với **@emotion/cache** để optimize performance của website - bạn nào chưa đọc [Phần 1 có thể xem lại ở đây](https://viblo.asia/p/cung-minh-tao-boilerplate-cho-du-an-nextjs-v12-voi-mui-v5-va-axios-hooks-phan-1-GyZJZnk2Jjm). Phần tiếp theo này chúng ta sẽ cùng tích hợp **i18n** để ứng dụng đa ngôn ngữ cho dự án, sau đó tích hợp **authentication** ( bao gồm _access_token_ và _refresh_token rotation_ ) với package **NextAuth.js**.

Các bạn có thể tại về source code **Phần 1** [ở đây](https://github.com/nntwelve/Boilerplate-NextJS-12-With-Axios-Hooks) và checkout ra branch **[part-1](https://github.com/nntwelve/Boilerplate-NextJS-12-With-Axios-Hooks/tree/part-1)** để thực hành theo bài viết hoặc sử dụng source code hoàn chỉnh ở branch **[main](https://github.com/nntwelve/Boilerplate-NextJS-12-With-Axios-Hooks)**.

# 1. Translation với next-i18next

![image.png](https://images.viblo.asia/868be6da-947d-43f2-a097-a9266854c59f.png)

## 1.1 Tại sao mình chọn dùng next-i18next?

Nếu như các bạn đã từng implement chức năng _translation_ cho website, ít nhiều thì cũng đã từng nghe qua **i18n** và **react-i18next**. Đó là các package support rất tốt về mảng _translation_, tuy nhiên để sử dụng với **SSG** và **SSR** thì vẫn còn một vài vấn đề phát sinh. Cho nên **next-i18next** được mở rộng ra từ **react-i18next** để giải quyết các vấn đề đó và vì thế chúng ta không cần lo về việc các tính năng có trên **react-i18next** mà trên **next-i18next** không có. Cùng mình điểm qua một số tính năng nổi bậc:

- Dễ cài đặt, dễ sử dụng.
- Tự động detect ngôn ngữ trình duyệt của người dùng.
- Đơn giản hóa internationalisation cho ứng dụng **NextJS** mà không cần thêm những dependencies khác.
- Support tốt với **SSG** và **SSR**.
- Dễ dàng bảo trì, quản lý các bản dịch cho ứng dụng với API đơn giản và trực quan cho phép thêm và cập nhật các bản dịch một cách dễ dàng.

## 1.2 Cài đặt và cấu hình next-i18next

> Lưu ý: ở phạm vi bài viết này mình sẽ phát triển dự án theo hướng chạy _production_ ở FE bằng **NodeJS** server chứ không sử dụng **static HTML** nên sẽ không sử dụng lệnh `next export`. Vì khi export **static HTML** một vài tính năng của **NextJS** sẽ không được hỗ trợ. Trong đó có **Internationalized Routing**, nguyên nhân làm cho chúng ta gặp lỗi với **i18n** khi chạy lệnh `next export`.
>
> Tuy nhiên vẫn có cách giải quyết nếu trong trường hợp khi yêu cầu từ dự án bắt buộc chúng ta phải sử dụng `next export` để export **static HTML** thì các bạn có thể chỉnh sửa lại logic code loại bỏ **International Routing** thay vào đó sử dụng tính năng [dynamic route segments](https://nextjs.org/docs/routing/introduction#dynamic-route-segments) thì sẽ giải quyết được lỗi trên. Các bạn có thể xem thêm [ở đây](https://dev.to/adrai/static-html-export-with-i18n-compatibility-in-nextjs-8cd).

- Tiến hành cài đặt với lệnh:

  `npm i next-i18next react-i18next i18next`

- Bước đầu tiên trong việc cấu hình chúng ta sẽ tạo file **next-i18next.config.js**

  ```js:next-i18next.config.js
  module.exports = {
    i18n: {
      defaultLocale: 'en',
      locales: ['en', 'vi'],
    },
  }
  ```

- Giải thích: chúng ta khai báo ngôn ngữ mặc định và các ngôn ngữ có thể có của dự án để có thể tải trước trên server.
- Sau đó thêm config chúng ta vừa tạo vào file **next.config.js**. Việc này giúp **NextJS** _enable localised URL routing_. Trước đó khi chưa config **i18n** nếu các bạn truy cập vào http://localhost:3000/vi/users sẽ trả về **404 Page Not Found**.

  ```js:next.config.js
  /** @type {import('next').NextConfig} */
  const { i18n } = require('./next-i18next.config');

  const nextConfig = {
    reactStrictMode: true,
    i18n,
  };

  module.exports = nextConfig;
  ```

- Cuối cùng là chỉnh sửa lại file **\_app.tsx** để thêm vào `i18nextProvider`. Dựa theo install document chúng ta sẽ sử dụng `appWithTranslation` HOC để wrap component `App` lại để có thể sử dụng **next-i18n** với **SSR**.
  > From [**i18n** documentation](https://react.i18next.com/latest/i18nextprovider):
  >
  > **What it does**
  >
  > The I18nextProvider does take an i18next instance via prop i18n and passes that down using the context API.
  >
  > **When to use?**
  >
  > You will only need to use the provider if you need to support multiple i18next instances - eg. if you provide a component library (like this example) or in scenarios for SSR (ServerSideRendering).

> Nếu bạn nào gặp lỗi không hiển thị được ngôn ngữ đã dịch sau khi đã thêm đầy đủ các bước ở dưới thì có thể do quên thêm provider chỗ này.

```jsx:_app.tsx
    ...
    import { appWithTranslation } from 'next-i18next';
    ...
    function App({
      Component,
      emotionCache = clientSideEmotionCache,
      pageProps,
    }: ExtendedAppProps) {
    ...
    }
    export default appWithTranslation(App); // <= Thêm vào ở đây
```

- Vậy là chúng ta đã cấu hình xong cho **next-i18next**, trước khi sử dụng mình sẽ giới thiệu đến các bạn 1 package giúp chúng ta tăng tốc độ lập trình với **i18n**.

## 1.3 Cài đặt i18next-scanner

- Thông thường khi chúng ta cài đặt xong **i18n** thì phải tạo các resource file dạng json và thêm vào các _key/value_ dùng để translate, việc đó tốn khá nhiều thời gian nếu dự án cập nhật các key thường xuyên và có nhiều ngôn ngữ. Để giải quyết vấn đề này mình giới thiệu đến các bạn package **i18next-scanner**, package này giúp tự động extract _key/value_ trong code của chúng ta và merge vào các resource file **i18n**. Cài thêm package:

  `npm i --save-dev i18next-scanner`

- Sau đó tạo file cấu hình cho **i18next-scanner**

  ```javascript:i18next-scanner.config.js
  module.exports = {
    input: [
      'src/**/*.{ts,tsx}',
      // Use ! to filter out files or directories
      '!src/**/*.test.{ts,tsx}',
    ],
    output: './public',
    options: {
      debug: true,
      func: {
        list: ['t'],
        extensions: ['.ts', '.tsx'],
      },
      trans: false,
      lngs: ['en', 'vi'],
      defaultLng: 'en',
      defaultNs: 'common',
      defaultValue: function (lng, ns, key) {
        return key;
      },
      removeUnusedKeys: false,
      resource: {
        loadPath: 'public/locales/{{lng}}/{{ns}}.json',
        savePath: 'locales/{{lng}}/{{ns}}.json',
        jsonIndent: 2,
        lineEnding: '\n',
      },
      nsSeparator: false, // namespace separator
      keySeparator: false, // key separator
      interpolation: {
        prefix: '{{',
        suffix: '}}',
      },
    },
  };
  ```

- Giải thích, các bạn chú ý 3 option ở dưới:
  - option `input` dùng để chọn/loại các file chứa key translation, ở đây chúng ta dùng **Typescript** nên chỉ cần scan các file .ts.
  - option `output` đường dẫn để tạo ra các file resource **i18n**.
  - option `lngs` và `defaultLng` là ngôn ngữ mặc định và các ngôn ngữ mà chúng ta hỗ trợ, **i18next-scanner** sẽ căn cứ vào thông tin ở đây mà tạo ra các file resource cho các ngôn ngữ đó.
- Để sử dụng chúng ta sẽ thêm script mới vào **scripts** của **package.json**

  ```json:package.json
  ...
  {
    ...
    "scripts": {
      ...
      "i18n:extract": "i18next-scanner"
    },
    ...
  ```

- Hiện tại source của chúng ta chưa có translate nên chưa có gì để extract ra resource file **i18n**, mình sẽ thêm vào để kiểm tra thử. Chỉnh sửa lại file **src/pages/users/index.tsx** và **src/views/users/UserFilter.tsx**

      ```jsx:src/pages/users/index.tsx
      import { useTranslation } from 'react-i18next';
      import { serverSideTranslations } from 'next-i18next/serverSideTranslations';
      ...

      function UserPage() {
        const { t } = useTranslation();
        ...

        return (
          <>
            <Head>
              <title>{t('Users')}</title>
            </Head>
            ...
          </>
        );
      }

      export async function getStaticProps({ locale }: { locale: string }) {
        return {
          props: {
            ...(await serverSideTranslations(locale, ['common'])),
          },
        };
      }
      export default UserPage;
      ```

      ```jsx:src/views/users/UserFilter.tsx
      import { useTranslation } from 'react-i18next';
      ...
      function UserFilter({ filter, onChange, onCreatedUser }: Props) {
        const { t } = useTranslation();
        ...
        return (
          <Toolbar disableGutters>
            <Typography
              sx={{ flexGrow: 1, display: 'flex', alignItems: 'center' }}
              variant="h5">
              {t('All Users')}
              ...
        ...
      ```

  > Lưu ý: các component trong views chúng ta không cần sử dụng **getStaticProps**, chỉ sử dụng cho **page**:
  >
  > From [next-i18n documentation](https://github.com/i18next/next-i18next):
  >
  > The **serverSideTranslations** HOC is primarily responsible for passing translations and configuration options into pages, as props – you need to add it to any **page** that has translations.

- Sau khi tạo xong chúng ta sẽ generate ra resource file **i18n** bằng lệnh:

  `npm run i18n:extract`

- **i18n-scanner** theo config khi nảy sẽ tạo cho chúng ta folder **locales** trong folder **public**, với nội dung là các key chúng ta đã tạo

![image.png](https://images.viblo.asia/8ec737a0-f84b-48a5-ba04-3e16071b4ad5.png)

- Dùng **ChatGPT** để dịch tự động các cặp _key/value_ sang tiếng Việt. Bắt đầu với câu `I'll send you "key":"value" pairs of JSON translation file, please translate and replace "value" with Vietnamese`. Sau đó gửi thông tin file JSON và nhận kết quả trả về.

![image.png](https://images.viblo.asia/510e8d4a-7066-4c1a-9a4b-bfd2abe3b33c.png)

- Mở file **common.json** trong folder **vi** cập nhật lại các _key/value_ từ kết quả của **ChatGPT**:
  ```json:vi/common.json
  {
    "All Users": "Tất cả người dùng",
    "Users": "Người dùng"
  }
  ```
  >
- Start lại next bằng lệnh `npm run dev` vì các thay đổi bên ngoài thư mục **src** next server sẽ không tự động cập nhật. Truy cập http://localhost:3000/vi/users để kiểm tra kết quả:

![image.png](https://images.viblo.asia/8eae56bc-ebdb-4e7c-8360-4ce17b893fd4.png)

- Thử lại với ngôn ngữ tiếng Anh http://localhost:3000/users hoặc http://localhost:3000/en/users đều được, vì chúng ta chọn mặc định là **en** nên không cần thiết phải thêm **/en**:

![image.png](https://images.viblo.asia/53a63c31-1f78-418f-8bca-b93077b579ca.png)

Vậy là chúng ta đã cài đặt xong đa ngôn ngữ cho dự án. Từ giờ trở đi mỗi khi các bạn có thêm các key mới chỉ cần sử dụng **i18next-scanner** để generate ra resource **i18n** và cập nhật value của key ở ngôn ngữ đó. Trường hợp có thêm ngôn ngữ mới thì thêm vào trong 2 file **next-i18next.config.js** và file **i18next-scanner.config.js**.

> Nếu các bạn lo lắng khi chạy lệnh `npm run i18n:extract` làm cho các key có value đã dịch rồi bị reset lại thì yên tâm, **i18next-scanner** sẽ tự động detect những key có value đã được dịch, nó chỉ thêm vào các key mới nên sẽ không bị vấn đề đó.

# 2. Authentication với NextAuth.js

![image.png](https://images.viblo.asia/0b568712-3f4c-4b01-bf84-300f2052cb95.png)

## 2.1 Tại sao mình chọn dùng NextAuth.js

Hiện nay **OAuth** đã và đang phát triển rất mạnh mẽ, trở thành một tiêu chuẩn quan trọng trong việc xác thực và ủy quyền truy cập. Bên cạnh việc đăng nhập bằng _username_ và _password_ thông thường khách hàng sẽ yêu cầu chúng ta tích hợp thêm các phương thức khác như **Google**, **Github**, **Facebook**,... Cho nên chúng ta không thể bỏ qua việc tích hợp **OAuth** vào dự án của mình nếu muốn đáp ứng được nhu cầu của người dùng hiện nay. Tuy nhiên việc tự viết cho từng phương thức sẽ mất không ít thời gian và công sức, đặc biệt là với những bạn mới làm quen với lập trình. **NextAuth.js** được sinh ra để hỗ trợ chúng ta trong vấn đề đó, cùng mình điểm qua một số lợi ích của nó với sự trợ giúp của **ChatGPT**:

![image.png](https://images.viblo.asia/5d921fbd-6446-4f4b-afd1-2c598a2e6993.png)

Có thể thấy chúng ta sẽ không cần phải mất thời gian để tự quản lí **Auth context** hoặc thêm các **Providers** mới khi có yêu cầu từ phía khách hàng, việc đó **NextAuth** sẽ hỗ trợ cho chúng ta.

## 2.2 Cấu hình NextAuth.js

- Tiến hành cài đặt dựa theo [document](https://next-auth.js.org/getting-started/example):

  `npm install next-auth`

- Tiếp theo để thêm **NextAuth.js** chúng ta sẽ tạo file **[...nextauth].ts** trong folder **src/pages/api/auth**. File này sẽ bao gồm _dynamic route handler_ và tất cả các cấu hình global của **NextAuth.js** ( các Provider mà website chúng ta hỗ trợ như **Google**, **Github**, **Facebook** sẽ được thêm vào ở đây ). Tất cả các requests đến /**api/auth/\*** (**signIn**, **callback**, **signOut**, ...) sẽ được tự động xử lý bởi **NextAuth.js** theo như cài đặt của chúng ta ở đây ( lát nữa mình sẽ dùng **/api/auth/signin** để sử dụng giao diện đăng nhập mặc định mà **NextAuth.js** cung cấp ).

  ```ts:src/pages/api/auth/[...next-auth].ts
  import NextAuth, { NextAuthOptions } from 'next-auth';
  import { Provider } from 'next-auth/providers';

  const providers: Provider[] = [
    // ...add providers here
  ];
  export const authOptions: NextAuthOptions = {
    providers,
  };

  export default NextAuth(authOptions);
  ```

- Giải thích: biến `providers` sẽ chứa các providers mà ứng dụng của chúng ta hỗ trợ ( **Username/password**, **Github**, **Facebook**, **Google**,...).
- Bước cuối cùng để có thể sử dụng **useSession** ( mục đích chính là dùng kiểm tra xem user đã đăng nhập chưa và lấy ra thông tin user ), chúng ta sẽ expose _session context_. Thêm `<SessionProvider />` wrap bên ngoài component `<Component />` trong file **\_app.tsx**:

```jsx:_app.tsx
...
import { SessionProvider } from 'next-auth/react';
...
function App({
  Component,
  emotionCache = clientSideEmotionCache,
  pageProps: { session, ...pageProps },
}: ExtendedAppProps) {
  return (
    <CacheProvider value={emotionCache}>
      <ThemeProvider theme={lightTheme}>
        {/* CssBaseline kickstart an elegant, consistent, and simple baseline to build upon. */}
        <CssBaseline />
        <SessionProvider session={session}>
          <Component {...pageProps} />
        </SessionProvider>
      </ThemeProvider>
    </CacheProvider>
  );
}

export default appWithTranslation(App);
```

- Từ giờ các instance tạo ra từ **useSession** sau đó sẽ có quyền truy cập vào _status_ và _data session_. `<SessionProvider />` cũng đảm nhiệm việc giữ cho session được cập nhật và đồng bộ hóa giữa các tab và cửa sổ trình duyệt.
  > Lưu ý: nếu quên config `SessionProvider` chúng ta sẽ gặp lỗi như bên dưới
  >
  > ![image.png](https://images.viblo.asia/b95efeed-0219-41a1-8b0b-a82997c9f429.png)
  >
  > Quá trình cấu hình đã xong, tiếp theo chúng ta sẽ cho **NextAuth.js** protect những trang mà user cần đăng nhập mới có thể truy cập được.

## 2.3 Cài đặt middleware để protect page

Bằng việc sử dụng **NextJS Middleware** với [**NextAuth.js**](https://next-auth.js.org/configuration/nextjs#basic-usage) chúng ta có thể chỉ định các trang cần được xác thực trước khi truy cập:

- Tạo file **middleware.ts** bên trong folder **src**. Mình sẽ chỉ cho phép user truy cập trang chủ, còn trang `/users` thì cần đăng nhập mới có thể truy cập.

  ```ts:middleware.ts
  export { default } from "next-auth/middleware"

  export const config = { matcher: ['/users'] }
  ```

- Thử truy cập http://localhost:3000 và http://localhost:3000/users để xem kết quả
  _ http://localhost:3000
  ![image.png](https://images.viblo.asia/34fa614b-f571-4988-a559-7c52e60ce8ac.png)
  _ http://localhost:3000/users sẽ tự động điều hướng chúng ta qua trang đăng nhập ( hiện tại chúng ta chưa cài đặt providers nên trang đăng nhập mặc định sẽ không có gì để hiển thị ).
  ![image.png](https://images.viblo.asia/75e2af85-fa91-4c04-8af2-b2054877d1c0.png)
  Bước tiếp theo chúng ta sẽ tạo provider đăng nhập với **Username/password**.

## 2.4 Login flow trong NextAuth.js

![](https://images.viblo.asia/c58521a4-2ab8-428b-b437-7b170f32b042.png)

## 2.5 Cài đặt login với username/password

- Để login bằng **Username/password** chúng ta sẽ dùng [Credentials Provider](https://next-auth.js.org/configuration/providers/credentials) được cung cấp sẵn bởi **NextAuth.js**. Chỉnh sửa lại file **[...next-auth].ts**

```ts:[...next-auth].ts
import axios from 'axios';
import NextAuth, { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import { AppConfig } from '@/configs/app.config';
import { Provider } from 'next-auth/providers';

const providers: Provider[] = [
  CredentialsProvider({
    // The name to display on the sign in form (e.g. 'Sign in with...')
    name: 'Credentials',
    // The credentials is used to generate a suitable form on the sign in page.
    // You can specify whatever fields you are expecting to be submitted.
    // e.g. domain, username, password, 2FA token, etc.
    // You can pass any HTML attribute to the <input> tag through the object.
    credentials: {
      username: { label: 'Username', type: 'username', placeholder: 'jsmith' },
      password: { label: 'Password', type: 'password' },
    },
    async authorize(credentials, req) {
      // You need to provide your own logic here that takes the credentials
      // submitted and returns either a object representing a user or value
      // that is false/null if the credentials are invalid.
      // e.g. return { id: 1, name: 'J Smith', email: 'jsmith@example.com' }
      // You can also use the `req` object to obtain additional parameters
      // (i.e., the request IP address)
      try {
        let res;
        if (AppConfig.enableApiMockup) {
          res = {
            status: 200,
            data: {
              access_token: 'mock_access_token',
              refresh_token: 'mock_refresh_token'
            },
          };
        } else {
          res = await axios.post(`http://localhost:3336/auth/sign-in`, {
            username: credentials?.username,
            password: credentials?.password,
          });
        }
        // If no error and we have user data, return it
        if (res.status === 200) {
          return {
            ...res.data,
            name: credentials?.username,
          };
        }
        // Return null if user data could not be retrieved
        return null;
      } catch (error) {
        return null;
      }
    },
  }),
  // ...add more providers here
];

export const authOptions: NextAuthOptions = {
  // Configure one or more authentication providers
  providers,
  callbacks: {
    async jwt({ token, user, account }) {
      // Persist the OAuth access_token to the token right after signin
      if (user) {
        token.access_token = user.access_token;
        token.refresh_token = user.refresh_token;
      }
      return token;
    },
    async session({ session, token }) {
      // Send properties to the client, like an access_token from a provider.
      session.access_token = token.access_token;
      return session;
    },
  },
};

export default NextAuth(authOptions);
```

- Giải thích

  - Các options trong `CredentialsProvider`
    - `name` là tên sẽ hiển thị trên form đăng nhập ( Sign in with `name` ).
    - `credentials` các field sẽ hiển thị trên form đăng nhập để nhập thông tin. Nếu các bạn dùng **email** thì có thể tinh chỉnh lại đổi **username** thành **email**.
    - `authorize` dùng để xử lí logic login. Phần này mình cũng sẽ kết hợp tính năng chuyển qua lại giữa mock data và API.
    - Vì API login của **reqres.in** thông tin trả về không phải **jwt** nên không có thời gian hết hạn, mình sẽ tạm thời sử dụng API local tự tạo ( logic đơn giản là gửi `username` và `password` đi và trả về `access_token` và `refresh_token` với type **jwt** ).
  - Theo như tài liệu thì **Credentials Provider** chỉ có thể sử dụng khi bật **JWT** với **session**. Do đó chúng ta sẽ thêm method `jwt` trong `callbacks` option trong `NextAuthOptions`
  - Các options trong `NextAuthOptions`
    - `secret`: sử dụng để _hash token_, _sign/encrypt_ cookies và tạo _cryptographic keys_. Hiểu đơn giản là thông tin được mã hóa vào cookie và mỗi lần sử dụng sẽ lấy từ cookie ra giải mã trả về cho người dùng, tránh trường hợp bị lộ _sensitive data_ khi cookie người dùng bị đánh cắp. Chúng ta có thể thêm biến `NEXTAUTH_SECRET` vào file **.env** để không phải thêm option ở đây, **NextAuth.js** tự động kiểm tra và nhận vào cấu hình.
    - `providers`: chứa các provider chúng ta dùng để đăng nhập.
    - `callbacks`: sử dụng để kiểm soát những gì xảy ra khi một hành động được thực hiện.
      - `jwt`: sẽ được gọi bất cứ khi nào **JWT** được tạo ( ví dụ khi đăng nhập ) hoặc được cập nhật ( bất cứ khi nào **session** được truy cập ở phía _client_ ). Các requests tới **/api/auth/signin**, **/api/auth/session** và gọi tới **getSession()**, **getServerSession()**, **useSession()** sẽ gọi tới function này. Ở đây mình sẽ dùng để gắn thông tin `access_token` và `refresh_token` trả về từ API cho token của **NextAuth.js**, sau đó sẽ dùng nó để truyền các biến cần thiết đến **session callback**
      - `session`: được gọi khi request tới **/api/auth/session** hoặc các function **getSession()**, **useSession()** được gọi. Dùng để expose các giá trị mà mình muốn từ token cho _client_ - "By default, **only a subset of the token is returned for increased security**". Chúng ta sẽ expose `access_token` để phía _client_ có thể sử dụng.

- Tiếp theo chúng ta sẽ thêm vào biến **NEXTAUTH_SECRET** trong file **.env** để config **JWT** như đã nói ở trên.

  ```shell:.env
  ...
  NEXTAUTH_SECRET=my-secret
  ```

- Cuối cùng chúng ta tạo _Layout component_ để hiển thị trạng thái đăng nhập của người dùng:

  ```jsx:src/views/shared/AppLayout.tsx
  import * as React from 'react';
  import AppBar from '@mui/material/AppBar';
  import Box from '@mui/material/Box';
  import Toolbar from '@mui/material/Toolbar';
  import Container from '@mui/material/Container';
  import Button from '@mui/material/Button';
  import Link from 'next/link';
  import { useTranslation } from 'next-i18next';
  import { signIn, signOut, useSession } from 'next-auth/react';

  type Props = {
    children: React.ReactElement;
  };

  const AppLayout = ({ children }: Props) => {
    const { data: session } = useSession();
    console.log({session});
    const { t } = useTranslation();
    const navItems = [
      { name: t('Home'), path: '/' },
      { name: t('Users'), path: '/users' },
    ];

    return (
      <>
        <AppBar position="fixed">
          <Container maxWidth="xl">
            <Toolbar disableGutters>
              <Box sx={{ flexGrow: 1, display: { xs: 'none', md: 'flex' } }}>
                {navItems.map(({ name, path }) => (
                  <Link key={path} href={path} passHref>
                    <Button sx={{ my: 2, color: 'white', display: 'block' }}>
                      {name}
                    </Button>
                  </Link>
                ))}
              </Box>
              <Box sx={{ flexGrow: 0, color: 'white' }}>
                {session ? (
                  <Box
                    sx={{
                      display: 'flex',
                      justifyContent: 'center',
                      alignItems: 'center',
                    }}>
                    <Box>{`Hi, ${session.user?.name}`}</Box>
                    <Button color="inherit" onClick={() => signOut()}>
                      {t('Sign Out')}
                    </Button>
                  </Box>
                ) : (
                  <Button color="inherit" onClick={() => signIn()}>
                    {t('Sign In')}
                  </Button>
                )}
              </Box>
            </Toolbar>
          </Container>
        </AppBar>
        <Box component="main" sx={{ p: 3 }}>
          <Toolbar />
          {children}
        </Box>
      </>
    );
  };
  export default AppLayout;
  ```

- Giải thích:
  - Chúng ta sẽ sử dụng `useSession` để kiểm tra người dùng đã đăng nhập chưa.
  - Nếu người dùng chưa đăng nhập thì sẽ hiển thị nút **Sign In**, nếu đã đăng nhập rồi thì sẽ hiển thị nút **Sign Out**. Ở đây mình dùng funtion `signIn` không tham số để sử dụng trang đăng nhập của **NextAuth.js**.

> Nếu các bạn tự tạo trang login riêng thì truyền data người dùng đã nhập vào tham số của `signIn` như hình dưới và chỉnh lại custom login page ở option **page**

> ![image.png](https://images.viblo.asia/c8e78efa-bd04-471a-bda3-ca7bd638b92d.png)

- Thêm _Layout component_ vừa tạo vào file **\_app.tsx** để hiển thị cho tất cả các page.

  ```jsx:_app.tsx
  import AppLayout from 'src/views/shared/AppLayout';
  ...
  function App({...}: ExtendedAppProps) {
    return (
      <CacheProvider value={emotionCache}>
        <ThemeProvider theme={lightTheme}>
          <CssBaseline />
          <SessionProvider session={session}>
            <AppLayout> // <=== Thêm vào đây
              <Component {...pageProps} />
            </AppLayout>
          </SessionProvider>
        ...
  ```

- Quay lại http://localhost:3000/users để xem kết quả.
  ![image.png](https://images.viblo.asia/066d701f-e314-40df-8a3f-b3ed12501e58.png)
- Do khi nảy chúng ta đã protect page với middeware nên nếu chưa đăng nhập sẽ tự động được chuyển đến trang đăng nhập. Hiện đang enable mock data nên có thể đăng nhập với tài khoản bất kì. Sau khi đăng nhập thành công thì đã có thể truy cập vào trang users

![image.png](https://images.viblo.asia/0bacefbc-7872-420c-9ce4-b5d0f754c6e2.png)

- Mở **developer tool** để kiểm tra thông tin trong session, có thể thấy trong session có chứa `access_token` và `name` mà chúng ta đã chỉ định ở file **[...next-auth].ts**

![image.png](https://images.viblo.asia/c8da0cc1-d7e0-4d5c-83bb-ec8569e5bf9b.png)

- Thử lại với `NEXT_PUBLIC_ENABLE_API_MOCKUP=0` để gọi đến API thực, có thể thấy chúng ta đã lấy được các giá trị cần thiết
  ![image.png](https://images.viblo.asia/2a9b062c-54e4-45d4-841d-d44d4d557bbf.png)

Vậy là chúng ta đã tích hợp xong xác thực cơ bản với **NextAuth.js**, tiếp theo chúng ta sẽ tích hợp xác thực **OAuth** với **Github** để hoàn thiện chức năng đăng nhập của ứng dụng.

## 2.6 Cài đặt login với github

Để đăng nhập với **Github** chúng ta cần **Client ID** và **Client secret**, chúng ta sẽ tạo thông qua các bước sau:

- Truy cập trang GitHub application: https://github.com/settings/applications/new.
- Nhập thông tin xác thực ứng dụng và nhấn **Register application**. Lưu ý ở mục **Authorization callback URL** các bạn nhớ để như mình, đó là url dùng để gọi callback **NextAuth.js**.

![](https://images.viblo.asia/0d93e506-1461-4578-92cf-74b156332d26.png)

- Nhấp vào **Generate a new client token** để nhận **Client secret**
  ![](https://images.viblo.asia/9edb5257-7e2e-449a-be83-f9dd5e60f893.png)
- Copy **Client ID** và **Client secret** và cho vào file **.env**

```.env
...
GITHUB_ID=your_client_ID
GITHUB_SECRET=your_client_secret
```

Sau khi đã tạo thành công chúng ta sẽ tiến hành tích hợp với **NextAuth.js**, các bước tiến hành lần lượt là:

- Thêm `GithubProvider` vào providers của chúng ta trong file **[...next-auth].ts**

  ```ts:[...next-auth].ts
  ...
  import GithubProvider from 'next-auth/providers/github';

  const providers: Provider[] = [
    CredentialsProvider({...}),
    GithubProvider({
      clientId: process.env.GITHUB_ID as string,
      clientSecret: process.env.GITHUB_SECRET as string,
    }),
    // ...add more providers here
  ];
  ```

- Sau khi đăng nhập với **Github** thành công từ **NextAuth.js** mình sẽ cho gọi đến API login với **Github** ở server để xác thực và gernerate `access_token` riêng của server. Để làm được việc đó mình sẽ chỉnh sửa lại ở `jwt` callback:

  ```ts:[...next-auth].ts
  ...
  export const authOptions: NextAuthOptions = {
    providers,
    callbacks: {
      async jwt({ token, user, account }) {
        if (user) {
          token.access_token = user.access_token;
          token.refresh_token = user.refresh_token;
        }
        if (account) {
          if (account.provider === 'github') {
            let res;
            if (AppConfig.enableApiMockup) {
              res = {
                status: 200,
                data: {
                  access_token: 'mock_access_token',
                  refresh_token: 'mock_refresh_token'
                },
              };
            } else {
              res = await axios.post(
                `${AppConfig.apiBase}/github-authentication`,
                {
                  access_token: account.access_token,
                  token_type: account.token_type,
                }
              );
            }
            token.access_token = res.data.access_token;
            token.refres_token = res.data.refresh_token;
          }
        }
        return token;
      },
      ...
  ```

- Giải thích:
  - Chúng ta tiếp tục dùng logic với biến chuyển giữa mock data và API.
  - Khi đăng nhập bằng **OAuth** với các provider khác nhau thì chúng ta sẽ gọi tới API tương ứng ở server, thông tin gửi đi sẽ là `access_token` của **Github** để phía server có thể xác thực lại với **Github** tránh trường hợp các token không hợp lệ mà hacker dùng.
  - Sau khi đã có token từ phía server chúng ta sẽ gắn vô token để truyền vào **session**.
- Quay lại http://localhost:3000/users tiến hành bấm vào **Sign In** để xem kết quả ( bạn nào chưa log out thì bấm vào **Sign Out** nha ).

![image.png](https://images.viblo.asia/64cd1d85-92f6-4d2a-9d81-9667ea52a5bb.png)

- **NextAuth.js** đã tự tạo cho chúng ta Button **Sign in with GitHub**, các bạn có thể bấm vào đăng nhập với tài khoản **Github** để xem kết quả.

![image.png](https://images.viblo.asia/9ec39adf-af1c-4dda-8fdd-e557317c31d2.png)

Có thể thấy username **Github** của mình đã xuất hiện, vậy là chúng ta đã tích hợp đăng nhập với **Github** thành công chỉ với vài bước đơn giản, dễ hơn nhiều so với chúng ta tự tạo.

## 2.7 Tự động gắn access token vào axios

Chúng ta đã có được thông tin _access_token_ từ **session**, để tự động thêm token vào request chúng ta sử dụng _interceptors_. Tiến hành chỉnh sửa file **axios.config.ts**

    ```ts:src/configs/axios.config.ts
    ...
    import { getSession } from 'next-auth/react';
    ...
    axios.interceptors.request.use(
      async (config) => {
        // Get session from NextAuth
        const session = await getSession();
        if (session?.access_token) {
          config.headers.Authorization = `Bearer ${session?.access_token}`;
        }
        return config;
    ...
    ```

Chuyển `NEXT_PUBLIC_ENABLE_API_MOCKUP=0` để thử gọi API và xem kết quả

![image.png](https://images.viblo.asia/24d9b95b-59a0-4d9c-8e23-23dd5ae843f2.png)

Theo như hình trên thì token đã được thêm vào request gửi đến server ( chỗ mình highlight ), từ giờ về sau các bạn sẽ không cần phải mất thời gian thêm token vào mỗi khi gọi request đến nữa.

## 2.8 Tự động refresh access token khi hết hạn

Chúng ta có thể triển khai theo 2 hướng khác nhau:

- Đặt logic refresh token trong **axios interceptors** để kiểm tra khi request có lỗi **401** sẽ dùng refresh token để lấy access token mới. Cách này sẽ ngắn gọn và đơn giản phù hợp với các dự án không cần UX quá cao vì khi access token hết hạn mới tiến hành renew.
- Đặt logic trong callback **jwt** của **NextAuth.js** và sử dụng **refreshInterval** để lấy token mới trước khi access token hết hạn. Cách triển khai sẽ phức tạp hơn nhưng giúp UX mượt mà hơn vì chúng ta sẽ cho renew access token trước khi nó hết hạn 30 phút ( hoặc thời gian bất kỳ ).

Mỗi cách triển khai sẽ phù hợp tùy theo yêu cầu dự án, mình sẽ triển khai cách 2 vì thấy nó khá hay và tích hợp với **NextAuth.js** ( hiện tại team dev của **NextAuth.js** đang phát triển tính năng này, có thể trong thời gian tới khi hoàn thiện sẽ ngắn gọn hơn cách chúng ta đang làm các bạn có thể theo dõi [ở đây](https://authjs.dev/guides/basics/refresh-token-rotation) ). Cách 1 các bạn có thể tự mình triển khai nếu có gì không hiểu có thể comment, mình sẽ hỗ trợ.

- Đầu tiên để kiểm tra `access_token` đã hết hạn hay chưa, chúng ta sẽ decode token ra để lấy thông tin, tiến hành cài thêm package

  `npm i jwt-decode`

> Nếu API login của các bạn có bao gồm thời gian hết hạn của `access_token` thì có thể bỏ qua bước decode này

- Tiếp theo chúng ta sẽ tạo function với chức năng renew `access_token` mới bằng `refresh_token`, giả sử http://localhost:3336/auth/refresh là API local chúng ta dùng để renew token, input là `refresh_token` và output là `access_token` và `refresh_token` mới.

  ```javascript:[...next-auth].ts
  import jwtDecode from 'jwt-decode';
  ...
  async function refreshAccessToken(token: any) {
    try {
      // Get a new set of tokens with a refreshToken
      const token_response = await axios.post(
        'http://localhost:3336/auth/refresh',
        {},
        {
          headers: {
            Authorization: `Bearer ${token.refresh_token}`,
          },
        }
      );

      return {
        ...token,
        access_token: token_response.data.access_token,
        access_token_expiry:
          token_response.data.access_token_expiry ||
          jwtDecode<{ exp: number }>(token_response.data.access_token).exp,
        refresh_token: token_response.data.refresh_token,
      };
    } catch (error) {
      return {
        ...token,
        error: 'RefreshAccessTokenError',
      };
    }
  ...
  ```

- Giải thích:

  - Nếu như API login của các bạn có thông tin hết hạn của `access_token` thì sẽ lấy thời gian đó, còn không sẽ decode `access_token` để lấy.
  - Trả về lỗi `RefreshAccessTokenError` nếu như `refresh_token` hết hạn hoặc lỗi phía API để chúng ta catch và log out user.

- Sau khi đã cài đặt chúng ta sẽ thêm vào logic kiểm tra `access_token` đã hết hạn hay chưa ở dưới cùng của **jwt callback**:

  ```javascript:[...next-auth].ts
  ...
  export const authOptions: NextAuthOptions = {
    // Configure one or more authentication providers
    providers,
    callbacks: {
      async jwt({ token, user, account }) {
        ...
        const decode_token: { exp: number; iat: number } = jwtDecode(
          token.access_token
        );
        const shouldRefreshTime = Math.round(
          decode_token.exp * 1000 - 30 * 60 * 1000 - Date.now()
        );
        // If the token is still valid, just return it.
        if (shouldRefreshTime > 1000) {
          return token;
        }

        return await refreshAccessToken(token);;
      },
  ```

- Giải thích:
  - `jwtDecode` sẽ trả về cho chúng ta field `exp`, đó chính là thông tin thời gian hết hạn của token.
  - `30 * 60 * 1000` chúng ta sẽ cho renew `access_token` nếu thời gian hết hạn của nó còn dưới 30p.
  - `if (shouldRefreshTime > 1000)` trong quá trình test với `if (shouldRefreshTime > 0)` mình thấy thỉnh thoảng có sai số nên làm chức năng hoạt động sai, sau nhiều lần test điều chỉnh về giá trị `1000` thì mình thấy căn bản đã giải quyết được sai số.
- Để có thể trigger được **jwt callback** chúng ta cần gọi đến **session**, vì mỗi khi **session** được gọi **NextAuth.js** sẽ **decode token** từ **cookie** và việc đó sẽ gọi đến **jwt callback**. Để làm được việc đó mình sẽ thêm vào **option refetchInterval** trong file **\_app.tsx**, option này có chức năng gọi lại **session** sau khoảng thời gian mà chúng ta chỉ định.
  ```jsx:_app.tsx
  import { useState } from 'react';
  ...
  function App({
    Component,
    emotionCache = clientSideEmotionCache,
    pageProps: { session, ...pageProps },
  }: ExtendedAppProps) {
    const [interval, setInterval] = useState(0);
      ...
          <SessionProvider session={session} refetchInterval={interval}>
            <AppLayout>
              <Component {...pageProps} />
            </AppLayout>
          </SessionProvider>
          ...
  ```
- Tiếp theo chúng ta cần kiểm soát được thời gian khi nào cần gọi **session** vì nếu như để thời gian cố định ( ví dụ interval là 5 giây ) thì sẽ làm cho nó gọi liên tục, nếu chúng ta vừa renew token xong mà vẫn gọi liên tục thì không hay. Chúng ta cần 1 giải pháp để xác định được thời gian khi nào `access_token` gần hết hạn mới gọi đến **session**. Để giải quyết mình sẽ tạo 1 component riêng để xử lí:

  ```jsx:src/views/shared/RefreshTokenHandler.tsx
  import { useSession } from 'next-auth/react';
  import { useEffect } from 'react';

  type Props = {
    setInterval: Function;
  };

  const RefreshTokenHandler = (props: Props) => {
    const { data: session } = useSession();

    useEffect(() => {
      if (!!session) {
        const timeRemaining = Math.round(
          (session.access_token_expiry * 1000 - 30 * 60 * 1000 - Date.now()) /
            1000
        );
        props.setInterval(timeRemaining > 0 ? timeRemaining : 0);
      }
    }, [session]);

    return null;
  };

  export default RefreshTokenHandler;

  ```

- Giải thích:

  - Mình sẽ dùng `useEffect` để kiểm tra theo sự thay đổi của **session**. Nếu như có sự thay đổi chúng ta sẽ cho tính lại thời gian truyền vào **refetchInterval**.
  - `30 * 60 * 1000`chúng ta cũng sẽ lùi về 30 phút trước khi token hết hạn.

- Cập nhật lại file **\_app.tsx** thêm vào component vừa tạo:

  ```jsx:_app.tsx
  import RefreshTokenHandler from '@/views/shared/RefreshTokenHandler';
  ...
  function App({
    Component,
    emotionCache = clientSideEmotionCache,
    pageProps: { session, ...pageProps },
  }: ExtendedAppProps) {
    const [interval, setInterval] = useState(0);
      ...
          <SessionProvider session={session} refetchInterval={interval}>
            <AppLayout>
              <Component {...pageProps} />
            </AppLayout>
            <RefreshTokenHandler setInterval={setInterval} /> // <= Thêm vào đây
          </SessionProvider>
          ...
  ```

- Mọi thứ gần như đã hoàn thiện, chúng ta đã có thời gian dùng để gọi renew `access_token`, bước cuối cùng là sẽ **log out** user nếu gọi renew token thất bại. Mình sẽ tạo 1 hook **useAuth** để kiểm soát vấn đề này:

  ```javascript:src/hooks/auth/useAuth.ts
  import { protect_route } from '@/configs/auth.config';
  import { signOut, useSession } from 'next-auth/react';
  import { useRouter } from 'next/router';
  import { useEffect, useState } from 'react';

  export default function useAuth(shouldRedirect: boolean) {
    const { data: session } = useSession();
    const router = useRouter();
    const [isAuthenticated, setIsAuthenticated] = useState(false);

    useEffect(() => {
      if (session?.error === 'RefreshAccessTokenError') {
        signOut({
          callbackUrl: `/api/auth/signin?callbackUrl=${router.asPath}`,
          redirect: protect_route.includes(router.asPath)
            ? shouldRedirect
            : false,
        });
      }

      if (session !== undefined || session !== null) {
        if (router.route === '/api/auth/signin') {
          router.replace('/');
        }
        setIsAuthenticated(true);
      }
    }, [session]);

    return isAuthenticated;
  }
  ```

- Cuối cùng thêm _hook_ vừa tạo vào file **AppLayout.tsx**:

  ```jsx:AppLayout.tsx
  import useAuth from '@/hooks/auth/useAuth';
  ...
  const AppLayout = ({ children }: Props) => {
    const isAuthenticated = useAuth(true);
    ...
        <Box component="main" sx={{ p: 3 }}>
          <Toolbar />
          {isAuthenticated && children}
        </Box>
  ...
  ```

Mọi thứ đã hoàn thiện, các bạn có thể thử set thời gian hết hạn của `access_token` thấp lại để kiểm tra kết quả. Ở dưới mình cho thời gian hết hạn là 5s và `refresh_token` hết hạn. Thì sau khi `access_token` hết hạn sẽ tự động redirect sang trang login.

![](https://images.viblo.asia/170038cd-676b-4a0a-9349-f082a8f7bb55.gif)

# Kết luận

Trong bài viết này, chúng ta đã tìm hiểu cách sử dụng **next-i18next** và **i18next-scanner** để hỗ trợ đa ngôn ngữ cho ứng dụng **NextJS**. Bằng cách sử dụng các thư viện này, các bạn có thể tạo ra một trang web đa ngôn ngữ một cách dễ dàng và thuận tiện.

Bên cạnh đó, chúng ta cũng đã sử dụng **NextAuth.js** để cho phép người dùng đăng nhập bằng tài khoản của họ hoặc thông qua **Github**. Điều này giúp cho ứng dụng trở nên dễ dàng tiếp cận và thu hút nhiều người dùng hơn.

Cuối cùng, chúng ta đã tìm hiểu cách sử dụng **refresh token** để đảm bảo rằng người dùng có thể tiếp tục truy cập vào ứng dụng mà không cần phải đăng nhập lại nhiều lần. Điều này giúp tăng trải nghiệm người dùng và giúp giảm thiểu việc người dùng bị gián đoạn trong quá trình sử dụng ứng dụng của chúng ta.

Tóm lại, với các kỹ thuật và thư viện đã đề cập trong bài viết, các bạn có thể tạo ra một ứng dụng **NextJS** đa ngôn ngữ, cho phép đăng nhập bằng nhiều phương thức và cung cấp trải nghiệm người dùng tốt hơn. Hy vọng bài viết đã giúp ích cho các bạn trong quá trình phát triển ứng dụng của mình.

Nếu các bạn có thắc mắc gì có thể comment ở đây hoặc inbox riêng cho mình. Cảm ơn các bạn đã giành thời gian theo dõi, hẹn gặp lại các bạn ở các bài viết tiếp theo

# Tài liệu tham khảo

- https://nextjs.org/docs/advanced-features/i18n-routing#limits-for-the-i18n-config
- https://viblo.asia/p/nextjs-da-ngon-ngu-khong-can-thu-vien-ngoai-Eb85oe0OZ2G
- https://dev.to/mabaranowski/nextjs-authentication-jwt-refresh-token-rotation-with-nextauthjs-5696 -->
