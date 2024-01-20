---
title: "Talk about TypeScript's interface"
date: 2023-02-04
thumbnailImagePosition: right
categories:
- FrontEnd
- JavaScript
tags:
- TypeScript

metaAlignment: center
---
""Call it a duck if it looks like one." Interface is used to define the shape or structure of an object.

<!--more-->
![recover](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/bg/typesc-brick-bg.jpg)

### Preface
TypeScript is a super set of JavaScript, which adds static type checking and other advanced features to JavaScript. One important concept is interface, which is used to define the shape or structure of an object.

Interface can be understood as a contract that describes the properties, methods, and function signatures of an object. In TypeScript, we can use interface to implement abstraction and modularity in code. The design principle of interface is based on the object-oriented programming idea, which abstracts the same or similar behaviors and characteristics into an interface, so as to achieve code reuse and reduce coupling.


## Basic Concepts
### Interface Definition:

```TypeScript
interface Person {
  firstName: string;
  lastName: string;
  age: number;
  greet(): void;
}
```
This interface defines an object of type Person, which has three properties: firstName, lastName, and age, as well as a greet method. Note that only the names and types of properties and methods are declared in the interface, without specific implementation.

We can use this interface to define an object that meets the requirements of the interface:

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

The above code defines a constant named "person", which is an object of type Person. The person object has four properties: firstName, lastName, age and greet, where the greet property points to a function. This function implements the greet method defined in the interface and can output a greeting.

Since the person object meets the definition of the Person interface, TypeScript does not give any warnings or error prompts.

In general, interface is one of the important concepts in TypeScript, which can be used to describe the shape of objects, implement abstraction and modularity, etc. In actual development, we can flexibly use interface according to specific situations, so as to improve code quality and maintainability.




![duck demo](https://pixelpig-1253685321.cos.ap-guangzhou.myqcloud.com/blog/bg/duck.jpg)

### Generic and Extension
**TypeScript's interface can also be extended with generics**
```TypeScript
interface Box<T> {
  value: T;
}

const box: Box<string> = { value: 'Hello, ts <T> fooing' };
```

**Type Alias: Type alias can be defined using the type keyword to facilitate the reuse of a type. For example:**
```TypeScript
type UserId = string;
type UserName = string;

interface User {
  id: UserId;
  name: UserName;
}
```

**Indexable Types: Interface can be used to describe types with index signatures (such as arrays or objects). For example:**
```TypeScript
interface StringArray {
  [index: number]: string;
}

const arr: StringArray = ['foo', 'bar'];
```

**Interface Inheritance: Another interface's property and method can be inherited through the extends keyword. For example:**
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

### Extensions of Popular Front-End Frameworks
Here are some popular frameworks that use these patterns:

> Vue.js uses advanced features such as interface inheritance and type aliases of TypeScript. For example, when defining a component in Vue.js, Props type alias can be used to specify the property type of the component:
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

> NestJS is a Node.js and TypeScript-based development framework that extensively uses advanced interfaces of TypeScript. For example, the service class in NestJS can define the parameter type of dependency injection:
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

> RxJS is a reactive programming library that uses TypeScript's generic interfaces to implement type-safe streaming operations. For example, the Observable class in RxJS is a generic interface:
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
- **TypeScript official doc**  
- **《The art of front-end interface》**  
- **The Evolution of Reactive Programming**  