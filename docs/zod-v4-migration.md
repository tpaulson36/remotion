# Zod v3 → v4 Migration Plan

> Catalog currently: `zod: 3.22.3` → target: `4.4.3`
> Identified by nightly-code-health report 2026-05-23.
> **Do not bump the catalog entry until all steps below are complete.**

---

## Why this is not a simple version bump

Zod v4 has three breaking changes that directly affect remotion's schema editor
architecture. All 71 import sites were audited; the risk is concentrated in
~20 SchemaEditor components and the `@remotion/zod-types` package.

---

## Breaking Change 1 — `._def` → `._zod.def`

Every internal property access on a Zod schema must change:

| v3 | v4 |
|----|-----|
| `schema._def` | `schema._zod.def` |
| `schema._def.typeName` | `schema._zod.def.typeName` |
| `schema._def.description` | `schema._zod.def.description` (verify) |
| `schema._def.schema` | `schema._zod.def.schema` (verify — may not exist after BC2) |

**Primary file:** `packages/studio/src/components/RenderModal/SchemaEditor/ZodSwitch.tsx`

```ts
// v3 — remove this
const def: z.ZodTypeDef = schema._def;
const typeName = (def as any).typeName as z.ZodFirstPartyTypeKind;

// v4 — replace with
const def = schema._zod.def;
const typeName = (def as any).typeName; // see BC3 for enum situation
```

**All files requiring `_def` → `_zod.def` updates:**
- `SchemaEditor/ZodSwitch.tsx`
- `SchemaEditor/ZodEffectEditor.tsx`
- `SchemaEditor/ZodDefaultEditor.tsx`
- `SchemaEditor/ZodOptionalEditor.tsx`
- `SchemaEditor/ZodNullableEditor.tsx`
- `SchemaEditor/ZodObjectEditor.tsx`
- `SchemaEditor/ZodArrayEditor.tsx`
- `SchemaEditor/ZodTupleEditor.tsx`
- `SchemaEditor/ZodDiscriminatedUnionEditor.tsx`
- `SchemaEditor/ZodUnionEditor.tsx`
- Any other component that reads `schema._def` directly

---

## Breaking Change 2 — `ZodEffects` removed

In v3, `.refine()` and `.transform()` wrapped the schema in a `ZodEffects`
class. `ZodSwitch` dispatches on `ZodFirstPartyTypeKind.ZodEffects` to
detect `zColor`, `zMatrix`, and generic effects.

In v4, **refinements live inside the schema's `checks[]` array** — there is
no `ZodEffects` wrapper. The `ZodEffects` branch in `ZodSwitch` will never
match.

### Impact on `@remotion/zod-types`

`zColor` and `zMatrix` use `.refine().describe(BRAND)` to attach a brand
string. The detection reads `schema._def.description === REMOTION_COLOR_BRAND`.

**v3 pattern (must change):**
```ts
// zod-types/z-color.ts
export const zColor = () =>
  z.string()
    .refine(...)
    .describe(REMOTION_COLOR_BRAND); // stored in ZodEffects._def.description

// ZodSwitch.tsx detection
if (typeName === z.ZodFirstPartyTypeKind.ZodEffects) {
  if (schema._def.description === REMOTION_COLOR_BRAND) { ... }
}
```

**v4 options (pick one):**

**Option A — `z.registry()` (recommended by Zod team):**
```ts
const remotionRegistry = z.registry<{ brand: string }>();

export const zColor = () => {
  const schema = z.string().refine(...);
  remotionRegistry.add(schema, { brand: REMOTION_COLOR_BRAND });
  return schema;
};

// Detection in ZodSwitch:
const meta = remotionRegistry.get(schema);
if (meta?.brand === REMOTION_COLOR_BRAND) { ... }
```

**Option B — `.meta()` (v4 preferred over `.describe()`):**
```ts
export const zColor = () =>
  z.string().refine(...).meta({ brand: REMOTION_COLOR_BRAND });

// Detection:
if ((schema as any)._zod.def.metadata?.brand === REMOTION_COLOR_BRAND) { ... }
```

**Option C — Symbol on the schema object (zero Zod API surface):**
```ts
const COLOR_BRAND = Symbol('remotion-color');
export const zColor = () => {
  const schema = z.string().refine(...);
  (schema as any)[COLOR_BRAND] = true;
  return schema;
};
// Detection: if ((schema as any)[COLOR_BRAND]) { ... }
```

Option A is safest long-term. Option C requires no Zod internal knowledge.

---

## Breaking Change 3 — `ZodFirstPartyTypeKind` status

This enum is quasi-internal and not covered by the official v4 migration guide.
The entire `ZodSwitch.tsx` type-dispatch (15+ branches) uses it:

```ts
z.ZodFirstPartyTypeKind.ZodObject
z.ZodFirstPartyTypeKind.ZodString
z.ZodFirstPartyTypeKind.ZodArray
// ... 12 more
```

**First step:** Check if `ZodFirstPartyTypeKind` still exists after installing v4:
```bash
cd packages/studio && bun add zod@4 --dry-run
node -e "const z = require('zod'); console.log(z.ZodFirstPartyTypeKind)"
```

**If still exported:** Only the `_def` → `_zod.def` and `ZodEffects` changes apply.

**If removed:** Replace each branch with `instanceof` or string literals:
```ts
// Option: instanceof
if (schema instanceof z.ZodObject) { ... }

// Option: string literal against _zod.def.typeName
if (def.typeName === 'ZodObject') { ... }
```

The `instanceof` approach is safest and most readable.

---

## Scope by package

| Package | Files | Risk | Notes |
|---------|-------|------|-------|
| `packages/studio` SchemaEditor | ~20 | 🔴 High | BC1 + BC2 + BC3 |
| `packages/zod-types` | 3 | 🟠 Medium | BC2 brand mechanism |
| `packages/core` | ~8 | 🟢 Low | `z.infer<>`, `.parse()` — stable |
| `packages/cloudrun` | 3 | 🟢 Low | Schema validation only |
| `packages/player` | 3 | 🟢 Low | Type usage only |
| `packages/web-renderer` | 3 | 🟢 Low | Type usage only |

---

## Recommended execution order

```
Step 1 — Verify ZodFirstPartyTypeKind
  Install zod@4 in an isolated check; confirm enum presence.
  Decision gates all dispatch strategy choices.

Step 2 — Migrate @remotion/zod-types
  Pick brand detection option (A, B, or C above).
  Update z-color.ts, z-matrix.ts, z-textarea.ts.
  Add/update tests for brand round-trip.

Step 3 — Update ZodSwitch.tsx
  _def → _zod.def
  Remove ZodEffects branch; replace with new brand detection.
  Update ZodFirstPartyTypeKind references per Step 1 outcome.

Step 4 — Update each SchemaEditor component
  ~20 files; mechanical _def → _zod.def substitution + any
  ZodEffects-specific inner schema accesses.

Step 5 — Update low-risk packages
  core, cloudrun, player, web-renderer.
  Likely type-only changes; verify .safeParse() error shape if used.

Step 6 — Bump catalog
  "zod": "3.22.3" → "4.4.3" in root package.json

Step 7 — Full test run
  turbo run lint test
  Manual smoke test of Studio SchemaEditor with a complex schema
  (object with color, matrix, enum, optional, discriminated union).
```

---

## Key references

- Zod v4 changelog: <https://zod.dev/v4/changelog>
- Zod v4 docs: <https://zod.dev/v4>
