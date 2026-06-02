# Frontend guidelines

Stack: Scala.js 1.x · Slinky 0.7 · React 18 · MUI 5 (ScalablyTyped) · Monix 3

## Rules

**RULE-FE-001 — Component definition pattern**

Every component follows this exact shape:

```scala
object MyWidget {
  def apply(ctx: AppContext, user: User): KeyAddingStage =
    component(Props(ctx, user))

  case class Props(ctx: AppContext, user: User)

  val component: FunctionalComponent[Props] = FunctionalComponent[Props] { props =>
    // hooks, then render
  }
}
```

`def apply` returns `KeyAddingStage` (not `ReactElement`). Props is always a case class.

@file: web/src/main/scala/net/wiringbits/components/widgets/AppBar.scala  
@file: web/src/main/scala/net/wiringbits/components/widgets/SignUpForm.scala

**RULE-FE-002 — Object-level `val` safety**

`val` at the object/companion level is safe **only** for plain JS style objects (`js.Dynamic.literal(...)`, `new CSSPropertiesUtils { ... }`). These hold no React state and are re-usable.

Never store `KeyAddingStage` or `ReactElement` in object-level `val`s. Slinky calls `.build` once, marking the element consumed; a second use throws `IllegalStateException: already been built`.

```scala
// SAFE — plain JS object, no React element
private val cardSx = js.Dynamic.literal(borderRadius = 2, padding = "16px")

// UNSAFE — KeyAddingStage captured at object level
private val myIcon = mui.Icon(muiIcons.Home())  // breaks on second render
```

@file: web/src/main/scala/net/wiringbits/components/widgets/AppBar.scala (navLinkActiveSx, signInBtnSx — safe examples)

**RULE-FE-003 — `def` for elements used more than once**

If a React element must appear in multiple places within one render, use `def` so each call site gets a fresh `KeyAddingStage`:

```scala
// Inside FunctionalComponent body:
def syncArrow = mui.Box()(mui.Icon(muiIcons.Sync()))
// use syncArrow twice — each call creates a new instance
```

@file: web/src/main/scala/net/wiringbits/components/pages/HomePage.scala (`def syncArrow`)

**RULE-FE-004 — State**

| State kind | Mechanism | Example |
|---|---|---|
| Global (auth, dark mode, language) | `AppContext` + monix `Var` with `$` prefix | `ctx.$auth`, `ctx.$darkMode` |
| Read a global reactive value | `ReactiveHooks.useDistinctValue(ctx.$auth)` | — |
| Local form / UI state | `Hooks.useState(initialValue)` | `val (open, setOpen) = Hooks.useState(false)` |
| Derived / memoised | `Hooks.useMemo(() => ..., deps)` | I18nHooks.useMessages |

@file: web/src/main/scala/net/wiringbits/AppContext.scala  
@file: web/src/main/scala/net/wiringbits/core/ReactiveHooks.scala

**RULE-FE-005 — No `useContext` — pass `AppContext` via Props**

React's `useContext` is not used in this codebase. `AppContext` is passed explicitly through `Props` at every level. This keeps the data-flow visible and avoids implicit coupling.

```scala
// Every component takes ctx as the first parameter
def apply(ctx: AppContext, ...): KeyAddingStage = component(Props(ctx, ...))
```

**RULE-FE-006 — `useEffect` and `useMemo` dependency encoding**

Slinky encodes React hook dependencies as follows:

| Intent | Slinky syntax | React equivalent |
|---|---|---|
| Run once (mount only) | `Hooks.useEffect(fn, "")` | `useEffect(fn, [])` |
| Re-run when value changes | `Hooks.useEffect(fn, List(obj))` | `useEffect(fn, [obj])` |
| Memoize once | `Hooks.useMemo(() => expr, "")` | `useMemo(() => expr, [])` |
| Re-memoize when value changes | `Hooks.useMemo(() => expr, List(obj))` | `useMemo(() => expr, [obj])` |

Never pass an empty `List()` as deps — Slinky does not treat it the same as `""`. Use `""` for the run-once / stable-forever case.

```scala
// Run-once memoization (ReCaptcha — avoid re-render on every keystroke)
Hooks.useMemo(
  () => AsyncComponent[String](fetch = ..., render = ...),
  ""
)

// Reactive to Monix Var (ReactiveHooks — subscribe on mount, update on change)
Hooks.useEffect(
  () => {
    val cancelable = value.foreach(setState.apply)
    () => cancelable.cancel()
  },
  List(value)
)
```

@file: web/src/main/scala/net/wiringbits/components/widgets/ReCaptcha.scala  
@file: web/src/main/scala/net/wiringbits/core/ReactiveHooks.scala

**RULE-FE-007 — `AsyncComponent` for data loading**

Use `AsyncComponent` whenever a component needs to fetch data before rendering. It handles loading and error states automatically — you only write the render path.

```scala
AsyncComponent[GetUserLogs.Response](
  fetch = () => props.ctx.api.client.getUserLogs,
  render = response => LogList(props.ctx, response),
  progressIndicator = () => Loader(props.ctx),
  watchedObjects = List(timesRefreshingData)   // re-fetches when this changes
)
```

