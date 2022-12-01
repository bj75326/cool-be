# Midway ioc（简略）

以下统一 Provider包装的类简称**注入类**，需要被注入类简称**目标类**。

## @Provider

Provider包装器在注入类上添加 metaData，用于在服务启动时被扫描到容器。

```typescript
export const class_key = 'ioc:tagged_class'

export function Provider (identifier?: string, args?: Array<any>) {
  return function (target: any) {
    // 驼峰命名，这个的目标是，注解的时候退出不传，就用类名的驼峰式
    identifier = identifier ?? camelcase(target.name);

    Reflect.defineMetadata(class_key, // metaData 标记字符串
      { 
        id: identifier, // key，用来注册Ioc容器
        args: args || [] // 实例化所需参数
      }, 
      target
    );

    return target;
  }
}
```

Container 容器类提供绑定方法

```typescript
export class Container {
  bindMap = new Map()

  // 绑定注入类信息
  bind(identifier: string, registerClass: any, constructorArgs: any[]) {
    this.bindMap.set(identifier, {registerClass, constructorArgs})
  }
  ...
```

load在启动时扫描，将注入类扫描存储到container map (演示只用单层目录扫描演示)。

```typescript
import { class_key } from './provider';

export function load(container, path) {
  const list = fs.readdirSync(path)
  for (const file of list) {
    if (/\.ts$/.test(file)) {
      const exports = require(resolve(path, file))

      for (const m in exports) {
        const module = exports[m]
        if (typeof module === 'function') {
          const metadata = Reflect.getMetadata(class_key, module)
          // register
          if (metadata) {
            container.bind(metadata.id, module, metadata.args)
          }
        }
      }
    }
  }
}
```

## @Inject

Inject包装器将目标类所有依赖注入信息添加到目标类的 metaData 内，

```typescript
export const props_key = 'ioc:inject_props'

export function Inject () {
  return function (target: any, targetKey: string) {  // target 目标类对象，targetKey 注入的变量名

    const annotationTarget = target.constructor // annotationTarget 目标类
    let props = {}
    
    if (Reflect.hasOwnMetadata(props_key, annotationTarget)) {
      props = Reflect.getMetadata(props_key, annotationTarget)  // 目标类是否已有元数据记依赖注入变量
    }

    props[targetKey] = {
      value: targetKey
    }

    Reflect.defineMetadata(props_key, props, annotationTarget)  // 将目标类所有依赖注入变量信息添加到目标类的 metaData 内
  }
}
```

Container 容器类提供 get 方法，使目标类实例可以获取注入类实例。

```typescript
import { props_key } from './inject'

export class Container {
  ...
  
  get<T>(identifier: string): T { // 供目标类实例获取注入类实例
    const target = this.bindMap.get(identifier)
    if (target) {
      const { registerClass, constructorArgs } = target // bindMap 内拿到注入类

      // 等价于 const instance = new A([...constructorArgs]) // 假如 registerClass 为定义的类 A
      // 对象实例化的另一种形式，new 前面须要跟大写的类名，而上面的形式能够不必，能够把一个类赋值给一个变量，通过变量实例化类
      const instance = Reflect.construct(registerClass, constructorArgs) // 获取注入类实例

      const props = Reflect.getMetadata(props_key, registerClass) // 获取注入类的依赖注入变量元数据
      for (let prop in props) {
        const identifier = props[prop].value
        
        instance[prop] = this.get(identifier) // 递归让注入类实例先获得它所有依赖注入的对象
      }
      return instance // 返回注入类实例给目标类实例
    }
  }
}

```

## @Controller

## @Get 和 @Query


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

