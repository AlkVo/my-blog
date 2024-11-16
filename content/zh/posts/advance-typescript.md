+++
title = 'Advance Typescript'
date = 2024-03-01T16:23:03+08:00
draft = false
tags = ["FE", "TypeScript"]
+++

## 背景

### 使用 TypeScript 的好处

使用 TypeScript 有很多的好处，它可以让代码变得更可靠和更容易维护：

<!--more-->

- **可靠性**: TypeScript 的类型系统能在写代码时捕捉到很多错误，这样可以减少运行时的出错几率，让代码更加稳定。
- **可读性**: 明确的类型定义让代码更容易理解，这对开发者来说维护起来更简单。
- **重命名和重构的低成本**:类型检查器可以帮助我们在重命名和重构代码时找到正确的内容，从而有效减少新错误的引入风险。

### 如果不小心滥用...?

然而，TypeScript 也像一把双刃剑，如果不小心滥用，可能会引发一些问题：

- **联合类型的滥用**: 随着业务逻辑的复杂化，联合类型可能会变得越来越复杂，导致某些属性变得不可访问。

  ```TypeScript
  type A = { foo: string };
  type B = { bar: number };
  type C = A | B;

  function getFoo(obj: C) {
    obj.foo(); // ❌ 不存在
    obj.bar(); // ❌ 不存在
  }

  ```

- **类型断言的滥用**: 为了处理复杂的类型问题，可能会频繁使用 `as`操作符。虽然这可以把问题解决，但会使代码变得不那么优雅和脆弱。

这些滥用常常会让代码更难维护，也更容易出错。

接下来，我会结合自己的开发经验，和大家讨论如何更好地使用 TypeScript。

例如泛型、`extends`、`keyof`、`infer`，甚至类型递归，帮助我们编写更健壮、优雅且易维护的代码。

通过深入理解和应用这些特性，我们可以更好地应对复杂类型的挑战，同时保持代码的清晰和可读性。

---

## Let's go ~

### 1. Generic

泛型是一种让函数、接口或类型可以接受多种类型的特性，使代码更加通用和可复用。

任何类型都可以成为泛型。它由你来制定。

比如下面的代码，当传入的类型 T 是 `number` ，那么 返回类型 T 就是 `number`。当传入的类型 T 是 `string` ，那么 返回类型 T 就是 `string`。

```TypeScript
function identity<T>(arg: T): T {
    return arg;
}
const num = identity(42); // number
const str = identity('hello'); // string
```

泛型的使用可以帮助我们减少代码重复，增强类型的灵活性和安全性。

例如处理 API 响应时，`ApiResponse<T>` 接口可以接受不同类型的数据结构。

```TypeScript
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
};

// 👤
type User = { id: number; name: string };
const userResponse: ApiResponse<User> = {
  data: { id: 1, name: "Alice" },
  status: 200,
  message: "Success",
};

// 📖
type Book = {
  id: number;
  title: string;
  author: string;
  price: number;
  inStock: boolean;
};
const bookResponse: ApiResponse<Book> = {
  data: {
    id: 1,
    title: "深入理解 TypeScript",
    author: "张三",
    price: 59.99,
    inStock: true,
  },
  status: 200,
  message: "Success",
};

```

### 2. keyof

`keyof` 可以轻松获取 object 的 keys

```TypeScript
type User = {
    id: number;
    name: string;
    age: number;
};

type UserKeys = keyof User; // "id" | "name" | "age"

```

假设我们的后端正在做 v1 到 v2 的版本升级，v1 版本接口和 v2 版本接口数据的改变是 key 的命名方式从 snake_case 变成 camelCase，其余都一致。那么原本函数的数据处理逻辑我们可以保留，只需把它传入的数据类型变为泛型，并且通过 `keyof` 约束需要被处理的 key 的输入。

