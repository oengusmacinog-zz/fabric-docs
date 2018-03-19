## Using `componentRef` Instead of `ref`

React exposes a special `ref` prop for components. If you use `ref` on an intrinsic elements such as divs or spans, React will give you a reference to the element, allowing you access to the public API for the elements.

Using `ref` to access component public methods is more error prone:

1. If the component wraps itself in a higher order component or decorator, the `ref` will return the wrapper component rather than what you intended to access.

2. Accessing the full component's public methods is probably not desirable. It isn't exactly intended to access `public render` or to allow the consumer to call `componentWillUnmount`, despite these being publicly exposed to React.

### Consuming a component's public API

There ARE limited cases where a component should expose a public API contract. Usually, things can be more declarative in React, but some scenarios which are perhaps easier to do imperatively are:

1. Exposing a `focus` method.
2. Accessing current values of uncontrolled prop values (such as the current value of a `TextField`.)

In Fabric, we do the following for components which expose an imperative API surface:

1. Define an `I{ComponentName}` interface which only exposes the methods intended to be supported.
2. Every component supports a `componentRef` prop, which will return the `I{ComponentName}` interface if provided.

So, use `componentRef` as a drop-in replacement for `ref`.

Example usage of `componentRef` to access the `IButton` interface:

```tsx
private _primaryButton: IButton = React.createRef<IButton>();

public render() {
  return <PrimaryButton componentRef={ this._primaryButton } ... />;
}

public componentDidMount() {
  this._primaryButton.value.focus();
}
```

### In Fabric, extend `BaseComponent` to abstract resolving the componentRef

If your component extends `BaseComponent`, the `componentRef` will be auto-resolved for you and you just need to focus on exposing it in the prop typings.

If you were to write a component which extended vanilla `React.Component`, you would need to resolve it manually. For example, on mount:

```tsx
public componentDidMount() {
  const { componentRef } = this.props;
  componentRef && componentRef(this);
}
```

...and clear it on unmount:

```tsx
public componentWillUnmount() {
  const { componentRef } = this.props;
  componentRef && componentRef(undefined);
}
```

