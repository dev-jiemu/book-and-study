## NestJS Decorator
#### 관점지향 프로그래밍 
tsconfig.json 파일에서 experimentalDecorators 옵션을 `true` 로 주면 사용 가능
```typescript
class CreateUserDto {
    @IsEmail()
    @MaxLength(60)
    readonly email: string
    
    @IsString()
    @Matches(/^[A-Za-z\d!@#$%^&*()]{8,30}$)
    readonly password: string
}
```
##### 데커레이터 합성
적용 서순
1. 위에서 아래로 평가 됨
2. 아래에서 위로 함수가 호출 됨
```typescript
const first = () => {
    console.log('first factory evaluated')
    return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
        console.log('first called')
    }
}

const second = () => {
    console.log('second factory evaluated')
    return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
        console.log('second called')
    }
}

class ExampleClass {
    @first
    @second
    test() {
        console.log('test method')
    }
}
// result
// first factory
// second factory
// second called
// first called
// test method
```

##### 클래스 데커레이터

```typescript
const reportableClassDecorator = <T extends { new(...args: any[]): {} }>(constructor: T) =>
        class extends constructor {
            reportingUrl = "https://www.example.com";
        }

@reportableClassDecorator
class BugReport {
    type = "report"
    title: string

    constructor(t: string) {
        this.title = t
    }
}

// {type: report, title: t, reportingUrl: https://www.example.com}
```

##### 메서드 데커레이터
1. 정적 멤버가 속한 클래스의 생성자 함수 또는 인스턴트 멤버에 대한 클래스 프로토타입
2. 멤버 이름
3. 속성 설명자 : PropertyDescriptor

`PropertyDescriptor Interface`
```typescript
interface PropertyDescriptor {
    configurable?: boolean
    enumerable?: boolean
    value?: any
    writable?: boolean
    get?(): any
    set?(v: any): void
}
```

##### 접근자 데커레이터

getter, setter
```typescript
const Enumerable = (enumerable: boolean) => 
    (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
        descriptor.enumerable = enumerable
    }


class Person {
    constructor(private name: string) {}
    
    @Enumerable(true)
    get getName() {
        return this.name
    }
    
    @Enumerable(false)
    set setName(name: string) {
        this.name = name
    }
}

// name: jiemu
// getName: jiemu
// setName 은 안나옴
const person = new Person('jiemu')
for (let key in person) {
    console.log(`${key}: ${person[key]}`)
}
```

##### 속성 데커레이터
**  프로토타입 멤버 정의할때 인스턴스 속성을 설명하는 메커니즘이 없고 관찰, 수정할 방법이 없어서 속성 설명자가 없다나.. ㅇㅂㅇ...
```typescript
const format = (formatString: string) =>
        (target: any, propertyKey: string): any => {
            let value = target[propertyKey]

            const getter = () => `${formatString} ${value}`

            const setter = (newValue: string) => {
                value = newValue
            }

            return {
                get: getter,
                set: setter,
                enumerable: true,
                configurable: true
            }
        }

class Greeter {
    @format('Hello')
    greeting: string
}
```