- Start Date: 2026-03-30
- RFC PR: [#112](https://github.com/inveniosoftware/rfcs/pull/112)
- Authors: Miroslav Bauer <bauer@cesnet.cz>
- State: DRAFT

# React v16 to v18 Migration

## Summary


Upgrade Invenio JavaScript packages from React 16.13.0 (EOL since 2020) to React 18.3.1 using `@cfaester/enzyme-adapter-react-18` to maintain existing tests, minimizing initial migration effort. Standardize test & build tooling (Jest, Eslint, RSPack) and enable future adoption of React 18 concurrent features and TypeScript as optional post-release enhancements.

## Motivation

React 16 reached EOL in March 2020, creating compounding problems:

- **Security exposure**: npm audit flags vulnerabilities with no upstream fix path, reliance on many nowadays unmaintained dependencies.
- **Ecosystem incompatibility**: Existing packages & tools dropping React 16 support. New ones totally incompatible.
- **Contributor friction**: New developers encounter legacy patterns they're not familiar with & have no good reason to learn.
- **Performance limitations**: Cannot leverage automatic batching, concurrent features, UX improvements.

**User stories:**
1. As a new frontend contributor, I want to use modern React patterns so I can submit PRs without learning lagacy React 16-specific patterns.
2. As a core team developer, I want code reviews focused on logic rather than deprecated lifecycle methods.
3. As a module maintainer, I want to upgrade peer dependencies without forking libraries.
4. As an InvenioRDM developer using TypeScript, I want proper type exports for IDE support.

**Minimal expected outcome:** All Invenio React packages on React 18.3.x with unified build & test tooling and preserved test coverage via Enzyme adapter.

## Detailed design

The migration is split into phases aligned to the InvenioRDM release train. Every package requires a major version bump (breaking changes are introduced).

### Phase 0 - Scope of changes, Prerequisites & Tooling

#### Prerequisites

- Node.js 18.0.0+ (LTS, React 18 compatible)
- Jest 27.0.0+ (React 18 compatible test runner)

#### Breaking Changes Summary

**React 18 Breaking Changes:**

1. **createRoot API** (Entry Points)
   - `ReactDOM.render()` → `createRoot().render()`
   - `ReactDOM.hydrate()` → `hydrateRoot()` (for SSR)
   - All entry points must be updated

2. **Automatic Batching**
   - State updates batched even in setTimeout/promises
   - Code reading state immediately after setState may see stale values
   - Use `flushSync()` for synchronous reads (escape hatch)

3. **Event Delegation**
   - Events no longer bubble to `document`
   - Click-outside handlers must use capture phase or ref-based approach

4. **StrictMode**
   - Components unmount/remount in development to detect side effects
   - Exposes side effects in constructors and effects
   - Requires proper cleanup functions

**Dependency Breaking Changes:**

| Package | Change | Impact |
|---------|--------|--------|
| `@tinymce/tinymce-react` | v4 → v6 | Editor API changes, event handlers |
| `tinymce` | v6.7.2 → v7.0.0 | Core library updates |
| `react-json-view` | Remove, use `@microlink/react-json-view` | Import path change |
| `@ckeditor/ckeditor5-react` | ^2.1.0 | Complete removal | Any CkEditor usage breaks | invenio-vocabularies, invenio-communities |
| `@ckeditor/ckeditor5-build-classic` | ^16.0.0 | Complete removal | Any CkEditor usage breaks | invenio-vocabularies |

**Package Version Bumps (Major):**
- `react-overridable`: v1.x → v2.0.0
- `react-invenio-forms`: v4.x → v5.0.0
- `react-searchkit`: v3.x → v4.0.0
- All Python packages with React entry points: major version bump

#### Implementation Workflow & Release Strategy

##### Repository Workflow

**For each package:**

1. **Fork & Branch**
   - Fork repository
   - Create branch: `react-18-migration`

2. **Development**
   - Install Enzyme adapter
   - Apply codemods
   - Update dependencies
   - Run tests locally

3. **Pull Request**
   - Create PR with breaking changes documented
   - Label: `react-18-migration`
   - Link to this RFC

4. **Merge & Release**
   - Merge when tests pass
   - Follow release sequence below

##### Release Strategy

Following [InvenioRDM Release Management](https://inveniordm.docs.cern.ch/maintenance/operations/release-management/):

**Versioning:**
- Semantic versioning strictly followed
- Breaking changes = major version bump
- Development versions: `v(X+1).0.0.dev0`

**Release Sequence (Dependency Order):**

Release `v(X+1).0.0.dev0` in order:

1. `react-overridable` v2.0.0.dev0 (no dependencies)
2. `react-invenio-forms` v5.0.0.dev0 (depends on #1)
3. `react-searchkit` v4.0.0.dev0 (depends on #1)
4. `invenio-theme`
5. `invenio-search-ui` (depends on #3)
6. `invenio-administration` (depends on #2)
7. `invenio-requests` (depends on #2)
8. `invenio-vocabularies` (depends on #2, #3)
9. `invenio-communities` (depends on #5, #6, #7, #8)
10. `invenio-rdm-records` (depends on #2, #3, #6)
11. `invenio-jobs` (depends on #2, #3)
12. `invenio-app-rdm` (top-level, depends on all above)
13. `cookiecutter-invenio-rdm` (template, depends on #12)

**Release Process:**
1. **Development:** `v(X+1).0.0.dev0` during development
2. **Pre-release:** Remove `.dev0` suffix when ready
3. **Release:** Create git tag `vX.Y.Z`, push to npm/PyPI

**Release Checklist (Per Package):**
- [ ] All dependent modules released with compatible versions
- [ ] `CHANGES.md` updated with breaking changes
- [ ] Version bumped in `package.json` / `__init__.py`
- [ ] Tests pass with React 18
- [ ] Integration tests pass (oarepo/invenio-testrig)

##### Rollback Procedure

If critical issues discovered post-release:

1. **Immediate:** Pin previous version in affected instances
2. **Communication:** Post in Discord/mailing list
3. **Hotfix:** Create branch from last stable tag
4. **Release:** Issue maintenance release following same process

**Rollback Criteria:**
- Critical security vulnerability in React 18 ecosystem
- Unresolvable breaking change in third-party dependency
- Performance regression >20% in core workflows
- >5 community reports of broken instance within 48 hours of release

#### Testing Strategy

**Primary Approach: Enzyme Adapter for React 18**

Use `@cfaester/enzyme-adapter-react-18` to maintain existing tests with minimal changes:

```javascript
import Enzyme from 'enzyme';
import Adapter from '@cfaester/enzyme-adapter-react-18';

Enzyme.configure({ adapter: new Adapter() });
```

**Known Limitations:**
- Wrap `simulate()` calls in `act()`:
```javascript
await act(() => {
  mountWrapper.find('form').simulate('submit');
});
```

**Risk Assessment:**
- **Status:** Unofficial community adapter (not maintained by Enzyme team)
- **Last updated:** March 2023 (used by react-invenio-app-ils)
- **Author disclaimer:** "Should you count on it? Probably not"
- **Maintenance:** Limited/non-existent support expected
- **Mitigation:** Fork if critical issues arise; RTL migration planned as Phase 3 optional enhancement

**Alternative:** React Testing Library (RTL) - see Phase 3 for optional migration

#### RSPack Configuration

**Critical Dependency:** Prepare RSPack build config before any migration work.

Requirements:
- `@babel/preset-react` with `runtime: 'automatic'` (no need to explicitly import React anymore)
- JSX transform compatible with React 18
- TypeScript loader (for future adoption)
- Tree-shaking for smaller bundles

#### Third-Party Package Updates/Replacement

Several dependencies require update/replacement for React 18 compatibility.

| Package | Current | Target | Breaking Changes | Affected Packages |
|---------|---------|--------|------------------|-------------------|
| `react-json-view` | ^1.21.3 | **REMOVE** | Deprecated | invenio-app-rdm |
| `@microlink/react-json-view` | - | ^1.31.15 | Drop-in replacement | invenio-app-rdm, invenio-jobs |
| `@tinymce/tinymce-react` | ^4.3.0 | ^6.0.0 | Editor API changes | 5 packages* |
| `tinymce` | ^6.7.2 | ^7.0.0 | Core library | Same 5 packages |
| `@ckeditor/ckeditor5-react` | ^2.1.0 | **REMOVE** | Complete removal | invenio-vocabularies, invenio-communities |
| `@ckeditor/ckeditor5-build-classic` | ^16.0.0 | **REMOVE** | Complete removal | invenio-vocabularies |

*invenio-app-rdm, invenio-rdm-records, invenio-communities, invenio-requests, invenio-administration

**react-json-view Migration:**
```python
# Before (webpack.py)
"react-json-view": "^1.21.3"

# After
"@microlink/react-json-view": "^1.31.15"
```

**@tinymce/tinymce-react v4 → v6 Breaking Changes:**
1. Editor component initialization API changes
2. Event handler signature changes (onEditorChange)
3. Deprecated props removed

**Coordination Required:** All 5 packages must update simultaneously to avoid version conflicts.

#### Codemods & Migration Tooling

Set-up shared repository containing codemods:

**Execution Order:**
1. `rename-unsafe-lifecycles` - Fix class components
2. `replace-reactdom-render` - Update entry points FIRST
3. `update-react-imports` - Clean up React imports
4. `React-PropTypes-to-prop-types` - Normalize PropTypes
5. `pure-component` - Convert safe classes to functions
6. `types-react-codemod preset-18` - TypeScript packages only

**Note:** `replace-reactdom-render` must run before `pure-component`.

Official mods: https://github.com/reactjs/react-codemod/tree/master/transforms

#### ESLint Configuration

Create a separate React 18 config to enable gradual migration:

**File: `configs/react18.yaml`**
```yaml
extends:
  - ./main.yaml

rules:
  react/react-in-jsx-scope: off
  no-restricted-imports:
    - error
    - paths:
        - name: enzyme-adapter-react-16
          message: "Remove, use @cfaester/enzyme-adapter-react-18"
  react-hooks/rules-of-hooks: error
  react-hooks/exhaustive-deps: warn
```

**Package.json exports:**
```json
{
  "exports": {
    ".": "./configs/main.yaml",
    "./react18": "./configs/react18.yaml"
  }
}
```

**Migration path:**
| Phase | Config | Use Case |
|-------|--------|----------|
| Legacy | `@inveniosoftware/eslint-config-invenio` | React 16 support (existing projects) |
| Migration | `@inveniosoftware/eslint-config-invenio/react18` | React 18 migration (opt-in per package) |

This allows packages to migrate individually without breaking existing React 16 consumers.

#### Unify test & lint tooling

At the root of each package, provide:

- `run-js-tests.sh` to run any Jest tests provided in package
- `run-js-lint.sh` to lint any JS code against `eslint-config-invenio` rules

**Implementation:** PR https://github.com/inveniosoftware/workflows/pull/26 adds CI support for `run-js-tests.sh` - the workflow attempts to execute the script and falls back to `npm test` if not present, maintaining backward compatibility.


#### StrictMode Compliance

React 18 StrictMode simulates mount-unmount-remount cycles in development to detect side effects.

**What StrictMode Does (Development Only):**

React 18 StrictMode intentionally double-invokes certain functions in development to detect side effects. This is a breaking change from React 16/17.

| Function/Lifecycle | React 16 | React 18 StrictMode |
|-------------------|----------|---------------------|
| Component lifecycle | Mount once | Mount → unmount → remount |
| Effect cleanup/setup | 1× each | 2× each (once per mount) |
| Ref callbacks | 1× | 2× (null, then value) |

**Common Patterns That Break:**

```javascript
// ❌ BROKEN: Constructor side effects
class SearchController extends Component {
  constructor(props) {
    super(props);
    this.api = new SearchApi(props.config); // Created per mount!
  }
}

// ✅ FIXED: Move to componentDidMount with cleanup
class SearchController extends Component {
  componentDidMount() {
    this.api = new SearchApi(this.props.config);
  }
  
  componentWillUnmount() {
    this.api?.cleanup?.();
  }
}

// ❌ BROKEN: useMemo for unique IDs
function useUniqueId() {
  return useMemo(() => `id-${Math.random()}`, []); // Different per mount!
}

// ✅ FIXED: Use React 18's useId hook
function Component() {
  const id = useId(); // Stable across remounts
}

// ❌ BROKEN: Effect without cleanup
useEffect(() => {
  const handler = () => setOpen(false);
  document.addEventListener('click', handler); // Leaked on double-run!
}, []);

// ✅ FIXED: Always provide cleanup
useEffect(() => {
  const handler = () => setOpen(false);
  document.addEventListener('click', handler);
  return () => document.removeEventListener('click', handler);
}, []);
```

**StrictMode Migration Checklist:**
- [ ] All tests pass with `<StrictMode>` wrapper
- [ ] No console warnings about deprecated APIs
- [ ] Constructor side effects moved to lifecycle methods
- [ ] All effects have proper cleanup functions
- [ ] Ref callbacks handle `null → value` transitions
- [ ] `useMemo` not used for unique ID generation (use `useId`)
- [ ] Event listeners properly cleaned up

**Testing with StrictMode:**
```javascript
import { StrictMode } from 'react';

render(
  <StrictMode>
    <Component />
  </StrictMode>
);
```

#### Entry Point Migration

The new `createRoot` API requires proper cleanup:

```javascript
// Before (React 16)
import ReactDOM from 'react-dom';
ReactDOM.render(<App />, document.getElementById('root'));

// After (React 18)
import { createRoot } from 'react-dom/client';

const container = document.getElementById('root');
const root = createRoot(container);
root.render(<App />);

// Cleanup for hot reloading, tests
// root.unmount();
```

#### Event Delegation Changes

React 18 changes event delegation from `document` to the root container:

- `e.stopPropagation()` stops at root, not document
- Click-outside detection for modals/dropdowns may break

```javascript
// React 16: This worked because events bubbled to document
document.addEventListener('click', (e) => {
  if (!modalRef.current.contains(e.target)) {
    closeModal();
  }
});

// React 18: Must listen at root or use capture phase
document.addEventListener('click', handleClick, true); // capture phase
```

### Phase 1 - Core NPM Packages

These are dependencies of most other front-end packages:

| Package | Changes |
|---------|---------|
| `react-overridable` | React 18.3 (dev), classic JSX transform, backward compatible with 16/17 |
| `react-invenio-forms` | React 18.3, Formik ^2.1.0, Enzyme adapter |
| `react-searchkit` | React 18.3, Enzyme adapter, JavaScript |

**Note on react-overridable implementation:** PR https://github.com/indico/react-overridable/pull/36 uses a **non-breaking approach** - peer deps remain `>=16.14.0`, classic JSX transform maintains React 16-18 compatibility.

**For each of these:**
1. Install `@cfaester/enzyme-adapter-react-18@^0.8.0`
2. Update test configuration
3. Bump React packages to v18.3.x
4. Apply available codemods to establish baseline
5. Manual work on remaining code
6. Validate by running tests & lint
7. Update documentation

**Note:** Tests remain on Enzyme. RTL migration is optional post-release (Phase 3).

**TypeScript preparation:** Set up build infrastructure, but do not convert source code yet. This enables future adoption without blocking React 18 release.

### Phase 2 - Python Packages with React Entry Points

Migrate all Python packages declaring React entry points in `webpack.py`:

`invenio-app-rdm`, `invenio-rdm-records`, `invenio-communities`, `invenio-requests`, `invenio-search-ui`, `invenio-administration`, `invenio-theme`

**Migration order:** Leaf packages first, `invenio-app-rdm` last

**Steps per Python package:**
1. Install Enzyme adapter in package dependencies
2. Update `webpack.py` React dependencies
3. Apply codemods to entry points (e.g. ReactDOM.render → createRoot)
4. Fix StrictMode issues, manual fixes
5. Validate locally (Jest tests, lint)
6. Run integration tests (oarepo/invenio-testrig)
7. Update documentation

### Phase 3 - Post-Release Optional Enhancements

**These enhancements are optional and can only begin after Phases 0-2 are complete and released.**

#### 1. Automatic Batching

Already active in React 18 - no implementation work required. Code relying on synchronous state reads may need `flushSync()`:

```javascript
// React 18 - state is batched, use flushSync for sync updates
import { flushSync } from 'react-dom';

handleClick = () => {
  flushSync(() => {
    this.setState({ count: 1 });
  });
  // Now guaranteed to see updated state
  console.log(this.state.count); // 1
};
```

#### 2. React Testing Library Migration (Optional)

Gradual migration from Enzyme to RTL:

- New tests should use RTL
- Legacy tests can remain on Enzyme adapter
- Migrate critical tests first
- Full migration timeline TBD

**Benefits:** Better long-term support, community standard

#### 3. useTransition

Mark non-urgent state updates as transitions:

```javascript
import { startTransition } from 'react';

const handleSearchChange = (value) => {
  setSearchQuery(value); // Urgent
  startTransition(() => {
    setFilterResults(filterItems(value)); // Can be interrupted
  });
};
```

#### 4. useDeferredValue

Defer re-rendering non-urgent parts:

```javascript
const [query, setQuery] = useState('');
const deferredQuery = useDeferredValue(query);
const results = useSearchResults(deferredQuery);
```

#### 5. useId for Accessibility

Generate stable unique IDs:

```javascript
import { useId } from 'react';
const id = useId();
```

#### 6. Suspense for Code Splitting

Declarative loading states for lazy-loaded components.

#### 7. react-redux v8 Upgrade (Optional)

Consider upgrading from ^7.2.0 to v8+ for better concurrent rendering support.

**Breaking Changes:**
- Subscription timing changes
- Selector stability requirements (use reselect)

#### 8. TypeScript Adoption

Gradual migration of packages to TypeScript, starting with public API types.

#### 9. Formik Upgrade (Optional)

Upgrade Formik from ^2.1.0 to ^2.4.0+ for latest bug fixes and improvements.

**Benefits:**
- Latest bug fixes
- Better React 18 compatibility
- Performance improvements

**Note:** This can be done independently of React 18 migration.

### Phase 4 - Documentation

Create a migration guide for the InvenioRDM community, based on applied patterns & final package versions, taking them from current React v16.13 front-end to fully v18.

## Example

### Entry Point Migration

**Before (React 16):**
```javascript
// invenio_app_rdm/theme/assets/semantic-ui/js/invenio_app_rdm/deposit/index.js
import ReactDOM from 'react-dom';
ReactDOM.render(<DepositForm />, document.getElementById('deposit-form'));
```

**After (React 18 with Enzyme Adapter):**
```javascript
// Entry point migration
import { createRoot } from 'react-dom/client';
const container = document.getElementById('deposit-form');
const root = createRoot(container);
root.render(<DepositForm />);
```

**Test Setup:**
```javascript
// test/setup.js
import Enzyme from 'enzyme';
import Adapter from '@cfaester/enzyme-adapter-react-18';

Enzyme.configure({ adapter: new Adapter() });
```

## How we teach this

### Official React Documentation

- [React.dev](https://react.dev/) - Main React documentation
- [React 18 Upgrade Guide](https://react.dev/blog/2022/03/08/react-18-upgrade-guide)
- [React Hooks Reference](https://react.dev/reference/react)
- [Testing Library Documentation](https://testing-library.com/docs/react-testing-library/intro/)

### InvenioRDM Documentation Updates

1. **Update React best practices guide:**
   https://inveniordm.docs.cern.ch/community/code/best-practices/react/
   - Document React 18 migration patterns
   - Add RTL migration guide (optional)
   - Add Typescript guide (optional)

2. **Create RDM-specific React 16 to 18 Migration guide:**
   - Step-by-step migration instructions
   - Before/after code comparisons
   - Common pitfalls and solutions

### User Impact

- No user-facing changes expected during core migration
- Performance improvements may be noticeable after Phase 3
- Gradual rollout reduces risk
- Improved responsiveness in search/filter operations (after useDeferredValue adoption)

## Drawbacks


| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| react-overridable maintained by other party | release could be delayed | High | Need to initiate early discussion, connect with the maintainers |
| Ecosystem package delays | High | Medium | Parallel workstreams, early forks |
| Test coverage gaps | High | Medium | Phase 0-1 focus on critical path testing |
| Enzyme adapter unmaintained | Medium | Medium | Fork if needed; RTL as fallback |
| Performance regression | Medium | Low | Benchmark tests, monitoring |
| IE11 support dropped | Medium | Low | Assess user impact; document workaround |
| StrictMode side-effect exposure | High | High | Move side effects from constructors |
| Event delegation breaking changes | Medium | Medium | Test modal click-outside detection |

### Known Limitations

1. **Error Boundaries**: Must remain class components (React API limitation).
2. **Enzyme Adapter Risk**: Unofficial adapter with limited maintenance.
3. **IE11 Deprecation**: React 18 drops IE11 support.
4. **StrictMode**: Development-only double-invocation exposes side effects.
5. **Automatic Batching**: Code relying on sync state reads may need `flushSync()`.
6. **Semantic UI React**: Must verify Modal/Portal components work with React 18 event delegation.
7. **CkEditor Removal**: Complete removal affects any instances using CkEditor.

## Alternatives

### Alternative 1: Stay on React 16

**Impact:**
- No migration effort required
- React 16 is EOL since March 2020
- Cannot use modern React features
- Growing ecosystem incompatibility
- Recruitment challenges

**Recommendation:** Not viable long-term

### Alternative 2: Skip to React 19

**Impact:**
- Newer version available
- Higher risk (React 19 is newer, less battle-tested)
- Less ecosystem support
- Not supported by Semantic-UI React

**Recommendation:** React 18 is safer choice

### Alternative 3: Migration without Phase 3 Features

**Impact:**
- Core migration only
- Can adopt features incrementally post-release
- Lower initial risk

**Recommendation:** Proceed with this approach

## Unresolved questions

1. **CI/CD Integration**: How to integrate migration testing into existing pipelines?
2. **Class Components Strategy**: Keep, or partially rewrite to functional?
3. **@tinymce/tinymce-react Version**: v5 (safer) or v6 (better React 18 support)?
4. **IE11 Impact Assessment**: Determine impact on InvenioRDM user base.
5. **Semantic UI React Verification**: Test Modal/Portal with React 18 event delegation.

## Resources/Timeline

### Resources

CESNET has committed frontend developer resources for implementation. Coordination with Invenio core team for architectural review and testing infrastructure.
