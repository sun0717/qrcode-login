# app/web 扫码登录
登录之后，app会进入确认界面，可以选择授权登录或者取消。
这边确认之后，pc 网站就登录了。
原理：二维码是个 url ，用手机浏览器打开，是下载app页面。在 APP 里打开，就是登录确认界面。

- 状态
二维码这里是有个唯一 id 的，通过这个 id 就知道是哪个二维码。
二维码有 5 个状态：
  - 未扫描  （最开始）
  - 已扫描，等待用户确认  （扫码后进入等待用户确认状态：扫码成功 请用手机授权登录）
  - 已扫描，用户同意授权  （登录成功）
  - 已扫描，用户取消授权  （进入取消授权状态）
  - 已过期               （长时间不操作进入过期状态）  
如何知道二维码状态改变？：
  每秒一次的轮询请求

服务端：
  服务端有个 qrcode/generate 接口，会生成一个随机的二维码 id ，存到 redis 里，并返回二维码。
  还有个 qrcode/check 接口，会返回 redis 里的二维码状态，浏览器里可以轮询这个接口拿到二维码状态。
APP：
  手机 APP 扫码，如果没登录，会先跳转到登录页面，登录之后会进入登录确认页面。
  这时候用户是登录的，jwt 的登录认证会携带 token ，服务端只要从 token 中取出用户信息，存入 redis 即可。
  另一边的轮询接口发现是确认状态，根据用户信息生成 jwt 返回。
  手机 APP 确认之后， pc 的浏览器就自动登录了该用户账号。
  jwt 是保存登录状态的一种方案，会把用户信息放在 token 里返回，然后每次访问接口带上 authorization 的 header, 携带 token

- 实战
生成二维码
```typescript
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('qrcode/generate')
  async generate() {
    // node 的 crypto 模块生成一个随机的 uuid 
    const uuid = randomUUID();
    // 判断是 APP 打开的还是其他方式打开的，分别会显示不同的内容
    const dataUrl = await qrcode.toDataURL(`http://localhost:3000/pages/confirm.html?id=${uuid}`);
    return {
      qrcode_id: uuid,
      img: dataUrl
    }
  }
}
```

手机端借助charles，在一个局域网下，代理 ip 是电脑 ip ，端口号是 charles 代理服务的默认端口 8888 

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';
import { randomUUID } from 'crypto';
import * as qrcode from 'qrcode';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('qrcode/generate')
  async generate() {
    // node 的 crypto 模块生成一个随机的 uuid 
    const uuid = randomUUID();
    // 判断是 APP 打开的还是其他方式打开的，分别会显示不同的内容
    const dataUrl = await qrcode.toDataURL(`http://172.16.111.12:3000/pages/confirm.html?id=${uuid}`);
    return {
      qrcode_id: uuid,
      img: dataUrl
    }
  }
}


```

web端扫码登录页面 index.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>扫码登录</title>
    <script src="https://unpkg.com/axios@1.5.0/dist/axios.min.js"></script>
</head>

<body>
    <img id="img" src="" alt="">
    <script>
        axios.get('http://localhost:3000/qrcode/generate').then(res => {
            document.getElementById('img').src = res.data.img;
        })
    </script>
</body>

</html>
```

实现其他接口

![image-20231207210447979](https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207210447979.png)

生成二维码之后，在 redis 里保存一份，这里用 map 保存
```typescript
const map = new Map<string, QrCodeInfo>();

interface QrCodeInfo {
  status: 'noscan' | 'scan-wait-confirm' | 'scan-confirm' | 'scan-cancel' | 'expired',
  userInfo?: {
    userId: number;
  }
}

// noscan 未扫描
// scan-wait-confirm 已扫描，等待用户确认
// scan-confirm 已扫描，用户同意授权
// scan-cancel 已扫描，用户取消授权
// expired 已过期
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('qrcode/generate')
  async generate() {
    // node 的 crypto 模块生成一个随机的 uuid 
    const uuid = randomUUID();
    // 判断是 APP 打开的还是其他方式打开的，分别会显示不同的内容
    const dataUrl = await qrcode.toDataURL(`http://172.16.111.12:3000/pages/confirm.html?id=${uuid}`);

    map.set(`qrcode_${uuid}`, {
      status: 'noscan'
    })

    return {
      qrcode_id: uuid,
      img: dataUrl
    }
  }

  @Get('qrcode/check')
  async check(@Query('id') id: string) {
    return map.get(`qrcode_${id}`);
  }

}

