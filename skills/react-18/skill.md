# React 18 Code Review Skill

## Overview

This skill provides code review guidelines for React 18 applications, focusing on hooks, concurrent features, Suspense, and modern patterns.

## Key Review Areas

### 1. Hooks Best Practices

**Check for proper hook usage:**
- Follow the Rules of Hooks (only call at top level, only in React functions)
- Use appropriate hooks for the use case
- Ensure custom hooks start with `use` prefix
- Check dependency arrays for completeness and correctness

```typescript
// Good: Proper hook usage
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(data => {
      if (!cancelled) {
        setUser(data);
        setLoading(false);
      }
    });
    return () => { cancelled = true; };
  }, [userId]); // Correct dependency

  return { user, loading };
}

// Avoid: Missing dependencies
useEffect(() => {
  doSomething(userId); // userId not in deps
}, []); // ESLint will warn
```

### 2. State Management

**Review state patterns:**
- Use `useState` for simple local state
- Use `useReducer` for complex state logic
- Avoid prop drilling with Context or state libraries
- Consider React Query/TanStack Query for server state

```typescript
// Good: useReducer for complex state
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset'; payload: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    case 'reset': return action.payload;
  }
}

function Counter() {
  const [count, dispatch] = useReducer(reducer, 0);
  // ...
}
```

### 3. Concurrent Features

**Verify proper use of concurrent features:**
- Use `useTransition` for non-urgent updates
- Use `useDeferredValue` for deferring expensive renders
- Understand automatic batching behavior
- Use Suspense boundaries appropriately

```typescript
// Good: useTransition for non-urgent updates
function SearchResults({ query }: { query: string }) {
  const [isPending, startTransition] = useTransition();
  const [results, setResults] = useState<Result[]>([]);

  function handleSearch(query: string) {
    startTransition(() => {
      // This update won't block the input
      setResults(searchData(query));
    });
  }

  return (
    <>
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}

// Good: useDeferredValue
function List({ items }: { items: Item[] }) {
  const deferredItems = useDeferredValue(items);
  // Renders with old items while new ones are computed
  return <ExpensiveList items={deferredItems} />;
}
```

### 4. Suspense and Data Fetching

**Check Suspense patterns:**
- Wrap lazy components with Suspense boundaries
- Use proper fallback UIs
- Consider error boundaries alongside Suspense
- Use data fetching libraries that support Suspense

```typescript
// Good: Suspense with lazy loading
const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Dashboard />
    </Suspense>
  );
}

// Good: Error boundary with Suspense
function DataSection() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Skeleton />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### 5. Performance Optimization

**Review for performance issues:**
- Use `React.memo` for expensive pure components
- Use `useMemo` for expensive computations
- Use `useCallback` for stable function references
- Avoid creating objects/arrays in render

```typescript
// Good: Memoization
const ExpensiveComponent = memo(function ExpensiveComponent({ data }: Props) {
  return <div>{/* expensive render */}</div>;
});

function Parent({ items }: { items: Item[] }) {
  // Memoize expensive computation
  const sortedItems = useMemo(
    () => items.sort((a, b) => a.name.localeCompare(b.name)),
    [items]
  );

  // Stable callback reference
  const handleClick = useCallback((id: string) => {
    selectItem(id);
  }, []);

  return <ExpensiveComponent data={sortedItems} onClick={handleClick} />;
}

// Avoid: Object creation in render
<Component style={{ color: 'red' }} /> // Creates new object every render
```

### 6. Component Patterns

**Verify component best practices:**
- Prefer function components over class components
- Use TypeScript for prop types
- Keep components focused and small
- Extract reusable logic into custom hooks

```typescript
// Good: Typed function component
interface ButtonProps {
  variant: 'primary' | 'secondary';
  children: React.ReactNode;
  onClick?: () => void;
  disabled?: boolean;
}

function Button({ variant, children, onClick, disabled }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
}
```

### 7. Testing

**Verify testing practices:**
- Use React Testing Library for component tests
- Test user behavior, not implementation details
- Mock external dependencies appropriately
- Use `act()` for state updates in tests

```typescript
// Good: Testing user behavior
test('submits form with user input', async () => {
  const onSubmit = jest.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  await userEvent.type(screen.getByLabelText(/email/i), 'test@example.com');
  await userEvent.type(screen.getByLabelText(/password/i), 'password123');
  await userEvent.click(screen.getByRole('button', { name: /submit/i }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123'
  });
});
```

## Common Issues to Flag

1. **Missing or incorrect dependency arrays** in useEffect/useCallback/useMemo
2. **Stale closures** capturing old values
3. **Memory leaks** from uncleared effects
4. **Unnecessary re-renders** from missing memoization
5. **Direct DOM manipulation** instead of using refs
6. **Mutating state directly** instead of using setState
7. **Missing keys** in list rendering
8. **Prop drilling** when Context would be better

## Review Checklist

- [ ] Hooks follow Rules of Hooks
- [ ] Dependency arrays are correct and complete
- [ ] Effects clean up properly
- [ ] State updates are immutable
- [ ] Memoization applied where beneficial
- [ ] Components are properly typed
- [ ] Error boundaries handle failures
- [ ] Tests cover key functionality
