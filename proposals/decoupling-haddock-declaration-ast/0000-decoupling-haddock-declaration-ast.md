# RFC - Decoupling the Haddock Declaration AST from GHC

## Abstract

Haddock reuses GHC's full `HsDecl`/`HsType` AST to represent and render
documentation declarations. An audit of Haddock's three rendering backends
(HTML, LaTeX, Hoogle) shows that only a small, well-defined subset is ever
reached: 3 HsDecl constructors (out of ~11), 4 `TyClDecl` shapes, ~20 `HsType`
constructors, and a handful of supporting types. The rest is never rendered.

This proposal defines a standalone, serializable "Haddock Declaration AST"
that captures exactly this subset, in a new `haddock-ast` package with no GHC
dependency. Combined with the already GHC-free `DocH` documentation AST from
`haddock-library`, the two form a complete intermediate representation (IR)
for Haskell documentation, backed by JSON and described by a JSON Schema.

The implementation covers: (1) auditing and documenting the exact GHC AST
subset Haddock consumes, (2) defining the Haddock Declaration AST as new
Haskell types, (3) implementing JSON serialization and deserialization,
(4) integrating the IR as a replacement for the existing `--json` flag and as
a new `--read-ir` / `--emit-ir` pipeline so that rendering can proceed without
GHC.

## Background

### How Haddock works today

Haddock's pipeline has five stages:

1. **Parse**: GHC's parser reads Haskell source and produces a `RenamedSource`
   AST indexed by `GhcRn`.
2. **Extract**: `haddock-api` extracts documentation comments, export items,
   instances, and fixities from the typechecked module, building an
   `Interface` value.
3. **Synify**: For exported entities, GHC's internal `TyThing` values are
   converted back to `HsDecl GhcRn` via `Haddock.Convert` (the "synify"
   process). This step is necessary because GHC's Core types are not
   source-level syntax. The `synifyType` function pattern-matches on every
   constructor of `Type` from `GHC.Core.TyCo.Rep` and reconstructs the
   corresponding `HsType GhcRn`.
4. **Rename**: All GHC `Name`s are replaced with `DocName` (a `Name` plus
   cross-reference info) using the `DocNameI` pass. `DocNameI` is a fake GHC
   pass (analogous to `GhcPs` or `GhcRn`) that maps the `IdP` type family to
   `DocName` instead of `Name`. This requires approximately 113 type family
   instances: 28 mapped to `DataConCantHappen`, 67 to `NoExtField`, 15 to
   `EpAnn NoEpAnns`, plus custom mappings for `XXType`, `XExportDecl`, and
   others.
5. **Render**: The backends (HTML, LaTeX, Hoogle) pattern-match on the
   resulting `HsDecl DocNameI` and `HsType DocNameI` trees to produce output.

Steps 1-4 require GHC. Step 5 (rendering) does not inherently require GHC,
but because it operates on GHC AST types, it currently depends on the `ghc`
package.

### haddock-library vs haddock-api

`haddock-library` is GHC-free. It defines:

- `DocH mod id`: the documentation markup AST (28 constructors), parametrized
  by two type variables so it is independent of any name representation.
- Parser: converts Haddock comments to `DocH`.
- Markup: a catamorphism (`DocMarkupH`) for rendering `DocH`.

`haddock-api` is where the coupling lives. It defines:

- `DocName` and `DocNameI` (bridging GHC `Name`s to cross-referenced names).
- `Interface` and `InstalledInterface` (the module-level documentation
  structure, containing `Module`, `DynFlags`, `ClsInst`, and
  `[ExportItem GhcRn]`).
- `ExportItem`, `ExportD`, `InstHead` (combining declarations with docs).
- `Convert.hs`: `TyThing` to `HsDecl GhcRn` synification (1172 lines).
- All rendering backends.

### The DocNameI pass