```

访问 /qrcode/check 拿到 id 的状态

```typescript
@Get('qrcode/check')
async check(@Query('id') id: string) {
    return map.get(`qrcode_${id}`);
}

@Get('qrcode/scan')
async scan(@Query('id') id: string) {
  const info = map.get(`qrcode_${id}`);
  if (!info) {
    throw new BadRequestException('二维码已过期');
  }
  info.status = 'scan-wait-confirm';
  return 'success';
}

@Get('qrcode/confirm')
async confirm(@Query('id') id: string) {
  const info = map.get(`qrcode_${id}`);
  if (!info) {
      throw new BadRequestException('二维码已过期');
  }
  info.status = 'scan-confirm';
  return 'success';
}

@Get('qrcode/cancel')
async confirm(@Query('id') id: string) {
  const info = map.get(`qrcode_${id}`);
  if (!info) {
      throw new BadRequestException('二维码已过期');
  }
  info.status = 'scan-cancel';
  return 'success';
}
````

- 扫码之后，pc上的二维码同步改变状态的原理

## 登录功能
确认之后，就要拿到这边的登录状态，从中取出用户信息。
引入 jwt 的包。

AppModule 里引入：
```typescript

@Module({
  imports: [
    JwtModule.register({
      secret: 'sun'
    })
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
postman测登录接口
```typescript
@Inject(JwtService)
private jwtService: JwtService;

private users = [
    {id: 1, username: 'dong', password: '111'},
    {id: 2, username: 'guang', password: '222'},
];

@Get('login')
async login(@Query('username') username: string, @Query('password') password: string) {

    const user = this.users.find(item => item.username === username);

    if(!user) {
      throw new UnauthorizedException('用户不存在');
    }
    if(user.password !== password) {
      throw new UnauthorizedException('密码错误');
    }

    return {
      token: await this.jwtService.sign({
      userId: user.id
    })
}

作者：zxg_神说要有光
链接：https://juejin.cn/post/7276266802824511540
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```
返回一串token：
添加一个 userInfo 的接口拿到用户信息：

```typescript
@Get('userInfo')
async userInfo(@Headers('Authorization') auth: string) {
  try {
    const [, token] = auth.split(' ');
    const info = await this.jwtService.verify(token);

    const user = this.users.find(item => item.id == info.userId);
    return user;
  } catch(e) {
    throw new UnauthorizedException('token 过期，请重新登录');
  }
}
```
从 header 中去除 token ，解析出其中的信息，从而拿到 userId , 然后查询 id 对应的用户信息返回。
加上 authorization 的 header
```javascript
authorization     Bearer  xxxxxxxx
```

在登录确认界面 添加按钮
```html

<button id="sun">登录 sun 账号</button>
<button id="guang">登录 guang 账号</button>
```
登录不同的账号，拿到 token :
```typescript
document.getElementById('confirm').addEventListener('click', () => {
    // axios.get('http://172.16.111.12:3000/qrcode/confirm?id=' + id).catch(e => {
    //     alert('二维码已过期');
    // });
    axios.get('http://172.16.111.12:3000/qrcode/confirm?id=' + id, {
        headers: {
            authorization: 'Bearer ' + token
        }
    }).catch(e => {
        alert('二维码已过期');
    })
});
```

具体实现还是：：：：
>
> 作者：zxg_神说要有光
> 链接：https://juejin.cn/post/7276266802824511540
> 来源：稀土掘金
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。