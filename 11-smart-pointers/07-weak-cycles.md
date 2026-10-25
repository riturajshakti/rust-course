[← 11.6 Cow: Clone on Write](06-cow.md) · [Course Index](../README.md) · [12.1 Modules and Visibility →](../12-modules-crates/01-modules.md)

# 11.7 — Weak References and Cycles

## What You'll Learn

- How two `Rc<RefCell<T>>` pointing at each other leak, with `Drop` never firing
- `Weak<T>`, `Rc::downgrade`, and `upgrade()` returning `Option`
- The parent-child tree pattern: strong down, weak up
- Why an index-based arena is usually the better answer

## Coming From Another Language

| Other | Rust |
|---|---|
| JS/Python/Go: tracing GC collects cycles | refcounting does not — a cycle leaks |
| Python `weakref.ref(x)` | `Rc::downgrade(&x)` -> `Weak<T>` |
| Java `WeakReference` | `Weak<T>`, `upgrade()` instead of `get()` |

Reference counting has exactly one failure mode: a cycle. If `a` holds an `Rc`
to `b` and `b` holds one back, neither count can reach zero, so neither value is
ever dropped. There is no cycle collector in Rust — you break the cycle yourself
with `Weak`.

## Building the leak

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Node {
    name: &'static str,
    link: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("dropping {}", self.name);
    }
}

fn main() {
    {
        let a = Rc::new(Node { name: "a", link: RefCell::new(None) });
        let b = Rc::new(Node { name: "b", link: RefCell::new(None) });
        *a.link.borrow_mut() = Some(Rc::clone(&b));
        *b.link.borrow_mut() = Some(Rc::clone(&a));
        println!("a strong = {}", Rc::strong_count(&a));
        println!("b strong = {}", Rc::strong_count(&b));
        println!("leaving scope");
    }
    println!("scope ended -- no drops printed above this line");

    {
        let c = Rc::new(Node { name: "c", link: RefCell::new(None) });
        println!("c strong = {}", Rc::strong_count(&c));
    }
    println!("done");
}
```

Output:

```
a strong = 2
b strong = 2
leaving scope
scope ended -- no drops printed above this line
c strong = 1
dropping c
done
```

`dropping a` and `dropping b` never print. The bindings went away, taking each
count from 2 to 1 — and each remaining handle is held by the *other* node, which
is also still alive. The memory is lost for the rest of the process.

The uncycled `c` drops immediately, which is the control case proving `Drop` is
wired up correctly.

This is a leak, not unsafety: no use-after-free, no UB. Rust's guarantee is
memory *safety*, not the absence of leaks.

## Breaking it by hand

If the cycle is unavoidable, you can cut an edge before the values go out of
scope. Setting one `link` to `None` drops that `Rc` and lets the chain unwind.

```rust
fn main() {
    let a = Rc::new(Node { name: "a", link: RefCell::new(None) });
    let b = Rc::new(Node { name: "b", link: RefCell::new(None) });
    *a.link.borrow_mut() = Some(Rc::clone(&b));
    *b.link.borrow_mut() = Some(Rc::clone(&a));
    println!("a = {}, b = {}", Rc::strong_count(&a), Rc::strong_count(&b));

    *b.link.borrow_mut() = None;
    println!("after cutting b->a: a = {}, b = {}", Rc::strong_count(&a), Rc::strong_count(&b));
    println!("-- end of main --");
}
```

Output:

```
a = 2, b = 2
after cutting b->a: a = 1, b = 2
-- end of main --
dropping a
dropping b
```

Workable, but it is manual memory management with extra steps — you have to
remember to do it on every path, including early returns and panics. `Weak`
makes it structural instead.

## Weak\<T\>

`Rc::downgrade(&rc)` gives a `Weak<T>` — a handle that does **not** keep the
value alive. To use it you must `upgrade()`, which returns `Option<Rc<T>>`:
`Some` if the value is still there, `None` if it's gone.

```rust
use std::rc::{Rc, Weak};

fn main() {
    let strong = Rc::new(String::from("value"));
    let weak: Weak<String> = Rc::downgrade(&strong);
    println!("strong = {}, weak = {}", Rc::strong_count(&strong), Rc::weak_count(&strong));
    println!("{:?}", weak.upgrade());
    if let Some(v) = weak.upgrade() {
        println!("upgraded: {} (strong now {})", v, Rc::strong_count(&strong));
    }
    println!("strong = {}", Rc::strong_count(&strong));
    drop(strong);
    println!("{:?}", weak.upgrade());
    println!("{}", weak.upgrade().is_none());

    let empty: Weak<i32> = Weak::new();
    println!("{:?}", empty.upgrade());
    println!("{} {}", size_of::<Weak<i32>>(), size_of::<Rc<i32>>());
}
```

Output:

```
strong = 1, weak = 1
Some("value")
upgraded: value (strong now 2)
strong = 1
None
true
None
8 8
```

Three things to notice. `upgrade()` temporarily bumps the strong count — the
`Rc` it hands you is a real owner for as long as you hold it. `Weak::new()`
gives a dangling weak with nothing behind it, which is how you initialize a
parent field before the parent exists. And `Weak<T>` is the same 8 bytes as
`Rc<T>`.

You cannot read through a `Weak` directly; there is no `Deref`. `upgrade()` is
mandatory, and that is precisely what makes it safe.

## A Weak keeps the allocation header alive

The *value* drops at strong count zero, but the small refcount block survives
until the last weak handle goes too.

```rust
use std::rc::Rc;

