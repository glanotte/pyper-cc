# TypeScript Patterns

## Type vs Interface

```tsx
// Use interface for objects that can be extended
interface User {
  id: string;
  name: string;
}

interface Admin extends User {
  permissions: string[];
}

// Use type for unions, tuples, and complex types
type Status = 'pending' | 'success' | 'error';
type Tuple = [string, number];
type Callback = (value: string) => void;
```

## Generic Components

```tsx
// Generic list component
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage - TypeScript infers T from items
<List
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

## Utility Types

```tsx
// Pick - select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit - exclude properties
type UserInput = Omit<User, 'id' | 'createdAt'>;

// Partial - all properties optional
type UserUpdate = Partial<User>;

// Required - all properties required
type CompleteUser = Required<User>;

// Record - object type with specific keys
type UserMap = Record<string, User>;

// Extract/Exclude - filter union types
type SuccessStatus = Extract<Status, 'success' | 'pending'>;
type FailureStatus = Exclude<Status, 'success'>;
```

## Discriminated Unions

```tsx
type LoadingState = { status: 'loading' };
type SuccessState<T> = { status: 'success'; data: T };
type ErrorState = { status: 'error'; error: Error };

type AsyncState<T> = LoadingState | SuccessState<T> | ErrorState;

function handleState<T>(state: AsyncState<T>) {
  switch (state.status) {
    case 'loading':
      return <Spinner />;
    case 'success':
      return <Data data={state.data} />;  // TypeScript knows data exists
    case 'error':
      return <Error error={state.error} />; // TypeScript knows error exists
  }
}
```

## Type Guards

```tsx
// Custom type guard
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// Usage
function processItem(item: unknown) {
  if (isUser(item)) {
    console.log(item.name);  // TypeScript knows item is User
  }
}
```

## Const Assertions

```tsx
// Without as const - type is string[]
const statuses = ['pending', 'success', 'error'];

// With as const - type is readonly ['pending', 'success', 'error']
const statuses = ['pending', 'success', 'error'] as const;
type Status = (typeof statuses)[number];  // 'pending' | 'success' | 'error'

// Objects
const config = {
  endpoint: '/api',
  timeout: 5000,
} as const;
// config.endpoint is '/api', not string
```

## Event Types

```tsx
// Form events
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};

// Input events
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setValue(e.target.value);
};

// Mouse events
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  console.log(e.clientX, e.clientY);
};

// Keyboard events
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === 'Enter') submit();
};
```

## Component Props Types

```tsx
// Extend HTML element props
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

// Get props type from component
type InputProps = React.ComponentProps<'input'>;
type MyComponentProps = React.ComponentProps<typeof MyComponent>;

// Children type
interface LayoutProps {
  children: React.ReactNode;
}
```

## Strict Null Checks

```tsx
// Avoid !
// Bad
const name = user!.name;

// Good - handle null explicitly
const name = user?.name ?? 'Unknown';

// Good - early return
if (!user) return null;
const name = user.name;  // user is narrowed to non-null
```
