# Component Patterns

## Basic Component Structure

```tsx
// components/UserCard/UserCard.tsx
import { type FC } from 'react';
import styles from './UserCard.module.css';

interface UserCardProps {
  user: User;
  onSelect?: (user: User) => void;
  className?: string;
}

export const UserCard: FC<UserCardProps> = ({ user, onSelect, className }) => {
  const handleClick = () => {
    onSelect?.(user);
  };

  return (
    <div className={`${styles.card} ${className ?? ''}`} onClick={handleClick}>
      <img src={user.avatarUrl} alt={user.name} className={styles.avatar} />
      <div className={styles.info}>
        <h3>{user.name}</h3>
        <p>{user.email}</p>
      </div>
    </div>
  );
};
```

## Directory Structure

```
components/
  UserCard/
    UserCard.tsx        # Component
    UserCard.test.tsx   # Tests
    UserCard.module.css # Styles
    index.ts            # Re-export
```

## Props Patterns

### Required vs Optional
```tsx
interface Props {
  id: string;           // Required
  name?: string;        // Optional
  onClick?: () => void; // Optional callback
}
```

### Children
```tsx
interface Props {
  children: React.ReactNode;  // Any renderable content
}

// Or more specific:
interface Props {
  children: React.ReactElement;  // Single element
  children: string;              // Text only
}
```

### Render Props
```tsx
interface Props<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
}

const List = <T,>({ items, renderItem }: Props<T>) => (
  <ul>
    {items.map((item, i) => (
      <li key={i}>{renderItem(item, i)}</li>
    ))}
  </ul>
);
```

## Composition Patterns

### Compound Components
```tsx
// Tabs.tsx
const TabsContext = createContext<TabsContextValue | null>(null);

export const Tabs = ({ children, defaultValue }: TabsProps) => {
  const [activeTab, setActiveTab] = useState(defaultValue);

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
};

Tabs.List = ({ children }: { children: React.ReactNode }) => (
  <div className="tabs-list">{children}</div>
);

Tabs.Tab = ({ value, children }: TabProps) => {
  const { activeTab, setActiveTab } = useContext(TabsContext)!;
  return (
    <button
      className={activeTab === value ? 'active' : ''}
      onClick={() => setActiveTab(value)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = ({ value, children }: PanelProps) => {
  const { activeTab } = useContext(TabsContext)!;
  if (activeTab !== value) return null;
  return <div className="tab-panel">{children}</div>;
};

// Usage
<Tabs defaultValue="tab1">
  <Tabs.List>
    <Tabs.Tab value="tab1">First</Tabs.Tab>
    <Tabs.Tab value="tab2">Second</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="tab1">First content</Tabs.Panel>
  <Tabs.Panel value="tab2">Second content</Tabs.Panel>
</Tabs>
```

### Higher-Order Components (HOC)
```tsx
// Use sparingly - prefer hooks
function withAuth<P extends object>(Component: React.ComponentType<P>) {
  return function AuthenticatedComponent(props: P) {
    const { user, loading } = useAuth();

    if (loading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;

    return <Component {...props} />;
  };
}
```

## Forwarding Refs

```tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', className, children, ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={`btn btn-${variant} ${className ?? ''}`}
        {...props}
      >
        {children}
      </button>
    );
  }
);

Button.displayName = 'Button';
```

## Conditional Rendering

```tsx
// Good - early return
const UserProfile = ({ user }: Props) => {
  if (!user) return null;

  return <div>{user.name}</div>;
};

// Good - ternary for simple cases
const Status = ({ isOnline }: Props) => (
  <span>{isOnline ? 'Online' : 'Offline'}</span>
);

// Good - && for presence check
const Notification = ({ count }: Props) => (
  <>
    {count > 0 && <Badge count={count} />}
  </>
);

// Avoid - nested ternaries
// Bad: {a ? (b ? <X /> : <Y />) : <Z />}
```

## Error Boundaries

```tsx
class ErrorBoundary extends React.Component<Props, State> {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error caught:', error, errorInfo);
    // Log to error reporting service
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <DefaultErrorUI />;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<ErrorMessage />}>
  <RiskyComponent />
</ErrorBoundary>
```
