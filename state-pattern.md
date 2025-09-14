# The State Pattern in Java — Practical Guide for Working Engineers

> Let an object change its behavior when its internal **state** changes. The object will appear to change its class. — GoF

This guide shows how to implement **State** cleanly in modern Java with practical, programmer‑day‑to‑day examples (DB connections and files). It also includes variants using **enum** and **sealed interfaces** (Java 17+), trade‑offs, and testing notes.


---

## 1) What Problem Does State Solve?

When behavior depends on an internal **mode** (state), code often devolves into sprawling conditionals:

```java
void executeQuery(String sql) {
    if (state == CONNECTED) {
        // run
    } else if (state == DISCONNECTED) {
        throw new IllegalStateException("Not connected");
    } else if (state == CONNECTING) {
        // ...
    } else if (state == ERROR) {
        // ...
    }
}
```

That “big if/switch” scatters across many methods and becomes hard to evolve.

**State Pattern** replaces those conditionals with **polymorphism**:

- Model each **state** as its **own type**.
- The context holds a reference to the **current state** and **delegates** behavior to it.
- To change behavior, you **swap the state object**, not `if/else` branches.


### Core Class Diagram (ASCII)

```
+-----------------+         holds a reference to        +-----------------+
|     Context     |------------------------------------>|     State       |
|-----------------|                                     |-----------------|
| - state: State  |                                     | + operation()   | <- interface / abstract
| + operation()   |                                     +-----------------+
| + setState(..)  |                                               ^
+-----------------+                                               |
                                                                  |
                             +--------------------+---------------+-------------------+
                             |                    |                                   |
                      +---------------+    +---------------+                 +---------------+
                      |  StateA       |    |  StateB       |                 |  StateC       |
                      | + operation() |    | + operation() |                 | + operation() |
                      +---------------+    +---------------+                 +---------------+
```


---

## 2) “Hello, State” in 30 Seconds (refining the classic example)

*Note:* This is a deliberately simple and not very realistic example — its purpose is just to give a first impression of the core concepts. The example originates from Bruce Eckel’s *Thinking in Java* and has been adapted here.


A minimal example: a `Stage` delegates to an `Actor`. Changing the `Actor` at runtime changes behavior.

```java
// Abstract base (could be an interface)
abstract class Actor { public abstract void act(); }

class HappyActor extends Actor { public void act() { System.out.println("HappyActor"); } }
class SadActor   extends Actor { public void act() { System.out.println("SadActor");   } }

class Stage {
    private Actor actor = new HappyActor();          // initial state
    public void change() { actor = new SadActor(); } // state swap
    public void performPlay() { actor.act(); }       // delegation
}

public class Transmogrify {
    public static void main(String[] args) {
        Stage s = new Stage();
        s.performPlay();  // HappyActor
        s.change();
        s.performPlay();  // SadActor
    }
}
```

> The key idea: **no `if/else`** in `Stage`; it **delegates** to the current `Actor`.


---

## 3) Real‑World Example A — Database Connection

We’ll implement **two states**: `DISCONNECTED` and `CONNECTED`.  
We’ll **not** create new state objects each time — states are **singletons** (stateless behavior), which is idiomatic and allocation‑free.

### 3.1 Interface + Final Singleton States

```java
// ---- State API ----
interface ConnectionState {
    void executeQuery(DatabaseConnection db, String sql);
    void connect(DatabaseConnection db);
    void disconnect(DatabaseConnection db);
    String name();
}

// ---- Concrete states (singletons) ----
final class DisconnectedState implements ConnectionState {
    public static final DisconnectedState INSTANCE = new DisconnectedState();
    private DisconnectedState() {}

    @Override public void executeQuery(DatabaseConnection db, String sql) {
        throw new IllegalStateException("Cannot execute: not connected");
    }
    @Override public void connect(DatabaseConnection db) {
        System.out.println("Connecting...");
        db.setState(ConnectedState.INSTANCE);
    }
    @Override public void disconnect(DatabaseConnection db) {
        System.out.println("Already disconnected.");
    }
    @Override public String name() { return "DISCONNECTED"; }
}

final class ConnectedState implements ConnectionState {
    public static final ConnectedState INSTANCE = new ConnectedState();
    private ConnectedState() {}

    @Override public void executeQuery(DatabaseConnection db, String sql) {
        System.out.println("Executing SQL: " + sql);
        // Real DB work would go here.
    }
    @Override public void connect(DatabaseConnection db) {
        System.out.println("Already connected.");
    }
    @Override public void disconnect(DatabaseConnection db) {
        System.out.println("Disconnecting...");
        db.setState(DisconnectedState.INSTANCE);
    }
    @Override public String name() { return "CONNECTED"; }
}

// ---- Context ----
class DatabaseConnection {
    private ConnectionState state = DisconnectedState.INSTANCE;

    void setState(ConnectionState s) { this.state = s; }
    public String state() { return state.name(); }

    public void executeQuery(String sql) { state.executeQuery(this, sql); }
    public void connect() { state.connect(this); }
    public void disconnect() { state.disconnect(this); }
}

// ---- Demo ----
public class StatePatternRealLife {
    public static void main(String[] args) {
        DatabaseConnection db = new DatabaseConnection();

        System.out.println("State: " + db.state());
        try {
            db.executeQuery("SELECT * FROM users"); // -> error
        } catch (IllegalStateException ex) {
            System.out.println("Caught error: " + ex.getMessage());
        }

        db.connect();                                // -> Connecting...
        System.out.println("State: " + db.state());
        db.executeQuery("SELECT * FROM users");      // -> Executing...
        db.disconnect();                             // -> Disconnecting...
        System.out.println("State: " + db.state());
    }
}
```

