# Why Records, Not Builders

### We agree on the goal

Immutable DTOs, shared in one place, no more mutation after construction, no more leaked references. The question is **how** — hand-written immutable classes with builders, or Java records.

---

### Side by side

```java
// HAND-WRITTEN IMMUTABLE CLASS: ~45 lines
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

    public String getId()           { return id; }
    public List<ItemDTO> getItems() { return items; }
    public BigDecimal getTotal()    { return total; }

    public Builder toBuilder() {
        return new Builder().id(id).items(items).total(total);
    }

    @JsonPOJOBuilder(withPrefix = "")
    public static final class Builder {
        private String id;
        private List<ItemDTO> items;
        private BigDecimal total;

        public Builder id(String id)             { this.id = id; return this; }
        public Builder items(List<ItemDTO> items) { this.items = items; return this; }
        public Builder total(BigDecimal total)    { this.total = total; return this; }
        public OrderDTO build() { return new OrderDTO(this); }
    }
}

// RECORD: 9 lines — same guarantees
@Jacksonized
@Builder(toBuilder = true)
public record OrderDTO(
    String id,
    List<ItemDTO> items,
    BigDecimal total
) {
    public OrderDTO {
        items = items == null ? List.of() : List.copyOf(items);
    }
}
```

Same builder. Same Jackson support. Same `toBuilder()`. Same defensive copy.  
**80% less code.** And `equals`, `hashCode`, `toString` are free.

---

### The real difference: who enforces immutability?

| | Hand-Written Class | Record |
|---|---|---|
| Can someone add a setter? | Yes — nothing stops them | No — **compiler rejects it** |
| Can someone make it non-final? | Yes — just remove `final` | No — records are final by spec |
| Can someone skip `List.copyOf`? | Yes — it's convention | Yes — same risk, but fewer places to forget |
| Can someone add mutable state? | Yes — add a non-final field | No — all fields are final by spec |
| `equals` / `hashCode` drift? | Yes — someone edits fields, forgets to update | No — always derived from components |

With hand-written classes, immutability is a **team agreement**. With records, it's a **language guarantee**. Six months from now, under deadline pressure, the agreement erodes. The guarantee doesn't.

---

### "But records can't do X"

| Concern | Reality |
|---|---|
| **No inheritance** | DTOs shouldn't use inheritance anyway — composition or interfaces. If one truly needs it, make *that one* a hand-written class. |
| **Accessors are `id()` not `getId()`** | Jackson, Spring, MapStruct all handle this. If internal tooling breaks, that tooling was relying on reflection hacks we should fix anyway. |
| **Can't have a private field** | A DTO's entire purpose is carrying data transparently. If you need hidden state, it's not a DTO. |
| **Lombok on records feels odd** | `@Jacksonized @Builder` on a record is 2 annotations vs 45 lines of hand-written builder. The oddness fades fast. |

---

### The proposal

- **Default:** records for all DTOs. Covers ~90% of cases.
- **Exception:** hand-written immutable class with builder, only when a DTO genuinely needs inheritance or hidden fields. Flag these explicitly in PR review.
- **No grey area:** if it's in the shared DTO module, it's a record unless there's a documented reason.

We get compiler-enforced immutability, drastically less code, and one clear rule instead of a style guide nobody re-reads.