struct Big([u8; 64]);
impl Drop for Big {
    fn drop(&mut self) { println!("Big dropped"); }
}

fn main() {
    let strong = Rc::new(Big([0; 64]));
    let w = Rc::downgrade(&strong);
    drop(strong);
    println!("value gone: {}", w.upgrade().is_none());
    println!("weak still held, allocation header alive");
    drop(w);
    println!("now fully freed");
}
```

Output:

```
Big dropped
value gone: true
weak still held, allocation header alive
now fully freed
```

So a long-lived `Weak` to a huge value is cheap, but a million stale `Weak`s
still cost a million headers. Prune them if you keep a registry.

## The parent-child tree

The canonical fix: children held strongly by the parent, parent held weakly by
each child. Strong edges point *down* the tree, weak edges *up*.

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("dropping node {}", self.value);
    }
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade().map(|p| p.value));
    println!("leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![Rc::clone(&leaf)]),
        });
        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!("leaf parent = {:?}", leaf.parent.borrow().upgrade().map(|p| p.value));
        println!("branch strong = {}, weak = {}", Rc::strong_count(&branch), Rc::weak_count(&branch));
        println!("leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));
        println!("branch child = {}", branch.children.borrow()[0].value);
        println!("-- leaving branch scope --");
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade().map(|p| p.value));
    println!("leaf strong = {}, weak = {}", Rc::strong_count(&leaf), Rc::weak_count(&leaf));
    println!("-- end of main --");
}
```

Output:

```
leaf parent = None
leaf strong = 1, weak = 0
leaf parent = Some(5)
branch strong = 1, weak = 1
leaf strong = 2, weak = 0
branch child = 3
-- leaving branch scope --
dropping node 5
leaf parent = None
leaf strong = 1, weak = 0
-- end of main --
dropping node 3
```

Read the counts: `branch` is strong 1 / weak 1 — only the binding owns it, the
child's back-pointer doesn't. So when the scope ends, `branch` drops *right
there*. Its `Vec<Rc<Node>>` drops with it, taking `leaf` back to strong 1, and
`leaf.parent.upgrade()` now returns `None` automatically. No dangling pointer,
no manual cleanup, no leak.

`RefCell::new(Weak::new())` is the initialization order trick: build the child
first with no parent, then patch the back-pointer once the parent exists.

## A registry of Weak handles

The other common use: observe things you don't own. `Weak::strong_count()` lets
you drop the dead entries without upgrading each one.

```rust
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Resource(u32);

struct Registry {
    entries: Vec<Weak<Resource>>,
}

impl Registry {
    fn live(&self) -> Vec<u32> {
        self.entries.iter().filter_map(|w| w.upgrade()).map(|r| r.0).collect()
    }
    fn prune(&mut self) {
        self.entries.retain(|w| w.strong_count() > 0);
    }
}

fn main() {
    let a = Rc::new(Resource(1));
    let b = Rc::new(Resource(2));
    let c = Rc::new(Resource(3));
    let mut reg = Registry {
        entries: vec![Rc::downgrade(&a), Rc::downgrade(&b), Rc::downgrade(&c)],
    };
    println!("{:?} len={}", reg.live(), reg.entries.len());
    drop(b);
    println!("{:?} len={}", reg.live(), reg.entries.len());
    reg.prune();
    println!("{:?} len={}", reg.live(), reg.entries.len());
    drop(a);
    drop(c);
    reg.prune();
    println!("{:?} len={}", reg.live(), reg.entries.len());
}
```

Output:

```
[1, 2, 3] len=3
[1, 3] len=3
[1, 3] len=2
[] len=0
```

`live()` skips the dead entry automatically via `filter_map`, but the slot stays
until `prune` runs — that is the stale-header cost from above. Exactly how an
event bus or observer list avoids keeping its subscribers alive.

## Drop order with cycles

Dropping a chain of `Rc`s is recursive and runs from the handle you drop
outward. Two consequences worth knowing:

- A cycle means `Drop` never runs for any node in it, so any cleanup you put in
  `Drop` (closing a file, flushing a buffer, decrementing a gauge) is silently
  skipped. The leak is the visible symptom; the missed cleanup is the subtle one.