- `fetch`: `() => Future[T]` — called once on mount (and again when `watchedObjects` changes)
- `render`: `T => ReactElement` — called only on success
- `progressIndicator`: always provide one; `() => Loader(props.ctx)` is the standard
- `watchedObjects`: pass a counter from `GenericHooks.useForceRefresh` to trigger manual refresh; omit if data never needs refreshing
- Error state: rendered automatically by the library; do not re-implement error rendering

@file: web/src/main/scala/net/wiringbits/components/widgets/Logs.scala  
@file: web/src/main/scala/net/wiringbits/components/widgets/UserInfo.scala

**RULE-FE-008 — `onComplete` for form submission**

When submitting a form, call the API and handle the result with `.onComplete`. Never use `.map`, `.flatMap`, or `.recover` for form submission side effects.

```scala
props.ctx.api.client.createUser(request).onComplete {
  case Success(_)  => setFormData(_.submitted)
  case Failure(ex) => setFormData(_.submissionFailed(ex.getMessage))
}
```

The full submit handler pattern:

```scala
def handleSubmit(e: SyntheticEvent[?, dom.Event]): Unit = {
  e.preventDefault()
  if (formData.isSubmitButtonEnabled) {
    setFormData(_.submit)
    for {
      request <- formData.data.submitRequest.orElse {
        setFormData(_.submissionFailed(texts.completeData))
        None
      }
    } yield props.ctx.api.client.doThing(request).onComplete {
      case Success(_)  => setFormData(_.submitted)
      case Failure(ex) => setFormData(_.submissionFailed(ex.getMessage))
    }
  }
}
```

Wrap inputs in an HTML `form` with `onSubmit` so the Enter key works. Never rely on button click alone.

```scala
form(onSubmit := (handleSubmit(_)))(
  // ... inputs, button with type="submit"
)
```

@file: web/src/main/scala/net/wiringbits/components/widgets/SignUpForm.scala

**RULE-FE-009 — Validated types and `FormData`**

Input models are validated primitives defined in `lib/common/shared/`. They extend `WrappedString` with a private constructor:

```scala
// Validate at boundary — returns ValidationResult
Email.validate("user@example.com")   // Right(Email(...)) or Left(error)

// Trust internal data (bypass validation — only in tests or server-side)
Email.trusted("user@example.com")
```

Each form has a `FormData` class (in `web/src/main/scala/net/wiringbits/forms/`) with:

- `fields: List[FormField[?]]` — all fields; used for validation errors
- `formValidationErrors: List[String]` — messages shown under the form
- `submitRequest: Option[Request]` — `Some` only when all fields are valid

State is wrapped in `StatefulFormData` which adds lifecycle:

```scala
val (formData, setFormData) = Hooks.useState(StatefulFormData(MyFormData.initial(...)))

setFormData(_.submit)                       // mark submitting (disables inputs + button)
setFormData(_.submitted)                    // mark done
setFormData(_.submissionFailed(message))    // re-enable, show error
setFormData(_.filling.copy(data = ...))     // update a field while user types
```

@file: web/src/main/scala/net/wiringbits/forms/SignUpFormData.scala  
@file: lib/common/shared/src/main/scala/net/wiringbits/common/models/Email.scala

**RULE-FE-010 — `.withKey()` for mapped lists**

Every element produced by a `.map` over a list must have a unique key:

```scala
items.zipWithIndex.map { case (item, i) =>
  MyRow(ctx, item).withKey(i.toString)
}
```

Omitting `.withKey()` causes React reconciliation warnings and can produce subtle rendering bugs when the list changes.

**RULE-FE-011 — Styling**

Prefer MUI theme tokens over hardcoded hex values:

```scala
// Preferred
js.Dynamic.literal(color = "primary.main", bgcolor = "background.paper")

// Avoid
js.Dynamic.literal(color = "#0E8C6F", bgcolor = "#FFFFFF")
```

Pseudo-selectors (`&:hover`, `&::after`) require `updateDynamic` on the literal because `js.Dynamic.literal` only accepts Scala identifiers as keys:

```scala
val btnSx: js.Object = {
  val obj = js.Dynamic.literal(color = "primary.main", fontWeight = 600)
  obj.updateDynamic("&:hover")(js.Dynamic.literal(background = "rgba(0,0,0,0.08)"))
  obj.asInstanceOf[js.Object]
}
```

Responsive breakpoint values use a nested literal:

```scala
js.Dynamic.literal(
  gridTemplateColumns = js.Dynamic.literal(xs = "1fr", sm = "1fr 1fr", md = "1fr 1fr 1fr")
)
```

@file: web/src/main/scala/net/wiringbits/components/widgets/AppBar.scala (navLinkActiveSx — complete example with ::after)

## Anti-patterns

See [antipatterns.md](antipatterns.md) §Frontend.

## Directory layout

```
web/src/main/scala/net/wiringbits/
  components/
    pages/      Full-page components (one per route)
    widgets/    Reusable sub-components and forms
  core/         ReactiveHooks, I18nHooks, MediaQueryHooks
  forms/        Form data models (StatefulFormData, validation)
  models/       AuthState, User, UserMenuOption
  services/     StorageService
  AppContext.scala
  AppRouter.scala
  AppTheme.scala
```
