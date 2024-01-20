---
title: "聊一聊TypeScript的interface"
date: 2023-04-14
thumbnailImagePosition: right
categories:
- FrontEnd
- JavaScript
tags:
- TypeScript

metaAlignment: center
---
"如果它表现得像个鸭子，那么就可以称它为鸭子"。interface（接口），它用于定义对象的形状或结构。

<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/bg/typesc-brick-bg.jpg)

### 前言
`TypeScript`是`JavaScript`的超集，它为`JavaScript`添加了静态类型检查和其他一些高级特性。其中一个重要的概念是interface（接口），它用于定义对象的形状或结构。

interface可以理解为一个契约，用于描述一个对象的属性、方法和函数签名。在TypeScript中，我们可以使用interface来实现对代码的抽象和模块化。interface设计原理基于面向对象编程思想，通过将相同或相似的行为和特征抽象为一个接口，从而实现代码重用和降低耦合度。

## 基础概念
### 接口定义:

```TypeScript
interface Person {
  firstName: string;
  lastName: string;
  age: number;
  greet(): void;
}
```
这个接口定义了一个Person类型的对象，它有firstName、lastName和age三个属性，以及一个greet方法。需要注意的是，在interface中只声明了属性和方法的名称和类型，没有具体的实现。  

我们可以使用这个接口来定义一个符合该接口要求的对象：  

```TypeScript
const person: Person = {
  firstName: 'John',
  lastName: 'Doe',
  age: 30,
  greet() {
    console.log(`Hello, my name is ${this.firstName} ${this.lastName}.`);
  },
};
```

上述代码定义了一个名为person的常量，它是一个Person类型的对象。person对象具有firstName、lastName、age和greet四个属性，其中greet属性指向一个函数。这个函数实现了在接口中定义的greet方法，它可以输出一条问候语。

由于person对象符合Person接口的定义，因此TypeScript不会给出任何警告或错误提示。

总的来说，interface是TypeScript中重要的概念之一，它可以用来描述对象的形状、实现抽象和模块化等。在实际开发中，我们可以根据具体情况灵活运用interface，从而提高代码质量和可维护性。


![duck demo](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/bg/duck.jpg)

### 泛型与延伸
**TypeScript的interface也可以结合泛型进行拓展**
```TypeScript
interface Box<T> {
  value: T;
}

const box: Box<string> = { value: 'Hello, ts <T> fooing' };
```

**类型别名：可以使用type关键字定义一个类型别名，方便复用某个类型。例如：**
```TypeScript
type UserId = string;
type UserName = string;

interface User {
  id: UserId;
  name: UserName;
}
```

**可索引类型：可以用接口来描述具有索引签名的类型（如数组或对象）。例如：**
```TypeScript
interface StringArray {
  [index: number]: string;
}

const arr: StringArray = ['foo', 'bar'];
```

**接口继承：可以通过extends关键字让一个接口去继承另一个接口中的属性和方法。例如：**
```TypeScript
interface Animal {
  eat(): void;
  sleep(): void;
}

interface Cat extends Animal {
  meow(): void;
}

class HouseCat implements Cat {
  eat() { console.log('Nom nom nom'); }
  sleep() { console.log('Zzzzzz...'); }
  meow() { console.log('Meow!'); }
}
```

### 常用前端框架的拓展
引用一些使用这些模式的流行框架及其内部代码：
> Vue.js 使用了 TypeScript 的接口继承和类型别名等高级特性。例如，Vue.js 中定义组件时可以使用Props类型别名来指定组件的属性类型： 
```Javascript
import { PropType } from 'vue';

type Props = {
  msg: string;
};

export default {
  props: {
    msg: {
      type: String as PropType<Props['msg']>,
      required: true
    }
  },
  // ...
}
```

> NestJS 是一个基于 Node.js 和 TypeScript 的开发框架，它广泛使用了 TypeScript 的 interface 高级用法。例如，NestJS 中的服务类可以定义依赖注入的参数类型
```JavaScript
@Injectable()
export class CatsService {
  constructor(
    private readonly catsRepository: CatsRepository,
    private readonly logger: LoggerService,
  ) {}

  async create(catDto: CreateCatDto): Promise<Cat> {
    const createdCat = await this.catsRepository.create(catDto);
    this.logger.log(`Created a cat with id ${createdCat.id}`);
    return createdCat;
  }
}
```

> RxJS 是一个响应式编程库，它使用了 TypeScript 的泛型接口来实现类型安全的流式处理操作。例如，RxJS 中的Observable类就是一个泛型接口：
```JavaScript
interface Observable<T> {
  subscribe(observer?: PartialObserver<T>): Subscription;
  // ...
}

const source$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});

source$.subscribe({
  next: value => console.log(value),
  complete: () => console.log('Complete!')
});
```

## 参考链接
- **TypeScript手册**  
- **《前端interface设计艺术》**  
- **响应式编程的演变**  