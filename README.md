# Immutable DTOs with Java Records

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

### The Fix: Java Records

Java 21 gives us `record` — immutable by design, with less code than what we have now.

```java
// BEFORE: ~40 lines
public class OrderDTO {
    private String id;
    private List<ItemDTO> items;
    private BigDecimal total;

    public OrderDTO() {}

    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public List<ItemDTO> getItems() { return items; } // 💣 leaked ref
    public void setItems(List<ItemDTO> items) { this.items = items; }
    public BigDecimal getTotal() { return total; }
    public void setTotal(BigDecimal total) { this.total = total; }

    // equals, hashCode, toString...
}

// AFTER: 6 lines
public record OrderDTO(
    String id,
    List<ItemDTO> items,
    BigDecimal total
) {
    public OrderDTO {
        items = List.copyOf(items); // safe forever
    }
}
```

**Less code. No setters. No leaked references. Free `equals`/`hashCode`/`toString`.**

---

### "But we don't have threading issues"

Correct — and this isn't about threads. It's about **local reasoning**.

When a DTO is mutable, every method that touches it is a suspect. You read line 40 and ask: *"could the DTO from line 12 have changed by now?"*

With an immutable DTO, the answer is always **no**. The value you received is the value you have.

This reduces the cognitive load on every developer, every day, in every review.

> (And when someone eventually adds `@Async`, virtual threads, or reactive endpoints — we're already safe.)

---

### FAQ

| Concern | Answer |
|---|---|
| **Spring/Jackson can't deserialize records** | Works since Spring Boot 2.7+ / Jackson 2.12+. We're fine. |
| **We need to build DTOs incrementally** | Use a Builder (Lombok `@Builder` works on records). Immutable once built. |
| **Big refactor, no user-facing change** | Every duplicated DTO is a place where someone updates one copy and forgets the other. That *is* user-facing when it ships a bug. |
| **Too much work** | The migration is mechanical: delete setters, rename class → record, add `List.copyOf` where needed. |

---

### The Proposal

We're already planning to deduplicate the DTOs. While we're touching every one of them:

1. **Consolidate** into a shared module — one source of truth.
2. **Convert** to `record` — less code, not more.
3. **Defensive-copy** collection fields in compact constructors.

We end up with fewer files, fewer lines, fewer bugs, and DTOs that are trivially safe to pass anywhere.
