# Angular 19 Code Review Skill

## Overview

This skill provides code review guidelines for Angular 19 applications, focusing on modern patterns including signals, standalone components, and the new control flow syntax.

## Key Review Areas

### 1. Signal-Based Reactivity

**Check for proper signal usage:**
- Use `signal()` for reactive state instead of BehaviorSubject where appropriate
- Use `computed()` for derived values
- Use `effect()` sparingly and only for side effects
- Prefer `input()` and `output()` signal-based component APIs

```typescript
// Good: Signal-based state
private count = signal(0);
private doubled = computed(() => this.count() * 2);

// Avoid: BehaviorSubject for simple state
private count$ = new BehaviorSubject(0);
```

### 2. Standalone Components

**Verify standalone component best practices:**
- All new components should be standalone (`standalone: true`)
- Import dependencies directly in component decorator
- Avoid NgModules for new code unless necessary for backward compatibility

```typescript
// Good: Standalone component
@Component({
  selector: 'app-example',
  standalone: true,
  imports: [CommonModule, RouterModule],
  template: `...`
})
export class ExampleComponent {}
```

### 3. New Control Flow Syntax

**Ensure usage of new template syntax:**
- Use `@if` instead of `*ngIf`
- Use `@for` instead of `*ngFor` with required `track` expression
- Use `@switch` instead of `*ngSwitch`
- Use `@defer` for lazy loading content

```html
<!-- Good: New control flow -->
@if (isLoading()) {
  <app-spinner />
} @else {
  @for (item of items(); track item.id) {
    <app-item [data]="item" />
  }
}

<!-- Avoid: Legacy structural directives -->
<app-spinner *ngIf="isLoading$ | async"></app-spinner>
```

### 4. Dependency Injection

**Review DI patterns:**
- Use `inject()` function instead of constructor injection
- Prefer functional guards and resolvers
- Use `providedIn: 'root'` for singleton services

```typescript
// Good: inject() function
export class MyComponent {
  private readonly http = inject(HttpClient);
  private readonly route = inject(ActivatedRoute);
}

// Good: Functional guard
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.isAuthenticated();
};
```

### 5. HTTP and RxJS

**Check HTTP client usage:**
- Use `HttpClient` with proper error handling
- Consider `toSignal()` for converting observables to signals
- Use `takeUntilDestroyed()` for subscription management

```typescript
// Good: Observable to signal conversion
private users = toSignal(this.http.get<User[]>('/api/users'), {
  initialValue: []
});

// Good: Automatic cleanup
ngOnInit() {
  this.data$.pipe(
    takeUntilDestroyed(this.destroyRef)
  ).subscribe(data => this.processData(data));
}
```

### 6. Performance Patterns

**Review for performance issues:**
- Check for proper use of `OnPush` change detection strategy
- Verify `trackBy` (or `track` in new syntax) is used in loops
- Look for unnecessary subscriptions or computations
- Check for proper use of `@defer` for lazy loading

### 7. Testing

**Verify testing practices:**
- Components should have corresponding `.spec.ts` files
- Use `TestBed.configureTestingModule` with standalone component imports
- Mock signals using `signal()` in tests
- Test computed values and effects appropriately

## Common Issues to Flag

1. **Missing track expression** in `@for` loops
2. **Zone pollution** from unnecessary async operations
3. **Memory leaks** from unmanaged subscriptions
4. **Improper signal usage** (calling signals without parentheses)
5. **Legacy patterns** when modern alternatives exist
6. **Missing error handling** in HTTP calls
7. **Over-complicated state management** when signals suffice

## Review Checklist

- [ ] Components are standalone
- [ ] Signals used for reactive state
- [ ] New control flow syntax used
- [ ] inject() function preferred over constructor DI
- [ ] Proper subscription management
- [ ] Performance optimizations applied
- [ ] Tests cover component functionality