**Why singletons?** These state objects are **stateless**; there’s no benefit in allocating new ones per transition.  
**Why `final`?** To **close** the implementation and protect invariants; it also helps JIT inlining.  
**Why an interface?** We want a **contract** without shared implementation; default methods remain available if needed.


### 3.2 Enum Variant — compact, naturally singleton

```java
enum ConnectionStateEnum {
    DISCONNECTED {
        @Override void executeQuery(DatabaseConnectionEnum db, String sql) {
            throw new IllegalStateException("Not connected");
        }
        @Override void connect(DatabaseConnectionEnum db) {
            System.out.println("Connecting...");
            db.setState(CONNECTED);
        }
        @Override void disconnect(DatabaseConnectionEnum db) {
            System.out.println("Already disconnected.");
        }
    },
    CONNECTED {
        @Override void executeQuery(DatabaseConnectionEnum db, String sql) {
            System.out.println("Executing SQL: " + sql);
        }
        @Override void connect(DatabaseConnectionEnum db) {
            System.out.println("Already connected.");
        }
        @Override void disconnect(DatabaseConnectionEnum db) {
            System.out.println("Disconnecting...");
            db.setState(DISCONNECTED);
        }
    };

    abstract void executeQuery(DatabaseConnectionEnum db, String sql);
    abstract void connect(DatabaseConnectionEnum db);
    abstract void disconnect(DatabaseConnectionEnum db);
}

class DatabaseConnectionEnum {
    private ConnectionStateEnum state = ConnectionStateEnum.DISCONNECTED;
    void setState(ConnectionStateEnum s) { this.state = s; }
    public String state() { return state.name(); }

    public void executeQuery(String sql) { state.executeQuery(this, sql); }
    public void connect() { state.connect(this); }
    public void disconnect() { state.disconnect(this); }
}
```

**Pros:** shortest code; instances are **inherent singletons**; robust to serialization and reflection.  
**Cons:** cannot subclass; not ideal if you need per‑state fields or families of states.


### 3.3 Sealed Interface + Final Classes (Java 17+)

```java
sealed interface SealedConnectionState
        permits SealedDisconnectedState, SealedConnectedState {
    void executeQuery(SealedDatabaseConnection db, String sql);
    void connect(SealedDatabaseConnection db);
    void disconnect(SealedDatabaseConnection db);
    String name();
}

final class SealedDisconnectedState implements SealedConnectionState {
    static final SealedDisconnectedState INSTANCE = new SealedDisconnectedState();
    private SealedDisconnectedState() {}

    @Override public void executeQuery(SealedDatabaseConnection db, String sql) {
        throw new IllegalStateException("Not connected");
    }
    @Override public void connect(SealedDatabaseConnection db) {
        System.out.println("Connecting...");
        db.setState(SealedConnectedState.INSTANCE);
    }
    @Override public void disconnect(SealedDatabaseConnection db) {
        System.out.println("Already disconnected.");
    }
    @Override public String name() { return "DISCONNECTED"; }
}

final class SealedConnectedState implements SealedConnectionState {
    static final SealedConnectedState INSTANCE = new SealedConnectedState();
    private SealedConnectedState() {}

    @Override public void executeQuery(SealedDatabaseConnection db, String sql) {
        System.out.println("Executing SQL: " + sql);
    }
    @Override public void connect(SealedDatabaseConnection db) {
        System.out.println("Already connected.");
    }
    @Override public void disconnect(SealedDatabaseConnection db) {
        System.out.println("Disconnecting...");
        db.setState(SealedDisconnectedState.INSTANCE);
    }
    @Override public String name() { return "CONNECTED"; }
}

class SealedDatabaseConnection {
    private SealedConnectionState state = SealedDisconnectedState.INSTANCE;
    void setState(SealedConnectionState s) { this.state = s; }
    public String state() { return state.name(); }

    public void executeQuery(String sql) { state.executeQuery(this, sql); }
    public void connect() { state.connect(this); }
    public void disconnect() { state.disconnect(this); }
}
```

