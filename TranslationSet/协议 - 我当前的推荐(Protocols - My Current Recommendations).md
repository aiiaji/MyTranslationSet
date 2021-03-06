# 协议 - 我当前的推荐

原文链接：[Protocols - My Current Recommendations](http://owensd.io/2015/08/06/protocols.html?utm_campaign=Swift%2BSandbox&utm_medium=web&utm_source=Swift_Sandbox_2)

最近 Swift 的热点都围绕在协议上。他们觉得任何东西都应该是协议。理论上这挺好，但是事实上，这种观点会产生一些不好的副作用。

<!--more-->

我在代码中使用协议时，总是牢记下面两条规则：

## 1.不要把协议当成类型

我看到的（和我一开始写的）许多方法都是在继承关系里把协议当成一个基类。我不认为这是协议的正确用法，这种设计模式仍然停留在“面向对象”的思维方式上。  

换句话说，假如你的协议只在继承关系里有意义，那就应该扪心自问是否真的有必要使用协议。我不赞同这种说法：“我的类型是结构体，所以我需要用协议来代替。”如果真的需要这样，那就重构它，把它变得更通用。  

进一步的验证可见：[http://swiftdoc.org/swift-2/](http://swiftdoc.org/swift-2/)。需要关注所有那些协议（不以 `_` 开头的协议）吗？所有的协议都能适用不同的类型，不管是不是类型继承。  

## 2.不要把协议泛型化，除非你必须这么做！

这并不是一个小问题，一旦你把协议泛型化，就无法再使用包含不同协议实例的类型集合。我认为这是严重的设计缺陷。比如说，无论在哪里使用协议，协议中所有不受 `Self` 限制的功能都应该保证调用安全。  

这条规则同样适用于遵守泛型协议的协议，比如 `Equatable` 。泛型会传染。  

下面这行代码：  
    
```swift
protocol Foo : Equatable {} 
```

就是典型的例子。  

来个实际点的例子吧：  

我们想要对 HTTP response 建模，并且想要支持两种不同的 response 类型：String 和 JSON。  

你可能会这样写：  

```swift
class HTTPResponse<ResponseType> {
    var response: ResponseType
    init(response: ResponseType) { self.response = response }
}
```

我认为这样写不好。一旦这样写，我们就人为地限制了使用这个类型的能力；比如，这不能在复杂情况下的集合里使用。现在，为什么我想要在集合里使用不同的 `ResponseType`？假设，我想要构建一个 response/request 的测试工具。返回 response 的集合里需要有我支持的类型：String 和 JSON。  

使用 `AnyObject` 是个不错的做法。虽然可行，但是真的很操蛋。  

另一种方法是使用协议。然而，除了构建 `ResponseType` 协议之外，让我们想想我们真正想要什么。我真正关心的是， `HTTPResponse` 接收到的 `ResponseType` 都能表示成 `String` 。  

揣着这样的想法，我写出这样的代码：

```swift
protocol StringRepresentable {
    var stringRepresentation: String { get }
}

class HTTPResponse {
    var response: StringRepresentable
    init(response: StringRepresentable) { self.response = response }
}
```

对我而言，这为 API 使用者提供了极大的方便，也维持了类型的明确性。

当然，这样做是有弊端的。假如你真的需要为 response 使用特定的类型，那么就需要进行类型转换。

```
class JSONResponse : StringRepresentable {
    var stringRepresentation: String = "{}"
}

let http = HTTPResponse(response: JSONResponse())
let json = http.response as? JSONResponse
```

>*这个方法明显更好。调用者知道可能的返回类型或返回值。这明显与遍历整个集合取出返回值是不同的，因为，代码的使用者可能使用的是其他的返回类型，比如 `XMLResponse` ，而我们的代码不可能知道这个。*

最好是这样写：  

```swift
class HTTPResponse<ResponseType : StringRepresentable> {
    var response: ResponseType
}

let responses = [json, string]  // responses 变量是以 HTTPResponse 为元素的数组，元素中的 ResponseType 是未指定的
```

如果要用集合，你还是需要强制转换 response 的类型，不过现在你可以直接用 `json` 实例来验证类型。

反正我是每次都用 `[HTTPResponse]` 类型的集合，而不是 `[AnyObject]` 。