```TypeScript
enum PayeeType {
  CASH_PAYEE = 1,
  NON_CASH_PAYEE = 2,
}

type Address = {
  is_cash_payment_stop: PayeeType
}

type AddressV2 = {
  isCashPaymentStop: PayeeType
}

const getCashPayeeText = <T>(address: T[], field: keyof T) => {
  const cashPayeeIndex = address.findIndex(
    (addr) => addr[field] === PayeeType.CASH_PAYEE,
  )

  if (cashPayeeIndex === -1) {
    return 'Not Specified'
  }
  const isFirstStop = cashPayeeIndex === 0
  if (isFirstStop) {
    return 'Sender'
  }

  const isLastStop = cashPayeeIndex === address.length - 1
  if (isLastStop) {
    return 'Recipient'
  }
  return `Stop ${cashPayeeIndex + 1}`
}

const getCashPayeeTextV1 = (address: Address[]) =>
  getCashPayeeText(address, 'is_cash_payment_stop')

const getCashPayeeTextV2 = (address: AddressV2[]) =>
  getCashPayeeText(address, 'isCashPaymentStop')

//❌
const getCashPayeeTextV2 = (address: AddressV2[]) =>
  getCashPayeeText(address, '123')
//❌
const getCashPayeeTextV2 = (address: AddressV2[]) =>
  getCashPayeeText(address, 'is_cash_payment_stop')
```

但在使用中可能会发现一个问题，由于 `keyof` 的结果是一个联合类型，那么 `Type[keyof Type]` 的类型也会是联合类型。

所以在下面的函数中，即使我们知道传入的 key 是什么类型，但是返回类型却是一个联合类型

```TypeScript
function getProperty<T>(obj: T, key: keyof T) {
  return obj[key]
}

const user = {
  id: 1,
  name: 'Alice',
  age: 30,
}

const myName = getProperty(user, 'name') //  string|number
const myAge = getProperty(user, 'age') // string|number
```

真是让人沮丧！

那么如何解决这个问题呢？交给 `extends`。

### 3. extends

`extends` 操作符可用于类型约束，确保传入的类型具有特定的结构。

`K extends keyof T`可以理解为，`K` 是某个类型，这个类型在 `keyof T` 内。

既然不是联合类型，而是单一类型 ，那么类型推导可以根据 `K` 的类型成功推导出 `T[K]`是什么类型了 🎉！

```TypeScript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const user = {
  id: 1,
  name: 'Alice',
  age: 30,
}

const myName = getProperty(user, 'name') //  string
const myAge = getProperty(user, 'age') // number
```

`extends`也可以和三目运算符结合，得到想要的类型

```TypeScript
type Action =
  | { action: 'create'; item: string }
  | { action: 'update'; itemId: number; newValue: string }
  | { action: 'delete'; itemId: number }

type ExtractAction<T, U> = T extends { action: U } ? T : never

type CreateAction = ExtractAction<Action, 'create'> // { action: 'create'; item: string }
type UpdateAction = ExtractAction<Action, 'update'> // { action: 'update'; itemId: number; newValue: string }
type DeleteAction = ExtractAction<Action, 'delete'> // { action: 'delete'; itemId: number }
```

### 4. in

`in` 操作符可以用于遍历对象类型的键，从而生成新的对象类型。
这种方式可以用于生成具有特定键名和类型的对象，增加代码的灵活性。

```TypeScript
type ValueBeKey<T extends Record<string, unknown>> = {
  [P in keyof T]: P;
};

const object = {
  a: 1,
  b: 2,
  c: 3,
};

const result: ValueBeKey<typeof object> = {
  a: 'a',
  b: 'b',
  c: 'c',
};
```

让我们来实现官方提供的 Partial 和 Required

```TypeScript
type User = {
  id: number
  name: string
  age: number
}
type Partial<T> = {
  [K in keyof T]?: T[K]
}
type Required<T> = {
  [K in keyof T]-?: T[K]
}
type OptionalUser = Partial<User>
type RequiredUser = Required<OptionalUser>

```

也可以结合 `as`,根据以前的 key 生成新的 key name

```TypeScript
type MappedUser = {
  [K in keyof User as `user_${K}`]: User[K]
}
type UserWithPrefixedKeys = MappedUser // { user_id: number; user_name: string; user_age: number }
```

或者和 `extends`结合，筛选出符合对应类型的 key

