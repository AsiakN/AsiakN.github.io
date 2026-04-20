---
title: Why Zanzibar matters
date: 2026-03-10
math: false
---

Any authorization system worth considering needs to deliver on three fronts:

1. **Flexibility.** Different users need to take different actions without being shoehorned into rigid roles.
2. **Correctness.** Tight control that ensures the right users take the right actions on the right resources.
3. **Scale.** It should protect thousands (or billions) of objects shared across large sets of actors.

RBAC scores reasonably well on correctness and scale but struggles with flexibility. The moment you need genuine per-resource permissions (think "Alice can edit *this* document but only view *that* one"), you start drowning in role explosion.

**Zanzibar's** relationship-based model addresses all three fronts, and the way it handles each is worth unpacking.

## The Zanzibar Model

Zanzibar, Google's authorization system, represents authorization as a graph of relationship tuples. A tuple looks like this:

```
(document:readme, editor, user:anne)
```

Three parts: an object, a relation, and a subject. *"Anne is an editor of the readme document."* Permissions aren't derived from roles attached to users. They're derived from relationships attached to resources. That small reframing is what makes everything else possible.

**Flexibility** comes from the fact that relationships compose. You can say "Anne is an editor of the engineering folder" and "the readme lives in that folder," then define in your model that editors of a folder inherit editor rights on the documents inside it. Suddenly you get inherited permissions across a whole resource hierarchy without writing a single line of application logic. Groups nest into groups, teams inherit from organizations, and sharing patterns that would require dozens of RBAC roles collapse into a handful of relationship definitions.

**Correctness** is about more than "explicit, auditable relationships." The harder problem is consistency. Authorization checks happen constantly, on every request, often many times per request. In a distributed system with replicated caches, you will eventually hit this bug: a user's permission gets revoked, but a stale cache says otherwise, and they perform an action they shouldn't. Zanzibar solves this with *zookies* (OpenFGA calls them zedtokens). They're opaque consistency tokens that pin a permission check to a specific version of the authorization graph. When a user loads a document, you record a zookie. 

When they later try to modify it, you check with that zookie. The system guarantees the check reflects a state at least as fresh as the one the user saw. Race conditions between permission changes and actions disappear. This is the part that's genuinely hard to get right, and it's the part that roll-your-own authorization systems almost universally get wrong.

**Scale** is proven. Google has been running Zanzibar for years across YouTube, Drive, Cloud, Calendar, and more. Billions of objects, millions of checks per second. The architecture, distributed tuple storage with aggressive caching keyed on consistency tokens, is designed for this from the ground up.

## OpenFGA: Zanzibar for the Rest of Us

OpenFGA is an open-source implementation of Zanzibar's concepts. Same relationship-based model, same tuple storage, same check semantics, all without building it from scratch.

Its DSL is one of the strongest features. An authorization model reads almost like English:

```
model
  schema 1.1

type user

type document
  relations
    define viewer: [user]
    define editor: [user]
    define owner: [user]
```

The people who understand the business rules can participate in defining access policies. That's a meaningful shift from authorization logic buried in application code that only engineers can modify.

## Final Thought

Zanzibar is a great example of how research and knowledge sharing from large organizations push the boundaries of system design for everyone. The problems Google solved at their scale (flexible, correct, relationship-based authorization) are the same problems the rest of us face, just at different magnitudes.

And I like the name Zanzibar. It's giving calm vibes.
