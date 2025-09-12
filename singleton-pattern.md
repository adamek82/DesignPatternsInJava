# The Singleton Pattern in Java

## What & Why

A **Singleton** ensures **exactly one** instance of a class exists in a JVM **and** provides a global access point to it.  
Use it when:
- there must be a single coordinator for shared resources (e.g., a logger, metrics registry, configuration),
- creating multiple instances would break correctness or waste resources (e.g., connection pools).

> Important: “one per JVM” is the practical guarantee. With multiple class loaders (app servers, plugins) you can accidentally get one-per-loader. See **Caveats**.

---

## Canonical Implementations (Modern Java)

### 1) Eager initialization (simple & safe)

```java
public final class AppSingleton {
    private static final AppSingleton INSTANCE = new AppSingleton();
    private AppSingleton() {}
    public static AppSingleton getInstance() { return INSTANCE; }
}
```

**Pros:** trivial, thread-safe by class initialization semantics; `final` helps safe publication.  
**Cons:** created even if never used; not suitable if you need lazy/deferred setup.

---

### 2) Lazy with `synchronized` (simple, but may contend)

```java
public final class LazySingleton {
    private static LazySingleton instance;
    private LazySingleton() {}
    public static synchronized LazySingleton getInstance() {
        if (instance == null) instance = new LazySingleton();
        return instance;
    }
}
```

**Pros:** simple, lazily initialized.  
**Cons:** `synchronized` on every call (often OK, but can be a hot path).

---

### 3) Double-Checked Locking (DCL) — fast & lazy

```java
public final class DclSingleton {
    private static volatile DclSingleton instance;
    private DclSingleton() {}

    public static DclSingleton getInstance() {
        DclSingleton result = instance;          // local read (fast path)
        if (result == null) {                    // first check (no lock)
            synchronized (DclSingleton.class) {
                result = instance;
                if (result == null) {
                    result = instance = new DclSingleton();
                }
            }
        }
        return result;
    }
}
```

**Why `volatile`:** prevents reordering that could publish a partially constructed object and ensures visibility across threads.  
**Pros:** no lock after initialization.  
**Cons:** more complex; easy to get wrong without `volatile`.

---

### 4) Initialization-on-Demand Holder (Idiom) — recommended lazy

```java
public final class HolderSingleton {
    private HolderSingleton() {}

    private static class Holder {
        static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Pros:** lazy, thread-safe, clean; relies on class-loader semantics.  
**Cons:** none for most cases.

---

### 5) Enum Singleton — robust against serialization & reflection

```java
public enum EnumSingleton {
    INSTANCE;
    public void doSomething() { /* ... */ }
}
```

**Pros:** simplest robust variant; safe vs serialization and reflective instantiation.  
**Cons:** cannot extend other classes (enums already extend `java.lang.Enum`); not great if you need lazy complex construction logic.

---

## A “classic static field” variant as a curiosity

This style pre-creates a single instance in a `static` field and exposes it via a static accessor:

```java
public final class StaticFieldSingleton {
    private static final StaticFieldSingleton INSTANCE = new StaticFieldSingleton();
    private StaticFieldSingleton() {}
    public static StaticFieldSingleton access() { return INSTANCE; }
}
```

Functionally similar to “eager initialization”; shown here because many older codebases still use it.

---

## Realistic Use Cases

### A) Logger (global, lightweight, threadsafe)

```java
public enum AppLogger {
    INSTANCE;

    public void info(String msg) {
        System.out.println("[INFO] " + msg);
    }
    public void error(String msg, Throwable t) {
        System.err.println("[ERROR] " + msg);
        if (t != null) t.printStackTrace(System.err);
    }
}

// Usage:
AppLogger.INSTANCE.info("Application started");
```

Why Singleton? loggers are stateless coordinators. You want one global sink to keep output ordering and configuration consistent.

### B) Configuration / Settings Registry

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class Config {
    private final Map<String,String> kv = new ConcurrentHashMap<>();
    private Config() {}
    private static class Holder { static final Config INSTANCE = new Config(); }
    public static Config get() { return Holder.INSTANCE; }

    public void put(String k, String v) { kv.put(k, v); }
    public String get(String k) { return kv.get(k); }
}
```

