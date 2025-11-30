# Hooks Patterns

## Rules of Hooks

1. Only call hooks at the top level (no conditions, loops, nested functions)
2. Only call hooks from React functions (components or custom hooks)
3. Name custom hooks with `use` prefix

## useState

```tsx
// Simple state
const [count, setCount] = useState(0);

// With TypeScript - type inference
const [user, setUser] = useState<User | null>(null);

// Lazy initialization (expensive computation)
const [data, setData] = useState(() => computeExpensiveValue());

// Functional updates (when new state depends on previous)
setCount((prev) => prev + 1);

// Object state - always spread
setUser((prev) => prev ? { ...prev, name: 'New Name' } : null);
```

## useEffect

```tsx
// Run on every render
useEffect(() => {
  console.log('Rendered');
});

// Run once on mount
useEffect(() => {
  fetchData();
}, []);

// Run when dependencies change
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// Cleanup
useEffect(() => {
  const subscription = subscribe(channel);
  return () => subscription.unsubscribe();
}, [channel]);

// Async in useEffect
useEffect(() => {
  const fetchData = async () => {
    const result = await api.getUsers();
    setUsers(result);
  };
  fetchData();
}, []);
```

## useCallback and useMemo

```tsx
// useCallback - memoize functions
const handleClick = useCallback((id: string) => {
  selectItem(id);
}, [selectItem]);

// useMemo - memoize expensive computations
const sortedItems = useMemo(() => {
  return items.sort((a, b) => a.name.localeCompare(b.name));
}, [items]);

// When to use:
// - useCallback: When passing callbacks to optimized child components
// - useMemo: When computation is expensive OR value used in dependency arrays
// - Don't overuse - React is fast, premature optimization costs readability
```

## useRef

```tsx
// DOM reference
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => {
  inputRef.current?.focus();
}, []);

// Mutable value that doesn't trigger re-render
const countRef = useRef(0);
countRef.current += 1;  // No re-render

// Previous value pattern
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

## useContext

```tsx
// Create context with type
interface ThemeContextValue {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | null>(null);

// Custom hook for consuming
function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// Provider component
export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = useCallback(() => {
    setTheme((t) => (t === 'light' ? 'dark' : 'light'));
  }, []);

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## Custom Hooks

```tsx
// Data fetching hook
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;

    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const result = await response.json();
        if (!cancelled) setData(result);
      } catch (e) {
        if (!cancelled) setError(e as Error);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchData();
    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Local storage hook
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
}

// Toggle hook
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue((v) => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse };
}
```

## useReducer

```tsx
interface State {
  count: number;
  error: string | null;
}

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' }
  | { type: 'setError'; payload: string };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 };
    case 'decrement':
      return { ...state, count: state.count - 1 };
    case 'reset':
      return { count: 0, error: null };
    case 'setError':
      return { ...state, error: action.payload };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, error: null });

  return (
    <>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </>
  );
}
```
