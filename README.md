# eps

路由：/admin/base/open/eps

功能实现：@cool-midway/core/rest/eps.js

1. 使用 Midway 内部提供的装饰器 api **listModule** 获取所有 entity，继而再用 orm api 获取数据库表的 columns。
2. 使用 **listModule** 获取所有 controller，**getClassMetadata** 获取 class 元信息。
3. 使用 Midway 路由表服务获取所有路由，并按照 prefix 分组。
4. 按照路由前缀 "/admin/" 开头与否，分别进行收集，最后按照 module 分组。

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



