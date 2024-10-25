+++
title = "From Props Hell to the Compound Pattern: How to Design a Common React Component Library"
date = 2024-01-11
draft = false
tags = ["React", "Patterns", "FE"]
+++

## Background

### The Emergence of Props Hell

When encapsulating a common component library, we often encounter a problem: the initial design seems reasonable, but as the project development progresses and UI requirements continuously change, the component's logic and props become increasingly complex.

<!--more-->

For example, the initial component design only needed to display `title` and `description`:

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

// Usage
<Card title={title} description={description} />;
```

But as the project progressed, the requirements changed—we needed to edit the card content in certain scenarios.

So we needed to:

1. Add an Edit Button.
2. When the Edit Button is clicked, a dialog pops up to modify the `description`.

To maintain compatibility with existing usage, we added two optional props: `isEditable` and `updateDescription`.

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

// Usage
<Card
  title={title}
  description={description}
  isEditable={true}
  updateDescription={updateDescription}
/>;
```

Later, after two more months, a new requirement came in: the description text of the popup dialog needs to be specified according to the scenario. So the `Card` component had to add another optional prop called `dialogDescription`.

If we need to use other encapsulated components, we might even have to pass `props` layer by layer.

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

Over time, the `Card` component becomes increasingly messy and hard to read, with `&&` scattered everywhere and props extending infinitely. The component's functionality relies entirely on guessing whether a certain prop is used.

This approach of gradually adding props is untenable.

## How to Deal with Props Hell?

After studying mature UI component libraries , I found that most of them use the Compound Pattern to solve this problem 💡!

**How do other component libraries use the Compound Pattern?**

- Split the component into multiple sub-components.
- Use the namespace component pattern to show the hierarchical relationship between components and avoid naming conflicts.
- Share state through Context.

## Implementing the Compound Pattern

### Component Decomposition

We can split the original `Card` component into multiple smaller sub-components, allowing each sub-component to focus on its own functionality.

For example, first extract parts like `title` and `description` that do not depend on optional props:

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

// Usage
<Card>
  <Card.Title>Hello</Card.Title>
  <Card.Description>World</Card.Description>
</Card>;
```

As for the content of the optional props, we'll handle them separately in the `Edit` component.

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

// Usage
return (
  <Card>
    <Card.Title>{title}</Card.Title>
    <Card.Description>{description}</Card.Description>
    <Card.Edit updateDescription={updateDescription} />
  </Card>
);
```

Doesn't it look much cleaner now? If the `Card` component needs to incorporate new sub-components, in most cases we only need to add sub-components without modifying the `Card` component itself.

Additionally, using `Card.xx` clearly shows the hierarchical relationship between components.

But what if someone misuses it? 🤔 For instance, uses `Card.xx` elsewhere—that would be awkward.

And what if we need to pass values to the sub-components of `Card`'s sub-components?

### State Management

Let's upgrade by using Context to share state and throw errors when there is no Context.

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

// Correct usage
<Card textColor="blue">
  <Card.Title>Hello</Card.Title>
  <Card.Description>World</Card.Description>
</Card>;

// Incorrect usage - Error: DescriptionInner must be used within a Card
<Card.Edit updateDescription={updateDescription} />;
```

## Conclusion

When developing a common component library, as the complexity of the component increases, simply passing props makes the component difficult to maintain.

At this point, using the Compound Pattern can effectively solve the problem. By splitting complex components into multiple single-function sub-components and managing state through Context, we enhance the component's extensibility and maintainability.

Programming is a dynamic process, and requirements are constantly changing. During development, we need to balance the flexibility and complexity of components and choose design patterns reasonably to solve problems.

Of course, over-designing can also increase development costs, so we need to make trade-offs based on actual situations.
