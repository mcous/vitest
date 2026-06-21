---
outline: deep
---

# vitest-browser-svelte

The community [`vitest-browser-svelte`](https://npmx.dev/package/vitest-browser-svelte) package renders [Svelte](https://svelte.dev/) components in [Browser Mode](/guide/browser/).

```ts
import { render } from 'vitest-browser-svelte'
import { expect, test } from 'vitest'
import Component from './Component.svelte'

test('counter button increments the count', async () => {
  const screen = await render(Component, {
    initialCount: 1,
  })

  await screen.getByRole('button', { name: 'Increment' }).click()

  await expect.element(screen.getByText('Count is 2')).toBeVisible()
})
```

::: warning
This library takes inspiration from [`@testing-library/svelte`](https://github.com/testing-library/svelte-testing-library).

If you have used `@testing-library/svelte` in your tests before, you can keep using it, however the `vitest-browser-svelte` package provides certain benefits unique to the Browser Mode that `@testing-library/svelte` lacks:

`vitest-browser-svelte` returns APIs that interact well with built-in [locators](/api/browser/locators), [user events](/api/browser/interactivity) and [assertions](/api/browser/assertions): for example, Vitest will automatically retry the element until the assertion is successful, even if it was rerendered between the assertions.
:::

The package exposes two entry points: `vitest-browser-svelte` and `vitest-browser-svelte/pure`. They expose identical API, but the `pure` entry point doesn't add a handler to remove the component before the next test has started.

## render

```ts
export function render<C extends Component, W extends Component>(
  Component: ComponentImport<C>,
  options?: ComponentOptions<C>,
  renderOptions?: SetupOptions<W>,
): Promise<RenderResult<C, W>>
```

The `render` function records a `svelte.render` trace mark, visible in the [Trace View](/guide/browser/trace-view).

### Options

`render` takes two option objects. `options` (the second argument) configures the component itself — its [props](#props) and [`mount`](https://svelte.dev/docs/svelte/imperative-component-api#mount) options like [`target`](#target). `renderOptions` (the third argument) configures the surrounding document and queries — [`baseElement`](#baseelement), [`wrapper`, and `wrapperProps`](#wrapper-and-wrapperprops).

#### props

Component props. The `options` argument supports either options that you can pass down to [`mount`](https://svelte.dev/docs/svelte/imperative-component-api#mount) or props directly:

```ts
const screen = await render(Component, {
  props: {
    // [!code --]
    initialCount: 1, // [!code --]
  }, // [!code --]
  initialCount: 1, // [!code ++]
})
```

#### target

By default, Vitest will create a `div`, append it to `document.body`, and render your component there. If you provide your own `HTMLElement` container, it will not be appended automatically — you'll need to call `document.body.appendChild(container)` before `render`.

For example, if you are unit testing a `tbody` element, it cannot be a child of a `div`. In this case, you can specify a `table` as the render container.

```ts
const table = document.createElement('table')

const screen = await render(TableBody, {
  props,
  // ⚠️ appending the element to `body` manually before rendering
  target: document.body.appendChild(table),
})
```

#### baseElement

The element that queries are scoped to and that [`debug`](#debug) prints. Defaults to [`target`](#target) if set, otherwise `document.body`. You should rarely, if ever, need to set `baseElement`.

#### wrapper and wrapperProps

Pass `wrapper` and `wrapperProps` to the `renderOptions` object to render your component as the child of another, e.g. a [context](https://svelte.dev/docs/svelte/context) provider it depends on.

```ts
const screen = await render(
  Component,
  { initialCount: 1 }, // props for `Component`
  { wrapper: Provider, wrapperProps: { theme: 'dark' } }, // renderOptions
)
```

See [Wrappers](#wrappers) for a complete example.

::: tip
If you can't test a component in isolation without a wrapper, this may be a sign that you're testing at the wrong level, or that your component structure should be rethought. Consider whether you can make your components more testable in isolation before reaching for a wrapper.
:::

### Render Result

In addition to documented return value, the `render` function also returns all available [locators](/api/browser/locators) relative to the [`baseElement`](#baseelement), including [custom ones](/api/browser/locators#custom-locators).

```ts
const screen = await render(TableBody, props)

await screen.getByRole('link', { name: 'Expand' }).click()
```

#### container

The containing DOM node where your Svelte component is rendered. This is a regular DOM node, so you technically could call `container.querySelector` etc. to inspect the children.

:::danger
If you find yourself using `container` to query for rendered elements then you should reconsider! The [locators](/api/browser/locators) are designed to be more resilient to changes that will be made to the component you're testing. Avoid using `container` to query for elements!
:::

#### component

The mounted Svelte component instance. You can use this to access component methods and properties if needed.

```ts
const { component } = await render(Counter, {
  initialCount: 0,
})

// Access component exports if needed
```

#### wrapper

The mounted [`wrapper`](#wrapper-and-wrapperprops) component instance, if one was provided. Otherwise `undefined`. Exposes the wrapper's exports, like [`component`](#component) does.

#### locator

The [locator](/api/browser/locators) of your `container`. It is useful to use queries scoped only to your component, or pass it down to other assertions:

```ts
import { render } from 'vitest-browser-svelte'

const { locator } = await render(NumberDisplay, {
  number: 2,
})

await locator.getByRole('button').click()
await expect.element(locator).toHaveTextContent('Hello World')
```

#### debug

```ts
function debug(el?: HTMLElement | HTMLElement[] | Locator | Locator[]): void
```

This method is a shortcut for `console.log(prettyDOM(baseElement))`. It will print the DOM content of the container or specified elements to the console.

#### rerender

```ts
function rerender(props: Partial<ComponentProps<T>>): Promise<void>
```

Updates the component's props and waits for Svelte to apply the changes. Use this to test how your component responds to prop changes. Also records a `svelte.rerender` trace mark in the [Trace View](/guide/browser/trace-view).

```ts
import { render } from 'vitest-browser-svelte'

const { rerender } = await render(NumberDisplay, {
  number: 1,
})

// re-render the same component with different props
await rerender({ number: 2 })
```

#### unmount

```ts
function unmount(): Promise<void>
```

Unmount and destroy the Svelte component. Also records a `svelte.unmount` trace mark in the [Trace View](/guide/browser/trace-view). This is useful for testing what happens when your component is removed from the page (like testing that you don't leave event handlers hanging around causing memory leaks).

```ts
import { render } from 'vitest-browser-svelte'

const { container, unmount } = await render(Component)
await unmount()
// your component has been unmounted and now: container.innerHTML === ''
```

## cleanup

```ts
export function cleanup(): void
```

Remove all components rendered with [`render`](#render).

## Extend Queries

To extend locator queries, see [`"Custom Locators"`](/api/browser/locators#custom-locators). For example, to make `render` return a new custom locator, define it using the `locators.extend` API:

```ts {5-7,12}
import { locators } from 'vitest/browser'
import { render } from 'vitest-browser-svelte'

locators.extend({
  getByArticleTitle(title) {
    return `[data-title="${title}"]`
  },
})

const screen = await render(Component)
await expect.element(screen.getByArticleTitle('Hello World')).toBeVisible()
```

## Wrappers

Sometimes a component can only render or operate as the child of another component, e.g. one that provides [context](https://svelte.dev/docs/svelte/context). Pass [`wrapper` and `wrapperProps`](#wrapper-and-wrapperprops) to wrap the component under test.

::: code-group

```js [child.test.js]
import { render } from 'vitest-browser-svelte'
import { expect, test } from 'vitest'

import Subject from './child.svelte'
import Wrapper from './wrapper.svelte'

test('notifications with messages from context', async () => {
  const messages = [
    { id: 'abc', text: 'hello' },
    { id: 'def', text: 'world' },
  ]

  const screen = await render(
    Subject,
    { label: 'Notifications' },
    { wrapper: Wrapper, wrapperProps: { messages } },
  )

  const status = screen.getByRole('status', { name: 'Notifications' })

  await expect.element(status).toHaveTextContent('hello world')
})
```

```svelte [child.svelte]
<script>
  import { getContext } from 'svelte'

  let { label } = $props()

  const messages = getContext('messages')
</script>

<div role="status" aria-label={label}>
  {#each messages.current as message (message.id)}
    <p>{message.text}</p>
  {/each}
</div>
```

```svelte [wrapper.svelte]
<script>
  import { setContext } from 'svelte'

  let { messages, children } = $props()

  setContext('messages', {
    get current() {
      return messages
    },
  })
</script>

{@render children?.()}
```

:::

## Snippets

For simple snippets, you can use a wrapper component and "dummy" children to test them. Setting `data-testid` attributes can be helpful when testing slots in this manner.

::: code-group

```ts [basic.test.js]
import { render } from 'vitest-browser-svelte'
import { expect, test } from 'vitest'

import SubjectTest from './basic-snippet.test.svelte'

test('basic snippet', async () => {
  const screen = await render(SubjectTest)

  const heading = screen.getByRole('heading')
  const child = heading.getByTestId('child')

  await expect.element(child).toBeInTheDocument()
})
```

```svelte [basic-snippet.svelte]
<script>
  let { children } = $props()
</script>

<h1>
  {@render children?.()}
</h1>
```

```svelte [basic-snippet.test.svelte]
<script>
  import Subject from './basic-snippet.svelte'
</script>

<Subject>
  <span data-testid="child"></span>
</Subject>
```

:::

For more complex snippets, e.g. where you want to check arguments, you can use Svelte's [`createRawSnippet`](https://svelte.dev/docs/svelte/svelte#createRawSnippet) API.

::: code-group

```js [complex-snippet.test.js]
import { render } from 'vitest-browser-svelte'
import { createRawSnippet } from 'svelte'
import { expect, test } from 'vitest'

import Subject from './complex-snippet.svelte'

test('renders greeting in message snippet', async () => {
  const screen = await render(Subject, {
    name: 'Alice',
    message: createRawSnippet(greeting => ({
      render: () => `<span data-testid="message">${greeting()}</span>`,
    })),
  })

  const message = screen.getByTestId('message')

  await expect.element(message).toHaveTextContent('Hello, Alice!')
})
```

```svelte [complex-snippet.svelte]
<script>
  let { name, message } = $props()

  const greeting = $derived(`Hello, ${name}!`)
</script>

<p>
  {@render message?.(greeting)}
</p>
```

:::

## See also

- [Svelte Testing Library documentation](https://testing-library.com/docs/svelte-testing-library/intro)
- [Svelte Testing Library examples](https://github.com/testing-library/svelte-testing-library/tree/main/examples)