To reuse GHC's AST types with Haddock's own name resolution, Haddock defines
`DocNameI` as a fake GHC "pass". It maps `IdP` to `DocName` and requires type
family instances for every extension point in the GHC AST. Any change to GHC's
AST extension points requires updating these instances. This is the main
maintenance burden of the GHC-Haddock coupling.

### Current serialization

Haddock has two existing serialization paths:

1. **Binary** (`.haddock` files): Uses `GHC.Utils.Binary`. Serializes
   `InstalledInterface` (which includes `DocMap`, `ArgMap`, fixities,
   warnings, `DocOption`, but not the declaration AST). Requires `NameCache`
   from GHC to deserialize. Version-controlled with a magic number
   (`binaryInterfaceVersion = 46` for GHC 9.11-9.13).

2. **JSON** (`--json` flag): Serializes `InstalledInterface` to GHC's internal
   `JsonDoc` format. Names become strings via `nameStableString`. The output
   is lossy: no declaration AST, no instances with types, module link labels
   discarded.

Neither format supports rendering without GHC. The binary format needs
`NameCache`, and the JSON format drops declarations entirely.

## Problem Statement

Haddock's reuse of GHC's AST creates three problems.

1. **Version lock-in.** Haddock must track GHC's AST changes. The `DocNameI`
   pass (~113 type family instances) must be updated whenever GHC's extension
   points change. The binary serialization format is tied to GHC's `Binary`
   instances and `NameCache`.

2. **GHC is required to render.** Even though rendering is a presentation
   concern, all three backends import from the `ghc` package because they
   pattern-match on `HsDecl`/`HsType` constructors.

3. **No external tooling can consume documentation.** Hackage, Hoogle,
   haskell-language-server, and other tools must either use `haddock-api`
   (pulling in GHC) or parse rendered HTML.

The requirements for a solution:

- The IR must capture everything Haddock's backends need to render
  documentation for a module.
- The IR must be serializable to JSON, described by a JSON Schema, and
  versioned.
- The IR must not require any GHC type or GHC package to deserialize.
- Backward compatibility: newer Haddock versions should read IR from older
  versions.

## Prior Art and Related Efforts

### Original Haddock IR proposal (HF tech-proposals PR #44)

"Maximally decoupling Haddock and GHC" proposed a serializable IR plus
refactoring Haddock into GHC-specific and GHC-agnostic packages. It was
discussed in TWG meetings from November 2022 through October 2023. A HSoC
student worked on it under Laurent R. de Cotret's mentorship. The scope
proved too large: it included creating `haddock-backends`, moving
GHC-agnostic code to `haddock-library`, and having GHC emit the IR. The
proposal was never merged. In October 2023, the Haddock-specific work was
explicitly moved to "Future Work."

This proposal reduces scope to the core deliverable: defining and serializing
the AST subset.

### proto-docser-hs

A prototype that serializes the entire GHC `RenamedSource` AST to JSON using
`aeson` `Generic`-derived `ToJSON` instances (~150 GHC AST types, 994 lines).
It demonstrated that brute-force serialization is possible but has critical
limitations:

- `Type` (GHC Core), `Var`, and `ThModFinalizers` cannot be meaningfully
  serialized. They are serialized as placeholder strings, losing all
  information.
- `Name` is serialized lossily (occurrence name + unique + source span),
  losing `NameSort` and other internals.
- The output is tightly coupled to GHC's internal type structure and would
  break across GHC versions.
- The output is large: the entire renamed AST is serialized, including parts
  that Haddock never uses (expressions, patterns, bindings, splice
  declarations, etc.).

Our approach differs: we define a minimal AST containing only what Haddock
needs. This is smaller, stable across GHC versions, and includes only
serializable types.

### rustdoc JSON (Rust RFC 2963)

The Rust community added JSON output to `rustdoc`. Before 2020, `rustdoc`
parsed and rendered HTML in one step. The JSON output decoupled these,
enabling `rust-analyzer` and other tools to consume Rust documentation
without the compiler. Our approach follows the same pattern.

### Pandoc types