**Why sealed?** You **close the set** of possible states at compile time — great for safety, reasoning, and pattern matching.


---

## 4) Real‑World Example B — File: CLOSED / OPEN / LOCKED

Try reading a closed file? That’s an error. Lock an open file? That transitions to `LOCKED`.  
Again: **no `if/else` on state** inside `FileContext` — everything is delegated.

### 4.1 Enum Variant

```java
enum FileState {
    CLOSED {
        @Override void open(FileContext f) { System.out.println("Opening..."); f.setState(OPEN); }
        @Override void close(FileContext f) { System.out.println("Already closed."); }
        @Override void read(FileContext f, String what) { throw new IllegalStateException("File closed"); }
        @Override void write(FileContext f, String data) { throw new IllegalStateException("File closed"); }
        @Override void lock(FileContext f) { System.out.println("Locking closed file..."); f.setState(LOCKED); }
        @Override void unlock(FileContext f) { System.out.println("Not locked."); }
    },
    OPEN {
        @Override void open(FileContext f) { System.out.println("Already open."); }
        @Override void close(FileContext f) { System.out.println("Closing..."); f.setState(CLOSED); }
        @Override void read(FileContext f, String what) { System.out.println("Reading: " + what); }
        @Override void write(FileContext f, String data) { System.out.println("Writing: " + data); }
        @Override void lock(FileContext f) { System.out.println("Locking..."); f.setState(LOCKED); }
        @Override void unlock(FileContext f) { System.out.println("Already unlocked."); }
    },
    LOCKED {
        @Override void open(FileContext f) { System.out.println("Locked: cannot open."); }
        @Override void close(FileContext f) { System.out.println("Locked: cannot close."); }
        @Override void read(FileContext f, String what) { throw new IllegalStateException("File locked"); }
        @Override void write(FileContext f, String data) { throw new IllegalStateException("File locked"); }
        @Override void lock(FileContext f) { System.out.println("Already locked."); }
        @Override void unlock(FileContext f) { System.out.println("Unlocking..."); f.setState(OPEN); }
    };

    abstract void open(FileContext f);
    abstract void close(FileContext f);
    abstract void read(FileContext f, String what);
    abstract void write(FileContext f, String data);
    abstract void lock(FileContext f);
    abstract void unlock(FileContext f);
}

class FileContext {
    private FileState state = FileState.CLOSED;
    void setState(FileState s) { this.state = s; }
    public String state() { return state.name(); }

    public void open() { state.open(this); }
    public void close() { state.close(this); }
    public void read(String what) { state.read(this, what); }
    public void write(String data) { state.write(this, data); }
    public void lock() { state.lock(this); }
    public void unlock() { state.unlock(this); }
}

class FileEnumDemo {
    public static void main(String[] args) {
        FileContext f = new FileContext();
        System.out.println("State: " + f.state());
        try { f.read("hdr"); } catch (Exception e) { System.out.println("ERR: " + e.getMessage()); }
        f.open(); f.write("v1"); f.lock();
        try { f.write("v2"); } catch (Exception e) { System.out.println("ERR: " + e.getMessage()); }
        f.unlock(); f.read("tail"); f.close();
        System.out.println("State: " + f.state());
    }
}
```


### 4.2 Sealed Interface + Final Classes (Java 17+)