- A very long `Rc` chain drops recursively and can overflow the stack. A
  100,000-node linked list built from `Rc` or `Box` can abort on drop. Fix it
  with an iterative `Drop` impl, or by not building a long pointer chain.

To find a leak: implement `Drop` with a `println!`, run, and look for the node
that never prints. `Rc::strong_count` stuck above the number of bindings you can
account for is the other tell.

## The better answer: an arena

`Rc<RefCell<Node>>` + `Weak` works, but every traversal is a `borrow()` plus an
`upgrade()` plus an `Option` match. For most graphs and trees, store the nodes in
a `Vec` and use `usize` indices as your edges.

```rust
#[derive(Debug)]
struct Node {
    value: i32,
    parent: Option<usize>,
    children: Vec<usize>,
}

struct Arena {
    nodes: Vec<Node>,
}

impl Arena {
    fn add(&mut self, value: i32, parent: Option<usize>) -> usize {
        let id = self.nodes.len();
        self.nodes.push(Node { value, parent, children: vec![] });
        if let Some(p) = parent {
            self.nodes[p].children.push(id);
        }
        id
    }
    fn path(&self, mut id: usize) -> Vec<i32> {
        let mut out = vec![];
        loop {
            out.push(self.nodes[id].value);
            match self.nodes[id].parent {
                Some(p) => id = p,
                None => break,
            }
        }
        out.reverse();
        out
    }
}

fn main() {
    let mut a = Arena { nodes: vec![] };
    let root = a.add(1, None);
    let mid = a.add(5, Some(root));
    let leaf = a.add(3, Some(mid));
    println!("{:?}", a.path(leaf));
    println!("{:?}", a.nodes[mid].children);
    println!("{} nodes, no Rc, no RefCell, no cycle risk", a.nodes.len());
}
```

Output:

```
[1, 5, 3]
[2]
3 nodes, no Rc, no RefCell, no cycle risk
```

Cycles are now just indices pointing at each other — harmless, because freeing
the `Vec` frees everything at once. Nodes are contiguous so traversal is
cache-friendly, and `Clone`/`Debug`/serialization all work normally.

The trade-off: indices can go stale if you remove nodes, and nothing stops you
indexing the wrong arena. Crates like `slotmap` and `generational-arena` fix that
with generation-tagged keys. Reach for `Rc`/`Weak` when nodes genuinely have
independent lifetimes; reach for an arena when they live and die together, which
is most of the time.

## Quick Reference

| Call | Returns | Notes |
|---|---|---|
| `Rc::downgrade(&rc)` | `Weak<T>` | does not own the value |
| `weak.upgrade()` | `Option<Rc<T>>` | `None` once the value is dropped |
| `Weak::new()` | `Weak<T>` | dangling; `upgrade()` is always `None` |
| `Rc::weak_count(&rc)` | `usize` | live `Weak` handles |
| `weak.strong_count()` | `usize` | 0 means dead; no upgrade needed |
| `Rc::strong_count(&rc)` | `usize` | live `Rc` handles |
| `Arc::downgrade` / `sync::Weak` | same API | thread-safe version |
| drop at strong 0 | value dropped | header lives until weak also 0 |
| `Weak<T>` size | 8 bytes | same as `Rc<T>` |
| no `Deref` on `Weak` | — | must `upgrade()` first |

## Common Mistakes

**Both directions strong.** The classic leak. One direction must be `Weak` —
conventionally the one pointing "up" or "back."

**`unwrap()` on `upgrade()`.** The whole point is that it can be `None`. Match
or `if let`.

**Holding the upgraded `Rc` longer than needed.** It counts as a strong
reference while alive, so keeping it in a field recreates the cycle.

**Assuming a leak is unsafe.** It isn't — no UB, just memory you never get
back, and `Drop` side effects that never happen.

**Modelling a tree with `Rc<RefCell<Node>>` by default.** Try indices into a
`Vec` first; it is simpler, faster, and cannot cycle.

## Exercises

1. Build the two-node cycle from this lesson, confirm neither `Drop` prints,
   then break it by setting one `link` to `None` before the scope ends and watch
   both drop.
2. Make a doubly linked list of three nodes with `Rc` forward and `Weak`
   backward. Walk it in both directions and print the values.
3. Write a registry `Vec<Weak<Resource>>`, drop two of five resources, then
   write a `prune` method that removes the dead entries. Print the length before
   and after.
4. Convert the parent-child tree in this lesson to an arena with `usize`
   indices. Implement `depth(id)` and `descendants(id)`.
5. Build a 100,000-node `Rc` chain and drop it. If it aborts, write an iterative
   `Drop` impl that unlinks nodes one at a time and confirm it no longer does.

---

[← 11.6 Cow: Clone on Write](06-cow.md) · [Course Index](../README.md) · [12.1 Modules and Visibility →](../12-modules-crates/01-modules.md)
