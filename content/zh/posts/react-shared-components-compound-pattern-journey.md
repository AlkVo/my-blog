+++
title = "从 Props Hell 到 Compound Pattern：该如何设计公共的 React 组件库" 
date = 2024-01-11 
draft = false 
tags = ["React","Patterns","FE"]
+++

## 背景

### Props Hell 的产生

在封装公共组件库时，我们常常会遇到一个问题：初始设计看似合理，但随着项目开发的深入，UI 需求不断变化，组件的逻辑和 props 变得愈加复杂。

<!--more-->

例如，一开始的组件设计是只需展示 `title` 和 `description：`

```tsx
type CardProps = {
  title: string;
  description: string;
};
const Card = ({ title, description }: CardProps) => {
  return (
    <div style={{ border: "1px solid black" }}>
      <h2>{title}</h2>
      <p>{description}</p>
    </div>
  );
};

// 使用
<Card title={title} description={description} />;
```

但随着项目进展，需求变化了——某些场景下需要编辑卡片的内容

所以我们需要：

1. 增加一个 Edit Button。
2. 点击 Edit Button 的时候，弹出弹框对 `description` 进行修改。

那么为了兼容旧的使用，新增了两个 optional props：`isEditable` 和 `updateDescription`。

```tsx
type CardProps = {
  title: string;
  description: string;
  isEditable?: boolean;
  updateDescription?: (newDescription: string) => void;
};

const Card = ({
  title,
  description,
  isEditable,
  updateDescription,
}: CardProps) => {
  const [isOpen, setIsOpen] = useState(false);
  const [newDescription, setNewDescription] = useState(description);

  const handleDescriptionChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setNewDescription(e.target.value);
  };
  const handleConfirm = () => {
    updateDescription?.(newDescription);
    setIsOpen(false);
  };

  return (
    <div style={{ border: "1px solid black" }}>
      <h2>{title}</h2>
      <p>{description}</p>
      {isEditable && <button onClick={() => setIsOpen(true)}>Edit</button>}
      {isOpen && (
        <dialog open>
          <input
            type="text"
            value={newDescription}
            onChange={handleDescriptionChange}
          />
          <button onClick={handleConfirm}>Confirm</button>
        </dialog>
      )}
    </div>
  );
};

// 使用
<Card
  title={title}
  description={description}
  isEditable={true}
  updateDescription={updateDescription}
/>;
```

后来再过了两个月，又来一个新需求，要求弹出的提示框的描述文字根据场景指定，那么 Card 组件又要新增一个叫做 `dialogDescription` 的 optional prop 了。

如果需要使用其他封装好的组件，我们甚至还要一层层地传递 `props`。

```tsx
const Card = ({
  title,
  description,
  isEditable,
  updateDescription,
  newProps1,
  newProps2,
}: CardProps) => {
  ...
  return (
    <div style={{ border: "1px solid black" }}>
      ...
      <OtherComponent neededProps={newProps1} />
      <OtherComponent neededProps={newProps2} />
    </div>
  );
};
```

长此以往，Card 组件会变得越来越丑陋和难读，`&&` 散播满地，`props` 无限延长，组件的功能全靠是否使用了某个 `prop` 来猜测。
这种逐渐增加 `props` 的方案是走不通了。

## 如何应对 Props Hell ？

经过研(偷)究(师)成熟的 UI 组件库 😏，我发现它们大多使用了 Compound Pattern 来解决这个问题 💡！

**其他组件库是怎么使用 Compound Pattern 的呢？**

- 将组件拆分为多个子组件。
- 通过命名空间组件模式，展示组件之间的从属关系，避免命名冲突。
- 通过 Context 来共享状态。

## 实现 Compound Pattern

### 组件拆分

我们可以把原来的 Card 组件拆分为多个更小的子组件，让每个子组件专注于自己的功能。

例如，先将 title 和 description 这些不依赖于 optional props 的部分拆分出来：