```java
sealed interface FileStateSealed permits ClosedState, OpenState, LockedState {
    void open(FileContextSealed f);
    void close(FileContextSealed f);
    void read(FileContextSealed f, String what);
    void write(FileContextSealed f, String data);
    void lock(FileContextSealed f);
    void unlock(FileContextSealed f);
    String name();
}

final class ClosedState implements FileStateSealed {
    static final ClosedState INSTANCE = new ClosedState();
    private ClosedState() {}
    public void open(FileContextSealed f) { System.out.println("Opening..."); f.setState(OpenState.INSTANCE); }
    public void close(FileContextSealed f) { System.out.println("Already closed."); }
    public void read(FileContextSealed f, String what) { throw new IllegalStateException("File closed"); }
    public void write(FileContextSealed f, String data) { throw new IllegalStateException("File closed"); }
    public void lock(FileContextSealed f) { System.out.println("Locking closed file..."); f.setState(LockedState.INSTANCE); }
    public void unlock(FileContextSealed f) { System.out.println("Not locked."); }
    public String name() { return "CLOSED"; }
}

final class OpenState implements FileStateSealed {
    static final OpenState INSTANCE = new OpenState();
    private OpenState() {}
    public void open(FileContextSealed f) { System.out.println("Already open."); }
    public void close(FileContextSealed f) { System.out.println("Closing..."); f.setState(ClosedState.INSTANCE); }
    public void read(FileContextSealed f, String what) { System.out.println("Reading: " + what); }
    public void write(FileContextSealed f, String data) { System.out.println("Writing: " + data); }
    public void lock(FileContextSealed f) { System.out.println("Locking..."); f.setState(LockedState.INSTANCE); }
    public void unlock(FileContextSealed f) { System.out.println("Already unlocked."); }
    public String name() { return "OPEN"; }
}

final class LockedState implements FileStateSealed {
    static final LockedState INSTANCE = new LockedState();
    private LockedState() {}
    public void open(FileContextSealed f) { System.out.println("Locked: cannot open."); }
    public void close(FileContextSealed f) { System.out.println("Locked: cannot close."); }
    public void read(FileContextSealed f, String what) { throw new IllegalStateException("File locked"); }
    public void write(FileContextSealed f, String data) { throw new IllegalStateException("File locked"); }
    public void lock(FileContextSealed f) { System.out.println("Already locked."); }
    public void unlock(FileContextSealed f) { System.out.println("Unlocking..."); f.setState(OpenState.INSTANCE); }
    public String name() { return "LOCKED"; }
}

class FileContextSealed {
    private FileStateSealed state = ClosedState.INSTANCE;
    void setState(FileStateSealed s) { this.state = s; }
    public String state() { return state.name(); }

    public void open() { state.open(this); }
    public void close() { state.close(this); }
    public void read(String what) { state.read(this, what); }
    public void write(String data) { state.write(this, data); }
    public void lock() { state.lock(this); }
    public void unlock() { state.unlock(this); }
}
```

*Demo is identical to `FileEnumDemo`, only using `FileContextSealed` instead of `FileContext`.*



---

## 5) Do I Need `new` for States? (No.)

The pattern **does not require** allocating a new object per transition. Choose based on needs:

- **Singleton/Flyweight per state** (used above): best when states are **stateless**; saves allocations and simplifies identity checks.
- **`enum`**: even more compact; each constant is a natural singleton.
- **Fresh objects**: when a state needs **per‑instance data** (e.g., retry counters, timers). In that case either store that data inside the **context** or create a new state instance on entry.


---

## 6) State vs Strategy — Quick Contrast

- **State**: the **context** drives transitions between concrete implementations (often the state objects themselves decide the next state).
- **Strategy**: the **client** chooses the algorithm/implementation; no autonomous transitions.  
If your object “**changes behavior over time** as it runs,” it’s probably **State**.


---

## 7) Testing & Maintainability

- **Unit tests:** exercise transitions and behavior per state. Because states are singletons/enums, tests are fast and allocation‑free.
- **Adding a state:** create a new type and wire transitions explicitly. No “touch every if/else” problem.
- **Sealed interfaces:** give compile‑time certainty about the set of states (and great switch‑matching support).


---

## 8) Choosing a Variant

- Want the **shortest**, robust code → **`enum`**.
- Need **closed set** + classes (DI, separate files) → **sealed interface + final** (Java 17+).
- Need **flexibility** without sealing → **interface + final singletons**.
- Need **per‑state data** → create **new instances** on entry (or store data in the context).


---

## 9) Anti‑Patterns & Smells

- **Giant if/switch** spread across methods — State exists to delete those.
- **Mutable state objects** shared globally — keep state objects **stateless** unless you’re creating new instances per context.
- **Hidden transitions** via external flags — transitions should be **explicit** inside states or the context.


---

## 10) Summary

The **State Pattern** gives you:
- **Locality** of behavior per state (readable, testable),
- **Explicit transitions** (clear flow),
- **No big conditionals** (polymorphism instead),
- Easy extension by **adding types**, not conditionals.

Pick **enum** for compactness, **sealed** for safety, **singleton states** for performance, and **fresh instances** only when you truly need per‑state data.
