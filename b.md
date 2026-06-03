# Immutable DTOs — Builder + Jackson

### The Problem We Have Today

We duplicate DTO classes across modules. When one copy is updated, others fall behind — and that causes bugs.

But duplication isn't our only problem. Our DTOs are **mutable**, and that hurts us in two concrete ways **right now**:

---

### 1. Mutation After Construction

```java
var order = new OrderDTO();
order.setId(id);
// ... 30 lines later, in a different service ...
order.setTotal(null); // "temporarily" — forgot to set it back
```

Any method that receives a DTO can't trust its state. You get temporal coupling: code that only works if methods happen to run in the right order. Every `setX` call is a place where a future bug can hide.

---

### 2. Leaked Mutable References

```java
public List<ItemDTO> getItems() {
    return items; // caller can .add(), .clear(), .remove()...
}
```

Anyone calling `getItems()` holds a live reference to the DTO's internals. One accidental `.add()` and the DTO's state is corrupted — silently, without any setter being called.

---

### The Fix: Immutable Classes + Builders

Final fields. No setters. Defensive copies on collections. A builder for ergonomic construction. Jackson wired to deserialize through the builder.

```java
// BEFORE: mutable, fragile, ~40 lines of getters/setters
public class OrderDTO {
    private String id;
    private List<ItemDTO> items;
    private BigDecimal total;

    public OrderDTO() {}

    public void setId(String id) { this.id = id; }
    public void setTotal(BigDecimal total) { this.total = total; }
    public List<ItemDTO> getItems() { return items; } // 💣 leaked ref
    // ... equals, hashCode, toString ...
}

// AFTER: immutable, safe, Jacksonized
@JsonDeserialize(builder = OrderDTO.Builder.class)
public final class OrderDTO {

    private final String id;
    private final List<ItemDTO> items;
    private final BigDecimal total;

    private OrderDTO(Builder b) {
        this.id = b.id;
        this.items = b.items == null ? List.of() : List.copyOf(b.items);
        this.total = b.total;
    }

    public String getId()          { return id; }
    public List<ItemDTO> getItems() { return items; }  // safe — already unmodifiable
    public BigDecimal getTotal()   { return total; }

    public Builder toBuilder() {
        return new Builder().id(id).items(items).total(total);
    }

    @JsonPOJOBuilder(withPrefix = "")
    public static final class Builder {
        private String id;
        private List<ItemDTO> items;
        private BigDecimal total;

        public Builder id(String id)              { this.id = id; return this; }
        public Builder items(List<ItemDTO> items)  { this.items = items; return this; }
        public Builder total(BigDecimal total)     { this.total = total; return this; }

        // convenience — add a single item without building a list yourself
        public Builder item(ItemDTO item) {
            if (this.items == null) this.items = new ArrayList<>();
            this.items.add(item);
            return this;
        }

        public OrderDTO build() { return new OrderDTO(this); }
    }
}
```

**What this gives us:**

- `List.copyOf()` at construction → getter returns an unmodifiable list. No leaked references, ever.
- `List.of()` as default → no nulls, no `if (getItems() != null)` guards downstream.
- `toBuilder()` → need a modified copy? `order.toBuilder().total(newTotal).build()`. Original untouched.
- Single `item()` method on builder → no need to pre-build a list for the common one-item case.
- `@JsonDeserialize` + `@JsonPOJOBuilder` → Jackson deserializes through the builder. Spring works out of the box.

---

### "But we don't have threading issues"

Correct — and this isn't about threads. It's about **local reasoning**.

When a DTO is mutable, every method that touches it is a suspect. You read line 40 and ask: *"could the DTO from line 12 have changed by now?"*

With an immutable DTO, the answer is always **no**. The value you received is the value you have.

This reduces the cognitive load on every developer, every day, every review.

> (And when someone eventually adds `@Async`, virtual threads, or reactive endpoints — we're already safe.)

---

### Patterns at a Glance

| Pattern | How |
|---|---|
| **Empty collection default** | `b.items == null ? List.of() : List.copyOf(b.items)` |
| **Singleton / single-element list** | `Builder.item(singleItem)` convenience method |
| **Need a modified copy** | `dto.toBuilder().field(newVal).build()` |
| **Jackson deserialization** | `@JsonDeserialize(builder = Dto.Builder.class)` + `@JsonPOJOBuilder(withPrefix = "")` |
| **Spring `@RequestBody`** | Works — Jackson handles it via the builder |
| **Null safety on collections** | Constructor guarantees non-null; callers never check |

---

### FAQ

| Concern | Answer |
|---|---|
| **Too much boilerplate vs mutable class** | It's ~the same line count once you delete setters, and the builder replaces the no-arg constructor + set-set-set pattern. Lombok `@Builder` + `@Jacksonized` can cut it further if we want. |
| **What if I need to "update" a DTO?** | `toBuilder()` — immutable original, new copy with changes. |
| **Performance — copying lists?** | `List.copyOf()` on an already-unmodifiable list returns the same instance (no copy). In practice the cost is near zero. |
| **Existing code passes DTOs around and mutates them** | That's exactly the class of bug we're eliminating. Compiler errors from the migration *are the audit* — every error shows a place where we were silently depending on mutation. |

---

### The Proposal

We're already planning to deduplicate the DTOs. While we're touching every one of them:

1. **Consolidate** into a shared module — one source of truth.
2. **Convert** to immutable classes with builders — final fields, `List.copyOf`, `List.of` defaults.
3. **Wire Jackson** via `@JsonDeserialize` / `@JsonPOJOBuilder`.
4. **Fix compile errors** — each one reveals a mutation site we need to clean up.

We end up with fewer files, safer code, and DTOs that are trivially safe to pass anywhere.
