+++
title = 'Advance Typescript'
date = 2024-03-01T16:23:03+08:00
draft = false
tags = ["FE", "TypeScript"]
+++

## Background

### Benefits of Using TypeScript

There are many benefits to using TypeScript, which can make your code more reliable and easier to maintain:

- **Reliability**: TypeScript's type system catches many errors at coding time, reducing runtime errors and making the code more stable.
- **Readability**: Explicit type definitions make the code easier to understand, making it simpler to maintain.
- **Low-Cost Refactoring**: The type checker assists in renaming and refactoring code, effectively reducing the risk of introducing new bugs.

### Potential Pitfalls of Misuse

However, TypeScript can also be a double-edged sword. If misused, it can lead to issues:

<!--more-->

- **Misuse of Union Types**: As business logic becomes more complex, union types may grow too large, making certain properties inaccessible.

  ```TypeScript
  type A = { foo: string };
  type B = { bar: number };
  type C = A | B;

  function getFoo(obj: C) {
    obj.foo(); // ❌ 不存在
    obj.bar(); // ❌ 不存在
  }

  ```

- **Overuse of Type Assertions**: The frequent use of the `as` operator to deal with complex type problems may solve the issue temporarily but can make the code fragile and less elegant.

These misuses often lead to code that is more difficult to maintain and easier to break.

Next, I will discuss how to make better use of TypeScript based on my own development experience.

We’ll cover generics, `extends`、`keyof`、`infer`,and even type recursion, which can help us write more robust, elegant, and maintainable code.

By deeply understanding and applying these features, we can better handle complex types while maintaining code clarity and readability.

---

## Let's go ~

### 1. Generic

Generics are a feature that allows functions, interfaces, or types to accept multiple types, making the code more reusable and generalized.

Any type can be used as a generic. It’s up to you.

For example, in the code below, when the type `T` is `number`, the return type is `number`. When T is `string`, the return type is `string`.

```TypeScript
function identity<T>(arg: T): T {
    return arg;
}
const num = identity(42); // number
const str = identity('hello'); // string
```

Using generics can help reduce code duplication and enhance type flexibility and safety.

For example, when dealing with API responses, the `ApiResponse<T>` interface can accept different data structures.

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

`keyof` can be used to easily get the keys of an object.

```TypeScript
type User = {
    id: number;
    name: string;
    age: number;
};

type UserKeys = keyof User; // "id" | "name" | "age"

```

Suppose our backend is migrating from v1 to v2, and the change involves converting the key naming style from snake_case to camelCase, with everything else remaining the same. We can keep the original function's data processing logic by passing the data type as a generic and using `keyof` to constrain the keys that need to be processed.

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

However, a problem might arise: since the result of `keyof` is a union type, `Type[keyof Type]` is also a union type.

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

Frustrating, isn't it?

So how do we solve this problem? Enter `extends`。

### 3. extends

The `extends` keyword is used for type constraints, ensuring that the type passed in has a specific structure.

`K extends keyof T` means that `K` is a type within `keyof T`.

Since `K` is not a union type but a specific key, TypeScript can accurately infer `T[K]` based on `K` 🎉!

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

`extends` can also be combined with the ternary operator to derive desired types.

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

The `in` operator iterates over the keys of an object type to generate a new object type. This method creates objects with specific key names and types, enhancing code flexibility.

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

Let's implement TypeScript's built-in `Partial` and `Required` utility types.

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

You can also use `as` to generate new key names based on existing keys.

```TypeScript
type MappedUser = {
  [K in keyof User as `user_${K}`]: User[K]
}
type UserWithPrefixedKeys = MappedUser // { user_id: number; user_name: string; user_age: number }
```

Or combine with `extends` to filter keys matching a specific type.

```TypeScript
type FilterByType<T, Condition> = {
  [K in keyof T]: T[K] extends Condition ? K : never
}[keyof T]
type StringKeysOfUser = FilterByType<User, string> // name
type NumberKeysOfUser = FilterByType<User, number> // "id" | "age"
```

Suppose there's an API where including `user` in the request returns a `user` object, including `driver` returns a `driver` object, and including both returns both objects. We can use `extends` to achieve this.

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

Using indexed access types, you can retrieve the types of elements within tuples or arrays.

```TypeScript
type Parent = [string, number];
type Child = Parent[number]; // string | number
```

### 6. infer

The `infer` keyword, used within conditional types and combined with `extends`, introduces a variable to infer a type.

It's particularly useful in complex type manipulations, especially when handling function return types.

```TypeScript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

interface OrderReportsProps {
  isDisplayLanguageDropdown: boolean;
  language: ReturnType<typeof useLanguage>;
}
```

You can achieve functionality similar to `[number]`:

```Typescript
type Parent = [string, number];
type Flatten<T> = T extends (infer R)[] ? R : T;
type Child = Flatten<Parent>; // string | number
```

It can also be used to extract specific field values:

```TypeScript
type Action =
  | { action: "create"; item: string }
  | { action: "update"; itemId: number; newValue: string }
  | { action: "delete"; itemId: number };

type GetActionValue<T> = T extends { action: infer U } ? U : never;
type Value = GetActionValue<Action>; // "create" | "update" | "delete"
```

### 7. Recursion

However, note that only types created with `type` can be recursive; those created with `interface` cannot.

```TypeScript
type Test = "123"
type StrintToUnion<T extends string> = T extends `${infer Letter }${infer Rest}`
  ? Letter | StrintToUnion<Rest>
  : never;
type Result = StringToUnion<Test> //  "1" | "2" | "3"
```

Suppose we want to retrieve all keys of a nested object to update it. To avoid typos, we can use recursion to create a type where keys and values are all keys of this nested object.

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

## More

TypeScript has many fancy patterns. I've learned a lot from this [repo](https://github.com/type-challenges/type-challenges). Interested folks can explore it to better understand TypeScript's type system!