Why Singleton? avoids scattering configuration state and ensures consistent view across modules.

### C) Connection Pool (wrapper/facade)

```java
public final class ConnectionPools {
    private final DbPool mainPool;
    private ConnectionPools() { this.mainPool = new DbPool(10); } // size=10

    private static class Holder { static final ConnectionPools I = new ConnectionPools(); }
    public static ConnectionPools get() { return Holder.I; }

    public DbConn borrow() { return mainPool.borrow(); }
    public void release(DbConn c) { mainPool.release(c); }
}
```

Why Singleton? pools are expensive; a single, shared pool coordinates resource usage.

---

## Serialization & Reflection Concerns (and fixes)

### 1) Serialization can create a new instance
If a singleton is `Serializable`, naïve deserialization produces **a new object**:

```java
public final class SerSingleton implements java.io.Serializable {
    private static final SerSingleton INSTANCE = new SerSingleton();
    private SerSingleton() {}
    public static SerSingleton get() { return INSTANCE; }
    // FIX: funnel deserialization back to the sole instance
    private Object readResolve() throws java.io.ObjectStreamException {
        return INSTANCE;
    }
}
```

### 2) Reflection can call the private constructor
Reflection can bypass `private` via `setAccessible(true)` and instantiate a second object.  
**Mitigations:**
- Add a constructor guard:
  ```java
  public final class GuardedSingleton {
      private static boolean created = false;
      private static final GuardedSingleton INSTANCE = new GuardedSingleton();
      private GuardedSingleton() {
          if (created) throw new IllegalStateException("Use getInstance()");
          created = true;
      }
      public static GuardedSingleton get() { return INSTANCE; }
  }
  ```
  (Not bulletproof against all reflective hacks, but raises the bar.)
- Prefer **enum singleton**: the JVM forbids reflective enum instantiation and handles serialization correctly by design.

---

## Concurrency: Synchronization vs Visibility vs Ordering

- **Synchronization** controls **who** enters critical sections (mutual exclusion).  
- **Visibility** ensures threads see **up-to-date** values (use `volatile`, `synchronized`, or final fields set in constructors).  
- **Ordering**: CPUs/JIT may **reorder** operations. `volatile` and entering/exiting a synchronized block impose the necessary memory barriers.

DCL relies on `volatile` to prevent publishing a partially constructed instance due to reordering.

---

## Caveats & Testing

- **Multiple class loaders** can yield one instance per loader. Be mindful on app servers or plugin systems.  
- **Global state** complicates tests. Provide reset hooks only in test builds or avoid singletons in domain logic (use dependency injection).  
- **Mutable singletons** still need proper concurrency control for their internal state (use thread-safe collections or synchronization).

---

## Quick Decision Guide

- Need the safest, simplest solution? **Enum singleton.**  
- Need lazy init with simple code? **Holder idiom.**  
- Need lazy init and ultra-hot access path? **DCL with `volatile`** (only if truly needed).  
- OK with eager creation? **Eager/static-field** variant.

---

## Minimal End-to-End Example

```java
public enum AppServices {  // robust singleton
    INSTANCE;

    private final Config config = Config.get(); // holder-based singleton
    public void start() {
        AppLogger.INSTANCE.info("Starting with DB=" + config.get("db.url"));
        // ...
    }
}
```

This combines an enum singleton (robust entrypoint), a holder-based configuration, and a global logger.

---

## Summary

A Singleton centralizes **a single shared instance**. In modern Java you should prefer:
- **Enum singleton** for robustness,
- **Holder idiom** for clean lazy init,
- **DCL + volatile** only when micro-performance matters,
- **Eager/static** when simplicity beats laziness.

Always consider testing & modularity; don’t let a singleton become hidden global state that’s hard to reason about.