Pandoc defines its document AST in a separate package (`pandoc-types`),
backed by JSON serialization. Third-party tools can read, transform, and
write Pandoc documents without depending on Pandoc itself. The
`haddock-library` `DocH` type already follows this pattern for documentation
markup. We extend it to declaration types.

### GHC JSON diagnostic dump (HF proposal #050)

Gained acceptance for a versioned JSON output of GHC diagnostics, described
by a JSON Schema, with top-level `version` and `ghcVersion` fields. We follow
the same conventions.

## Technical Content

### Step 1: Audit of the GHC AST subset

An audit of Haddock's three backends reveals exactly which GHC AST
constructors are used.

#### `HsDecl` constructors used by backends

| Constructor | HTML | LaTeX | Hoogle | Notes |
|---|---|---|---|---|
| `TyClD` (`FamDecl`) | yes | yes | yes | Type/data families |
| `TyClD` (`DataDecl`) | yes | yes | yes | Data/newtype |
| `TyClD` (`SynDecl`) | yes | yes | yes | Type synonyms |
| `TyClD` (`ClassDecl`) | yes | yes | yes | Type classes |
| `SigD` (`TypeSig`) | yes | yes | yes | Function signatures |
| `SigD` (`PatSynSig`) | yes | yes | yes | Pattern synonyms |
| `SigD` (`ClassOpSig`) | yes | yes | -- | Class methods |
| `SigD` (`MinimalSig`) | yes | -- | -- | Minimal complete def |
| `ForD` (`ForeignImport`) | yes | yes | yes | FFI imports |
| `ForD` (`ForeignExport`) | -- | -- | yes | FFI exports |
| `InstD` | consumed, no output | same | -- | Filtered out |
| `DerivD` | consumed, no output | same | -- | Filtered out |
| `ValD`, `SpliceD`, `RuleD`, `AnnD`, `WarningD`, `DefD`, `DocD` | never | never | never | Not used |

This gives 3 productive `HsDecl` constructors: `TyClD`, `SigD`, `ForD`.

#### `HsType` constructors used by all three backends

`HsForAllTy`, `HsQualTy`, `HsFunTy`, `HsTyVar`, `HsStarTy`, `HsAppTy`,
`HsAppKindTy`, `HsListTy`, `HsIParamTy`, `HsTupleTy`, `HsSumTy`, `HsOpTy`,
`HsParTy`, `HsKindSig`, `HsExplicitListTy`, `HsExplicitTupleTy`,
`HsTyLit` (`HsNumTy`, `HsStrTy`, `HsCharTy`), `HsWildCardTy`, `HsDocTy`
(stripped, annotation discarded), `HsBangTy`, `HsRecTy`.

Not used: `HsSpliceTy` (dead code in the backends), `HsCoreTy` (produces an
error in the backends).

#### `TyClDecl` constructors

All 4: `FamDecl`, `DataDecl`, `SynDecl`, `ClassDecl`.

#### Sig constructors

5: `TypeSig`, `PatSynSig`, `ClassOpSig`, `MinimalSig`, `FixSig`.

#### Other types

`ConDecl` (H98 and GADT), `FamilyDecl`, `FamilyInfo`, `FamilyResultSig`,
`HsDataDefn`, `HsSigType`, `HsForAllTelescope`, `HsTyVarBndr`, `HsBndrVar`,
`HsBndrKind`, `HsBndrVis`, `HsTypeArg`, `HsMultAnn`, `FunDep`,
`InjectivityAnn`, `Fixity`, `BooleanFormula`, `ForeignDecl`.

Names: `Name`, `Module`, `ModuleName`, `OccName`.

#### What the renaming discards

The `GhcRn` -> `DocNameI` renaming explicitly discards: `tcdMeths` (class
default methods), `tcdDocs` (class docs), `dd_derivs` (deriving clauses),
`cid_binds` (instance bindings), `cid_sigs` (instance signatures), and `HsDoc`
identifiers (cleared to `[]`). None of this discarded data is used by any
backend.