```tsx
const Title = ({ children }: React.PropsWithChildren) => {
  return <h2>{children}</h2>;
};

const Description = ({ children }: React.PropsWithChildren) => {
  return <p>{children}</p>;
};

const Card = ({ children }: React.PropsWithChildren) => {
  return <div style={{ border: "1px solid black" }}>{children}</div>;
};

Card.Title = Title;
Card.Description = Description;

// 使用
<Card>
  <Card.Title>Hello</Card.Title>
  <Card.Description>World</Card.Description>
</Card>;
```

至于 optional props 的内容，我们就桥归桥，路归路，放在 Edit 组件里处理。

```tsx
type EditProps = {
  updateDescription: (newDescription: string) => void;
};
const Edit = ({ updateDescription }: EditProps) => {
  const [isOpen, setIsOpen] = useState(false);
  const [newDescription, setNewDescription] = useState("");

  const handleDescriptionChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setNewDescription(e.target.value);
  };
  const handleConfirm = () => {
    updateDescription?.(newDescription);
    setIsOpen(false);
  };

  return (
    <>
      <button onClick={() => setIsOpen(true)}>Edit</button>
      {isOpen && (
        <dialog open>
          <input
            type="text"
            value={newDescription}
            onChange={handleDescriptionChange}
          />
          <button onClick={handleConfirm}>Confirm</button>
        </dialog>
      )}
    </>
  );
};

Card.Edit = Edit;

// 使用
return (
  <Card>
    <Card.Title>{title}</Card.Title>
    <Card.Description>{description}</Card.Description>
    <Card.Edit updateDescription={updateDescription} />
  </Card>
);
```

这么一看是不是清爽多了，万一 Card 组件需要新加入子组件，大多数情况下，我们只需新增子组件，而不用去修改 Card 组件。

并且可以通过 `Card.xx` 的方式，清晰地展示组件之间的从属关系。

不过有人手贱怎么办呢？🤔 在别的地方使用 `Card.xx` ，那就很尴尬了。

而且如果需要向 Card 的子组件的子组件传递值呢？

### 状态管理

让我们升级一下，利用 context 来共享状态 ，以及判断是否有 context 来抛出错使用错误。

```tsx
const DEFAULT_COLOR = "#000";

type CardProps = {
  textColor: string;
};

const CardContext = createContext<CardProps | null>(null);

const DescriptionInner = ({ children }: React.PropsWithChildren) => {
  const context = useContext(CardContext);
  if (!context) {
    throw new Error("DescriptionInner must be used within a Card");
  }
  return <span style={{ color: context?.textColor }}>{children}</span>;
};

const Description = ({ children }: React.PropsWithChildren) => {
  return (
    <p>
      <DescriptionInner>{children}</DescriptionInner>
    </p>
  );
};

const Card = ({ children, textColor }: React.PropsWithChildren<CardProps>) => {
  return (
    <CardContext.Provider value={{ textColor }}>
      <div style={{ border: "1px solid black" }}>{children}</div>
    </CardContext.Provider>
  );
};

// 正确使用
<Card textColor="blue">
  <Card.Title>Hello</Card.Title>
  <Card.Description>World</Card.Description>
</Card>;

// 错误使用 - 报错 DescriptionInner must be used within a Card
<Card.Edit updateDescription={updateDescription} />;
```

## 总结

在开发公共组件库时，随着组件的复杂度增加，简单地通过传递 props 的方式会让组件变得难以维护。

这个时候，使用 Compound Pattern 可以很好地解决问题，将复杂的组件拆分为多个功能单一的子组件，并通过 Context 进行状态管理，提升组件的可扩展性和可维护性。

编程是一个动态过程，需求也在不断变化。在开发过程中，我们需要平衡组件的灵活性和复杂性，合理选择设计模式来解决问题。

当然，过度设计也会增加开发成本，需要根据实际情况进行取舍。
