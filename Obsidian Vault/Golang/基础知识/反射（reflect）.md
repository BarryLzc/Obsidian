```
type Sign interface {  
	model.CustomPlugin  
	//Handle 用于指定插件实现具体参数/返回值  
	Handle(context *model.Context, in *modelIn.EnterpriseLimitIn, globalIns map[model.Type]any) (out *modelIn.EnterpriseLimitOut, pluginGlobalOut any, fastFail bool, err error)  
}

// Map 发布平台依赖判断自定义插件实现类型 需要平台实现  
var Map map[model.PluginType]reflect.Type

Map = map[model.PluginType]reflect.Type{  
enum.Sign: reflect.TypeOf((*open.Sign)(nil)).Elem(),  
enum.GraphqlLimit: reflect.TypeOf((*open.GraphqlLimit)(nil)).Elem(), 

}
```
重点笔记：
1. golang不允许获取interface{}的反射，所以需要使用(*open.Sign)(nil)，初始化interface{}。
2. 使用reflect.TypeOf()，获取interface{}的空指针类型。
3. 使用reflect.TypeOf().Elem()，获取该空指针所指向类型的反射信息。