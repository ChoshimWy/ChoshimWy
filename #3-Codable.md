### Codable
#### 1.子类属性无法编码问题
* **属性没有继承自父类的Codable协议**: 确保子类的属性也声明为Codable,以便能够被编码为JSON.你可以在子类中添加Codable协议,并确保父类和子类的所有属性都是Codable的.
* **子类属性的访问级别限制**: 确保子类属性具有公共访问级别,以便JSONEncoder能够访问和编码它们.你可以将属性的访问级别设置为public或更高级别.
* **在子类中实现了CodingKeys枚举来指定子类属性的编码键**.

此外,我们还重写了`init(from:)`和`encode(to:)`方法，以便在解码和编码时正确处理子类属性.

创建一个`Personal`类
```Swift
class Personal: Codable {
     var name: String = "Xiao Ming"
     var age: Int = 21
}
```
继承`Personal`的子类
```Swift
class Employee: Personal {
    var uuid: String = UUID().uuidString
    
    
    enum CodingKeys: String, CodingKey {
        case uuid
    }

    
    override init() {
        super.init()
    }
    
    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.uuid = try container.decode(String.self, forKey: .uuid)
        try super.init(from: decoder)
    }
    
    override func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(uuid, forKey: .uuid)
        try super.encode(to: encoder)
    }
}
```
测试`Encoder`方法,如果子类没有重写'encode'方法 此处的encode只会编码父类属性
```Swift
func testEncoder() {
     let employee = Employee()
        do {
            /// 如果子类没有重写'encode'方法 此处的encode只会编码父类属性
            let data = try JSONEncoder().encode(employee)
            let json = try JSONSerialization.jsonObject(with: data)
            print(json)
        } catch {
        
        }
}
```
测试`Decoder`, 如果子类没有重写'init(from:)', 此处只会解码父类属性
```Swift
func testDecoder() {
            guard let data = """
{
    "name": "Huangshan",
    "age": 32,
"uuid": "asdasdajsdkasjdkkasdsak"
}
""".data(using: .utf8) else {
            return
        }
        
        do {
            let model = try JSONDecoder().decode(Employee.self, from: data)
            /// 如果子类没有重写'init(from:)', 此处只会解码父类属性
            dump(model)
        } catch  {
            
        }
}
```