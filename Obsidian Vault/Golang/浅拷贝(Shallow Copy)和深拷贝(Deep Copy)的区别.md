1. 浅拷贝：浅拷贝是指创建一个新对象，这个新对象与原对象有着相同的属性值。对于值类型（例如基本数据类型）的属性，浅拷贝会直接复制其值。而对于引用类型（例如指针、切片、映射、接口等）的属性，浅拷贝只会复制引用，因此新旧对象中的这些属性仍然指向同一个底层数据。
```
// 浅拷贝
package main

import (
    "fmt"
)

type Person struct {
    Name string
    Age  int
    Address *Address
}

type Address struct {
    City string
    State string
}

func main() {
    original := Person{
        Name: "John",
        Age:  30,
        Address: &Address{
            City:  "New York",
            State: "NY",
        },
    }

    // 浅拷贝
    shallowCopy := original

    // 修改浅拷贝的地址
    shallowCopy.Address.City = "San Francisco"

    fmt.Println("Original:", original.Address.City)    // San Francisco
    fmt.Println("Shallow Copy:", shallowCopy.Address.City) // San Francisco
}

```

2. 深拷贝：深拷贝是指创建一个新对象，这个新对象与原对象有着相同的属性值，同时对于引用类型的属性，深拷贝会递归地创建新的实例，并复制所有的子属性。因此，新旧对象中的引用类型属性指向不同的内存地址，修改其中一个不会影响另一个。
```
package main

import (
    "fmt"
)

type Person struct {
    Name    string
    Age     int
    Address *Address
}

type Address struct {
    City  string
    State string
}

// 深拷贝函数
func deepCopyPerson(original Person) Person {
    copyAddress := *original.Address
    return Person{
        Name:    original.Name,
        Age:     original.Age,
        Address: &copyAddress,
    }
}

func main() {
    original := Person{
        Name: "John",
        Age:  30,
        Address: &Address{
            City:  "New York",
            State: "NY",
        },
    }

    // 深拷贝
    deepCopy := deepCopyPerson(original)

    // 修改深拷贝的地址
    deepCopy.Address.City = "San Francisco"

    fmt.Println("Original:", original.Address.City) // New York
    fmt.Println("Deep Copy:", deepCopy.Address.City) // San Francisco
}

```

3. 小节
- **浅拷贝**：复制对象，但对于引用类型，只复制引用，因此新旧对象共享引用类型的数据。
- **深拷贝**：复制对象及其引用类型的所有子属性，创建新的实例，因此新旧对象完全独立。