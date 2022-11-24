# eps

路由：/admin/base/open/eps

功能实现：@cool-midway/core/rest/eps.js

1. 使用 Midway 内部提供的装饰器 api **listModule** 获取所有 entity，继而再用 orm api 获取数据库表的 columns。
2. 使用 **listModule** 获取所有 controller，**getClassMetadata** 获取 class 元信息。
3. 使用 Midway 路由表服务获取所有路由，并按照 prefix 分组。
4. 按照 controller 逐个添加eps，路由前缀 "/admin/" 开头与否，分别进行收集，最后按照 module 分组。

```typescript
{
  module, // 当前模块名
  api: routers[prefix], // 按照 prefix 获取路由
  name, // curdOption entity 名字 
  columns: entitys[name] || [], // entity columns
  prefix, // 前缀
}
```

> cool admin 提供的 controller decorator 会在包装自定义控制器时使用 Midway 提供的 **saveClassMetadata** 保存以下元数据
> 1. prefix `// 路由前缀`
> 2. routerOptions `// 路由配置，middleware，sensitive`
> 3. target `// 装饰的类`
> 4. curdOption `// CRUD配置`
> 5. module `// 当前模块`

> 路由前缀自动生成规则为：/ controller 文件夹下的文件夹名或者文件名 / 模块文件夹名 / 方法名

# 登录

路由：/admin/base/open/login

功能实现：/src/modules/base/service/sys/login.ts

## 验证码

1. 使用 svg-captcha 生成验证码文本和 svg 字符串。
2. 使用 uuid 生成唯一标识符 captchaId。
3. 将 captchaId，验证码文本存入缓存，保留半小时。
4. 将加工好的验证码 svg 字符串和 captchaId 返回前端。

```typescript
async captcha(...){
  ...
  // 返回 captchaId 和 svg
  const result = {
    captchaId: uuid(),
    data: svg.data.replace(/"/g, "'"),
  };
  ...
  // 缓存 captchaId，验证码文本
  await this.cacheManager.set(
    `verify:img:${result.captchaId}`,
    svg.text.toLowerCase(),
    { ttl: 1800 }
  );
  ...
}
```

## jwt

## 登入

1. 使用 Midway validator 组件按照 LoginDTO 进行参数校验。
2. 校验captcha。
3. 

## 登出