### Step 2: Define the Haddock Declaration AST

We create a new package `haddock-ast` with no GHC dependency. It defines
Haskell types mirroring only the needed subset.

The types are not parametrized by a pass index. In the serialized form, names
are plain structured data (`{package, module, occurrence}`) rather than GHC
`Name` values. The types can be instantiated with either GHC names (for the
conversion layer in `haddock-api`) or plain strings (for deserialization).

Top-level structure:

```
HaddockPackage
  version        :: String
  ghcVersion     :: String
  package        :: Maybe PackageInfo
  linkEnv        :: Map NameRef ModuleRef
  modules        :: [HaddockModule]

HaddockModule
  module         :: ModuleRef
  isSignature    :: Bool
  description    :: Maybe Documentation
  info           :: HaddockModInfo
  exports        :: [HaddockExport]
  exportedNames  :: [NameRef]
  visibleNames   :: [NameRef]
  instances      :: [DocInstance]
  fixities       :: [(NameRef, Fixity)]
  warnings       :: [(NameRef, Doc)]
  options        :: [DocOption]
```

Exports:

```
HaddockExport
  = ExportDecl   HaddockDecl DocForDecl [DocInstance] [(NameRef, Fixity)]
  | ExportNoDecl NameRef [NameRef] (Maybe DocForDecl)
  | ExportGroup  Int String (Maybe Doc)
  | ExportDoc    MDoc
  | ExportModule ModuleRef
```

Declarations:

```
HaddockDecl
  = HaddockTyCl  HaddockTyClDecl
  | HaddockSig   HaddockSigDecl
  | HaddockFor   HaddockForeignDecl
```

No `InstD` or `DerivD`: both are consumed by the pipeline but produce no
output in any backend.

`HaddockTyClDecl` has 4 variants (`FamDecl`, `DataDecl`, `SynDecl`,
`ClassDecl`) with fields matching what the backends access. `HaddockType`
has ~20 constructors matching the audit. `HaddockConDecl` has H98 and GADT
variants.

Key design decisions:

- **No extension points.** Unlike GHC's "trees that grow," the Haddock
  Declaration AST has no type family hooks. Extension is handled by versioning
  the JSON schema.
- **Names as data, not references.** In the serialized form, names are
  `{package, module, occurrence}` rather than GHC `Unique`-based identifiers.
  This enables cross-version compatibility.
- **Documentation reuse.** The `DocH` type from `haddock-library` is reused
  as-is for documentation content. `haddock-ast` depends on
  `haddock-library`.

### Step 3: JSON serialization and schema

We implement `ToJSON`/`FromJSON` instances for all types in `haddock-ast`,
using the tagged union pattern: each AST node is a JSON object with a `"tag"`
field and constructor-specific fields.

Example type signature:

```json
{
  "tag": "HsFunTy",
  "multiplicity": "Unannotated",
  "arg": { "tag": "HsTyVar", "promoted": false, "name": { "package": "base", "module": "GHC.Base", "occurrence": "Int" } },
  "result": { "tag": "HsTyVar", "promoted": false, "name": { "package": "base", "module": "GHC.Base", "occurrence": "Bool" } }
}
```

Names are structured objects, not strings:

```json
{ "package": "base", "module": "Data.List", "occurrence": "map" }
```

The existing `Interface/Json.hs` in `haddock-api` already serializes `DocH`
using this tagged union pattern (the `jsonDoc` function). We reuse and extend
this approach.