```TypeScript
type FilterByType<T, Condition> = {
  [K in keyof T]: T[K] extends Condition ? K : never
}[keyof T]
type StringKeysOfUser = FilterByType<User, string> // name
type NumberKeysOfUser = FilterByType<User, number> // "id" | "age"
```

假如存在一个接口，它的请求中如果带有 `user` 则接口中就会返回 `user`对象，它的请求中如果带有 `driver` 则接口中就会返回 `driver`对象。它的请求中如果带有`user`和`driver`，那么就会返回`user`和`driver`对象。那么我们就可以结合上面的 `extends` 帮我们完成这件事。

```TypeScript
type Extensions = {
  user: { id: string; name: string }
  driver: { phone: string }
}

type GetOrderParams<K extends keyof Extensions> = {
  orderUuid: string
  extendKeys?: K[]
}
type OrderResponse<K extends keyof Extensions> = {
  orderUuid: string
  cityId: string
  marketId: number
  extensions: {
    [key in K]: Extensions[key]
  }
}

export const getOrder = <K extends keyof Extensions>({
  orderUuid,
  extendKeys,
}: GetOrderParams<K>) =>
  httpClient.get<OrderResponse<K>>('/api/order/detail', {
    params: {
      orderUuid,
      ...(!isEmpty(extendKeys) && { extendKeys: extendKeys?.join(',') }),
    },
  })

const order = getOrder({ orderUuid: '123', extendKeys: ['user', 'driver'] })
order.then(({ data }) => {
  console.log(data.extensions.user.name)
  console.log(data.extensions.driver.phone)
})
```

### 5. [number]

使用索引访问类型可以获取元组或数组中元素的类型。

```TypeScript
type Parent = [string, number];
type Child = Parent[number]; // string | number
```

### 6. infer

`infer` 用于在条件类型中和 `extends `结合引入一个变量，以推断某个类型。

它在复杂类型操作中非常有用，特别是在处理函数返回值类型时。

```TypeScript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

interface OrderReportsProps {
  isDisplayLanguageDropdown: boolean;
  language: ReturnType<typeof useLanguage>;
}
```

可以实现和上面的 `[number]` 一样的功能

```Typescript
type Parent = [string, number];
type Flatten<T> = T extends (infer R)[] ? R : T;
type Child = Flatten<Parent>; // string | number
```

也可以用于获取某个 filed 的值

```TypeScript
type Action =
  | { action: "create"; item: string }
  | { action: "update"; itemId: number; newValue: string }
  | { action: "delete"; itemId: number };

type GetActionValue<T> = T extends { action: infer U } ? U : never;
type Value = GetActionValue<Action>; // "create" | "update" | "delete"
```

### 7. 递归

是的，类型也可以使用递归 ! 有很多有趣的玩法。

但要注意，只有 `type` 创造的类型才可以进行递归，`interface` 的不可以

```TypeScript
type Test = "123"
type StrintToUnion<T extends string> = T extends `${infer Letter }${infer Rest}`
  ? Letter | StrintToUnion<Rest>
  : never;
type Result = StringToUnion<Test> //  "1" | "2" | "3"
```

假设我们想要拿到某个 nested 对象的 key 去更新，为了避免 typo ，就可以利用递归创建一个 key value 都等于这个 nested 对象所有 key 的类型。

```TypeScript
type ObjectKeyPaths<T, K extends keyof T = keyof T> = T extends Record<
  string,
  unknown
>
  ? K extends string
    ? `${K}${'' | `${'.'}${ObjectKeyPaths<T[K]>}`}`
    : never
  : never

type Paths = ObjectKeyPaths<{
  foo: {
    bar: {
      baz: number
    }
  }
  arr: number[]
}>

const pathsObj: { [key in Paths]: key } = {
  foo: 'foo',
  arr: 'arr',
  'foo.bar': 'foo.bar',
  'foo.bar.baz': 'foo.bar.baz',
}
```

## 更多

TypeScript 其实有很多 fancy 的写法，我也是从这个 [repo](https://github.com/type-challenges/type-challenges) 学了不少感兴趣的小伙伴可以多刷刷来更好地了解 TS 的类型系统 ～
