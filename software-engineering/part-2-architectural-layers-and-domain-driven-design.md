---
title: Part 2 - Thw How - Architectural Layers & Domain-Driven Design
date: 2026-09-20
author: Jorge Castillo
published: false
---

In **Part 1**, we established *why* modern React applications degrade over time: unmanaged dependencies, framework lock-in, and the "Fat Component" anti-pattern. The remedy is the **Dependency Rule**—ensuring core business logic never depends on volatile outer infrastructure like UI frameworks, state management libraries, or REST API endpoints.  

Now, we move from principle to execution. How do we physically structure a React project to enforce these boundaries without devolving into folder chaos?  

In this second installment, we break down the modular architecture of our reference repository, [React-Clean-Architecture](https://github.com/schorts99/React-Clean-Architecture), exploring how to model domain entities, enforce boundary constraints, orchestrate application use cases, and implement resilient infrastructure adapters.  

## 📁 Feature-First, Module-Oriented Directory Structure

Instead of grouping code by technical role (`/components`, `/hooks`, `/services`), Clean Architecture organizes the codebase into **domain modules**. Each module represents a distinct business boundary (a Bounded Context in Domain-Driven Design terminology) and encapsulates its own three-layer architecture.  

```text
src/
├── modules/
│   └── shopping-cart/             <-- Bounded Context / Domain Module
│       ├── domain/                <-- Pure TypeScript (Zero External Dependencies)
│       │   ├── entities/          <-- Cart.ts, CartItem.ts
│       │   ├── value-objects/     <-- Money.ts, Quantity.ts
│       │   └── errors/            <-- InvalidQuantityError.ts
│       │
│       ├── application/           <-- Use Cases & Workflow Orchestration
│       │   ├── commands/          <-- AddItemToCartCommand.ts
│       │   ├── queries/           <-- GetCartSummaryQuery.ts
│       │   └── ports/             <-- ICartRepository.ts (Primary & Secondary Ports)
│       │
│       ├── infrastructure/        <-- Outer Drivers & Technology Implementations
│       │   ├── adapters/          <-- ApiCartRepository.ts, IndexedDBCartRepository.ts
│       │   └── mappers/           <-- CartMapper.ts
│       │
│       └── presentation/          <-- React UI Components & Custom Hooks
│           ├── components/        <-- CartView.tsx, CheckoutButton.tsx
│           └── hooks/             <-- useCartViewModel.ts
```

By enforcing this structure, any engineer working inside `/domain` or `/application` can immediately see that importing anything from `/presentation` or `/infrastructure` violates the architecture.  

## 💎 Layer 1: The Domain Layer (Core Business Invariants)

The **Domain Layer** sits at the absolute center of your application. It contains no React imports, no `fetch` calls, and no browser storage references. It is composed entirely of pure TypeScript primitives, **Entities**, and **Value Objects**.  

### 1. Value Objects

Value Objects are immutable types defined entirely by their attributes rather than an identity. They encapsulate validation rules, type safety, and domain calculations.  

```ts
// domain/value-objects/Quantity.ts
export class Quantity {
  private constructor(readonly value: number) {
    if (value <= 0 || !Number.isInteger(value)) {
      throw new Error("Quantity must be a positive integer.");
    }
  }

  static create(value: number): Quantity {
    return new Quantity(value);
  }

  add(other: Quantity): Quantity {
    return new Quantity(this.value + other.value);
  }
}

// domain/value-objects/Money.ts
export class Money {
  private constructor(readonly amount: number, readonly currency: string) {
    if (amount < 0) {
      throw new Error("Money amount cannot be negative.");
    }
  }

  static create(amount: number, currency = "USD"): Money {
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch.");
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }
}
```

## 2. Domain Entities

Entities possess a unique identity (`id`) that persists across state changes. They enforce business rules (invariants) whenever state mutations occur.  

```ts
// domain/entities/CartItem.ts
export class CartItem {
  private constructor(
    readonly productId: string,
    readonly unitPrice: Money,
    private _quantity: Quantity
  ) {}

  static create(productId: string, unitPrice: Money, quantity: number): CartItem {
    return new CartItem(productId, unitPrice, Quantity.create(quantity));
  }

  get quantity(): Quantity {
    return this._quantity;
  }

  get totalPrice(): Money {
    return this.unitPrice.multiply(this._quantity.value);
  }

  increaseQuantity(additional: Quantity): CartItem {
    return new CartItem(
      this.productId,
      this.unitPrice,
      this._quantity.add(additional)
    );
  }
}

// domain/entities/Cart.ts
export class Cart {
  private constructor(
    readonly id: string,
    private _items: CartItem[],
    private _discountCode?: string
  ) {}

  static create(id: string, items: CartItem[] = []): Cart {
    return new Cart(id, items);
  }

  get items(): readonly CartItem[] {
    return [...this._items];
  }

  // Business Invariant: Business rules computed directly inside domain boundary
  calculateTotal(): Money {
    const subtotal = this._items.reduce(
      (sum, item) => sum.add(item.totalPrice),
      Money.create(0)
    );
    return subtotal;
  }

  addItem(newItem: CartItem): void {
    const existingIndex = this._items.findIndex((item) => item.productId === newItem.productId);
    if (existingIndex >= 0) {
      this._items[existingIndex] = this._items[existingIndex].increaseQuantity(newItem.quantity);
    } else {
      this._items.push(newItem);
    }
  }
}
```

## ⚙️ Layer 2: The Application Layer (CQRS & Ports)

The **Application Layer** orchestrates use cases. It tells the domain entities what to do and communicates with outer systems using abstract interfaces called **Ports**.  

To keep write operations distinct from read operations, we use a lightweight **CQRS (Command Query Responsibility Segregation)** pattern:  

- **Commands**: Intentional actions that alter application state (e.g., `AddItemToCartCommand`).
- **Queries**: Read-only requests that retrieve formatted data for the view without side effects (e.g., `GetCartSummaryQuery`).

### 1. Defining Application Ports

The application layer defines what data access services it requires without caring how they are implemented.  

```ts
// application/ports/ICartRepository.ts
export interface ICartRepository {
  getById(cartId: string): Promise<Cart | null>;
  save(cart: Cart): Promise<void>;
}

// application/ports/IProductCatalogService.ts
export interface IProductCatalogService {
  getPrice(productId: string): Promise<Money>;
}
```

### 2. Implementing Command & Query Handlers

Command handlers execute state mutations, while Query handlers project data directly for consumption.  

```ts
// application/commands/AddItemToCartHandler.ts
export class AddItemToCartHandler {
  constructor(
    private readonly cartRepository: ICartRepository,
    private readonly catalogService: IProductCatalogService
  ) {}

  async execute(command: { cartId: string; productId: string; quantity: number }): Promise<void> {
    const cart = await this.cartRepository.getById(command.cartId);
    if (!cart) {
      throw new Error("Cart not found.");
    }

    const price = await this.catalogService.getPrice(command.productId);
    const item = CartItem.create(command.productId, price, command.quantity);
    
    cart.addItem(item); // Enforces internal domain rules

    await this.cartRepository.save(cart); // Persists via abstract port
  }
}
```

## 🔌 Layer 3: The Infrastructure Layer (Adapters & Mappers)

The **Infrastructure Layer** fulfills the abstract contracts defined by the application layer. It handles network protocol execution, IndexedDB caching, or third-party client integrations.  

### Data Mappers: Isolating API Schemas

External API payload formats should never contaminate inner domain entities. **Mappers** explicitly transform raw Data Transfer Objects (DTOs) into clean domain objects.  

```ts
// infrastructure/mappers/CartMapper.ts
export interface ApiCartResponseDTO {
  cart_id: string;
  line_items: Array<{
    product_id: string;
    unit_amount: number;
    currency_code: string;
    qty: number;
  }>;
}

export class CartMapper {
  static toDomain(raw: ApiCartResponseDTO): Cart {
    const items = raw.line_items.map((item) =>
      CartItem.create(
        item.product_id,
        Money.create(item.unit_amount, item.currency_code),
        item.qty
      )
    );
    return Cart.create(raw.cart_id, items);
  }

  static toDTO(cart: Cart): Record<string, unknown> {
    return {
      cart_id: cart.id,
      line_items: cart.items.map((item) => ({
        product_id: item.productId,
        unit_amount: item.unitPrice.amount,
        currency_code: item.unitPrice.currency,
        qty: item.quantity.value,
      })),
    };
  }
}
```

## 🖥️ Layer 4: The Presentation Layer (Thin React Views)

With domain logic and use case orchestration safely encapsulated in inner layers, UI components become extraordinarily clean. React components focus entirely on presentation, event listening, and rendering.  

```tsx
// presentation/hooks/useAddToCartViewModel.ts
export const useAddToCartViewModel = () => {
  const [isLoading, setIsLoading] = useState(false);
  // Inject handler via container/context in production
  const handler = useDependency<AddItemToCartHandler>("AddItemToCartHandler");

  const executeAddToCart = async (params: { cartId: string; productId: string; quantity: number }) => {
    setIsLoading(true);
    try {
      await handler.execute(params);
    } finally {
      setIsLoading(false);
    }
  };

  return { executeAddToCart, isLoading };
};

// presentation/components/AddToCartButton.tsx
export const AddToCartButton = ({ cartId, productId }: { cartId: string; productId: string }) => {
  const { executeAddToCart, isLoading } = useAddToCartViewModel();

  const handleClick = () => {
    executeAddToCart({ cartId, productId, quantity: 1 });
  };

  return (
    <button onClick={handleClick} disabled={isLoading}>
      {isLoading ? "Adding..." : "Add to Cart"}
    </button>
  );
};
```

## 📊 Summary of Architectural Responsibilities

| Layer              | Responsibility                                              | Allowed Dependencies        |
|--------------------|-------------------------------------------------------------|-----------------------------|
| **Domain**         | Modeling core business rules, Entities, & Value Objects.    | None (Pure TypeScript)      |
| **Application**    | Use-case orchestration (CQRS) & Port interface definitions. | Domain Layer only           |
| **Infrastructure** | Database drivers, HTTP adapters, and DTO Mappers.           | Domain & Application Layers |
| **Presentation**   | React components, UI state, rendering hooks, & views.       | Application & Domain Layers |

## 📢 Up Next...

Now that we have separated our codebase into domain entities, application ports, adapters, and thin UI views, how do we wire all these dependencies together dynamically at runtime without manually instantiating classes everywhere?  

**Coming up next in Part 3: Advanced Frontend Engineering Patterns**. We will explore Dependency Injection with InversifyJS, Composition Roots for browser vs. SSR environments, tag-based cache invalidation, and lightning-fast isolated testing strategies.  

Follow along in the reference implementation on GitHub:  

👉 [github.com/schorts99/React-Clean-Architecture](https://github.com/schorts99/React-Clean-Architecture)