The JSON Schema is provided alongside this proposal as `schema.json`. It
defines the complete structure and can be used by tools in any language to
validate and parse the output. The schema follows the same conventions as the
GHC JSON diagnostic dump (proposal #050): top-level `version` and `ghcVersion`
fields, semantic versioning, `additionalProperties: false` for forward
compatibility.

### Step 4: Integration into Haddock

Two conversion functions are added to `haddock-api`:

- **`GHC AST -> HaddockDecl`**: implemented in `haddock-api`, where GHC is
  available. This replaces the current `DocNameI` renaming with a conversion
  to the standalone AST. The conversion function takes an `Interface` (which
  contains `[ExportItem GhcRn]`) and produces a `[HaddockExport]`.
- **`HaddockDecl -> rendering`**: the backends are refactored to
  pattern-match on `HaddockDecl`/`HaddockType` instead of
  `HsDecl GhcRn`/`HsType GhcRn`.

The existing pipeline (GHC parse -> synify -> rename -> render) remains
operational. The new pipeline runs in parallel:

- **Emit**: GHC parse -> synify -> convert to `HaddockDecl` -> serialize to
  JSON. Invoked by `--emit-ir` (replaces `--json`).
- **Read**: deserialize JSON to `HaddockDecl` -> render. Invoked by
  `--read-ir`. No GHC involved.

The `--json` flag is replaced by `--emit-ir`. The old behavior (lossy JSON
output without declarations) is superseded by the complete IR output.

### Benefits

- Haddock rendering becomes GHC-free when using `--read-ir`.
- The IR format is versioned and backward-compatible.
- Tools can consume Haskell documentation in any programming language.
- Hackage can store IR at upload time and re-render with any Haddock version,
  solving the frozen documentation problem.
- The `haddock-ast` package is stable: it only changes when Haddock's
  rendering needs change, not when GHC's internal AST changes.
- The `DocNameI` pass and its ~113 type family instances can eventually be
  removed, reducing maintenance burden.

### Drawbacks and risks

- The `haddock-ast` package is a new type to maintain alongside GHC's AST.
  Every change to Haddock's rendering must be reflected in the AST. The risk
  is mitigated because the AST is small (3 decl constructors, ~20 type
  constructors) and well-defined by the audit.
- The conversion from GHC AST to `haddock-ast` must track GHC version changes.
  This is no worse than the current situation (the `DocNameI` instances also
  track GHC), but is now concentrated in one conversion function rather than
  spread across ~113 type family instances.
- The JSON output will be larger than the current binary format. This is
  acceptable for an IR intended for tooling consumption, not for distribution.
  The binary `.haddock` format continues to exist for Haddock's internal
  package reading.
- The Hoogle backend currently relies on GHC's `Outputable` class for
  rendering types. It will need its own type pretty-printer when using the
  standalone AST. This is additional work but straightforward: the HTML and
  LaTeX backends already do their own pretty-printing rather than using
  `Outputable`.

## Stakeholders

- **Haddock maintainers**: the proposal changes the internal AST
  representation and backend interface.
- **Hackage maintainers**: the IR enables re-rendering documentation without
  GHC, addressing the frozen documentation problem. Hackage is the primary
  consumer of the `--emit-ir` output.
- **Hoogle**: the IR provides structured documentation data without requiring
  GHC or `haddock-api`.
- **haskell-language-server**: the IR can be consumed to show documentation
  without running GHC's full pipeline.
- **GHC developers**: the proposal reduces the coupling surface between GHC
  and Haddock, making GHC's AST changes less likely to break Haddock.
- **Tool authors in other languages**: a JSON-based IR enables documentation
  tooling outside the Haskell ecosystem (e.g., `haskell-spotlight`, a
  TypeScript VS Code extension).

## Success

Success criteria:

1. The `haddock-ast` package is defined as a standalone Haskell package with no
   GHC dependency, containing `HaddockDecl`, `HaddockType`, `HaddockExport`,
   and all supporting types.
2. `ToJSON` and `FromJSON` instances are implemented for all types. JSON
   serialization and deserialization round-trip correctly for all constructors.
3. The `--emit-ir` flag produces valid JSON matching the schema for any
   compilable Haskell package.
4. The `--read-ir` flag produces identical HTML output to the existing
   pipeline for the `html-test` test suite in the Haddock repository.
5. The JSON Schema validates against example outputs from the `html-test`
   suite.
