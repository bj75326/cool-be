# Midway ioc（简略）

以下统一 Provider 包装的类简称**注入类**，需要被注入类简称**目标类**。

## @Provider

Provider 装饰器在注入类上添加 metaData，用于在服务启动时被扫描到容器。

```typescript
export const class_key = 'ioc:tagged_class'

export function Provider (identifier?: string, args?: Array<any>) {
  return function (target: any) {
    // 驼峰命名，这个的目的是，注解的时候退出不传，就用类名的驼峰式
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
}
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

Inject 装饰器将目标类所有依赖注入信息添加到目标类的 metaData 内，

```typescript
export const props_key = 'ioc:inject_props'

export function Inject () {
  return function (target: any, targetKey: string) {  // target 目标类，targetKey 注入的变量名

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

Controller 装饰器在被包装类上添加 metaData，其中包括前缀等，

```typescript
export const class_key = 'ioc:controller_class'

export function Controller (prefix = '/') {
  return function (target: any) {

    const props = {
      prefix
    }

    Reflect.defineMetadata(class_key, props, target)
    return target
  }
}
```

之后 load 在启动应用时

```typescript
import { class_key } from './provider';
import { class_key as controller_class_key } from './controller';
import { props_key, params_key } from './request';

const req_mthods_key = 'req_methods'
const joinSymbol = '_|_'

export function load(container, path, ctx) {
  // 
  const list = fs.readdirSync(path);

  for (const file of list) {
    if (/\.ts$/.test(file)) {
      const exports = require(resolve(path, file));

      for (const m in exports) {
        const module = exports[m]
        if (typeof module === 'function') {
          const metadata = Reflect.getMetadata(class_key, module);

          if (metadata) {
            container.bind(metadata.id, module, metadata.args)
            // 以上为容器扫描目录下 Provider 包装的所有类。

            // 之后收集路径，方法和它们各自对应的控制类，处理函数和参数，并放入容器内
            const controllerMetadata = Reflect.getMetadata(controller_class_key, module); 
            if (controllerMetadata) {
              const reqMethodMetadata = Reflect.getMetadata(props_key, module); // 获取控制器类上的方法元数据 [{method: 'GET', routerName, fn: targetKey}, ...] 
              
              if (reqMethodMetadata) {
                const methods = container.getReq(req_mthods_key) || {};  
                const reqMethodParamsMetadata = Reflect.getMetadata(params_key, module); // 获取控制器类上的参数元数据 {[targetKey] : [{type: 'query', index, paramName}, ...], ...}

                reqMethodMetadata.forEach(item => {
                  const path = controllerMetadata.prefix + item.routerName  // 补全路径
                  // 收集路径，方法及对应的控制器类，具体的处理函数，参数信息 
                  methods[item.method + joinSymbol + path] = {
                    id: metadata.id, // Controll 类
                    fn: item.fn, // Get 包装方法名
                    args: reqMethodParamsMetadata ? reqMethodParamsMetadata[item.fn] || [] : [] // Get 方法的参数信息 [{type: 'query', index, paramName}, ...]
                  }
                });

                container.bindReq(req_mthods_key, methods) // 放入容器内
              }
            }
          }
        }
      }
    }
  }

  const reqMethods = container.getReq(req_mthods_key);

  if (reqMethods) {
    // ctx.req.url /api/c?id=12
    const [urlPath, query] = ctx.req.url.split('?');
    // key: request 方法 + 路径
    const methodUrl = ctx.req.method + joinSymbol + urlPath;
    // 依据 key 取出对应数据
    const reqMethodData = reqMethods[methodUrl];
    if (reqMethodData) {
      const {id, fn, args} = reqMethodData;
      let fnQueryParams = [];
    
      if (args.length) {  // 处理方法需要参数
        const queryObj = queryParams(query) // 获取 query 字符串转化为对象
        // 这儿先依据参数在函数中的地位进行排序，这儿只解决了 Query 的状况， 再依据参数名从查问对象中取出数据
        fnQueryParams = args.sort((a, b) => a.index - b.index).filter(item => item.type === 'query').map(item => queryObj[item.paramName])
      }

      // 调用控制器类的具体处理函数，传入参数，生成响应
      const res = container.get(id)[fn](...fnQueryParams);
      ctx.res.end(JSON.stringify(res));
    }
  }
}
```

## @Get 和 @Query

Get 装饰器收集方法，路径，处理函数名，放入当前控制类的元数据内。

```typescript
export const props_key = 'ioc:request_method';

export function Get (path?: string) {
  return function (target: any, targetKey: string) {  // target 控制器类，targetKey 装饰器包装的函数名
    
    const annotationTarget = target.constructor // 控制器类

    let props = []
    
    if (Reflect.hasOwnMetadata(props_key, annotationTarget)) {
      props = Reflect.getMetadata(props_key, annotationTarget)  // 检查当前控制器类 metaData 是否已经有相关方法记录
    }

    const routerName = path ?? '';  // 路由路径

    props.push({  // [{method: 'GET', routerName, fn: targetKey}, ...] 
      method: 'GET',
      routerName,
      fn: targetKey
    }); 

    Reflect.defineMetadata(props_key, props, annotationTarget);
  }
}
```

Guery 装饰器

```typescript
export const params_key = 'ioc:request_method_params'

export function Query () {
  return function (target: any, targetKey: string, index: number) { // target 控制器类, targetKey 装饰器包装的函数名，index 装饰的函数第几个参数
    
    const annotationTarget = target.constructor;  // 控制器类

    const fn = target[targetKey]  // 装饰的函数
    
    const args = getParamNames(fn)  // 函数参数列表
    
    let paramName = '';  
    if (fn.length === args.length && index < fn.length) {
      paramName = args[index]; // 获取参数名
    }

    let props = {};
    
    if (Reflect.hasOwnMetadata(params_key, annotationTarget)) {
      props = Reflect.getMetadata(params_key, annotationTarget); // 检查当前控制器类 metaData 是否已经有相关参数记录
    }

    const paramNames = props[targetKey] || [];
    paramNames.push({type: 'query', index, paramName}); 

    props[targetKey] = paramNames;  // {[targetKey] : [{type: 'query', index, paramName}, ...], ...} 存放到控制器类的 metaData

    Reflect.defineMetadata(params_key, props, annotationTarget)
  }
}
```

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

cool admin 使用 token 和 refreshToken 维护用户登录状态。

token：无状态，时效2小时，
refreshToken：有状态，时效30天，

## 登入

1. 使用 Midway validator 组件按照 LoginDTO 进行参数校验。
2. 校验 captcha。
3. 校验用户账户密码。
4. 校验用户角色。
5. 生成 token，refreshToken。
6. 保存用户相关信息至缓存。
7. 返回 token，refreshToken 和各自有效时间。

## 登出

# CRUD

