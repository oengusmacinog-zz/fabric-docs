## New Components

### Styling

When a consumer uses a component, providing style overrides for the component should not be a fragile guessing game. We should be able to easily tweak component styling and also create variants for subcomponents without a lot of complex effort that's bound to break later on.

A component consists of dom elements, or "areas". Each of the areas should be targetable for styling. The styling applied to each area may depend on the state of the component as well as the contextual theme settings. So, styling should be defined as a function of these inputs:

```tsx
// Take in styling input, spit out styles for each area.
function getStyles(props: IComponentStyleProps): IComponentStyles {
  return {
    root: { /* styles */ },
    child1: { /* styles */ },
    child2: { /* styles */ }
  };
}
```

With this in mind, let's make `getStyles` an optional prop to the component. Now style overrides can be applied in a functional way and even be conditionalized depending on the component state:

```tsx
const getStyles = props => ({
  root: [
    {
      background: props.theme.palette.themePrimary,
      selectors: {
        ':hover': {
          background: props.theme.palette.themeSecondary,
        }
      }
    },
    props.isExpanded
    ? { display: 'block' }
    : { display: 'none' }
  ]
});

<MyComponent getStyles={ myStyleOverrides } />
```

Quite often, variants of a component which use the style must be created to abstract some customizations. A `styled` HOC is provided which can make creating variants simple:

```tsx
import { styled } from 'office-ui-fabric-react/lib/Styling';
import { getStyles } from './MyComponentVariant.styles';

const MyComponentVariant = styled(
  MyComponent,
  getStyles
  })
);
```

Within the component implementation, when we need to convert these styles into class names, we will use a helper called `classNameFunction` to create a memoized function which can translate the style objects into strings:

```tsx
const getClassNames = classNameFunction<ICompStyleProps, ICompStyles>();

class Comp extends React.Component {
  public render() {
    const { getStyles, theme } = this.props;
    const classNames = getClassNames(getStyles, { /* style props */ });

    return (
      <div className={ classNames.root }>...</div>
    );
  }
}
```

#### Component anatomy

A component should consist of these files:

* `ComponentName.props.ts` - The interfaces for the component. We separate these out for documentation reasons.
* `ComponentName.base.tsx` - The unstyled component. This renders DOM structure and conrtains logic, MINUS styling opinions.
* `ComponentName.styles.ts` - Exports a `getStyles` function for the component which takes in `IComponentNameStyleProps` and returns `IComponentNameStyles`.

Once these are defined, you can export the component which ties it all together:

* `ComponentName.tsx` - Using the `styled` helper, exports a new component tying the base component to 1 or more style helpers.

Additionally, each component should have these:

* `ComponentName.checklist.ts` - A checklist status export which allows the documentation to render notes on what validation has been done on the component.
* `ComponentName.test.tsx` - Unit tests for the component.

The idea is that components, especially reusable atomic components, should by default be exported unstyled. This gives us the flexibility to create variants.

#### ComponentName.props.ts changes

The props file should contain these 4 interfaces, in addition to any enums or consts externally required:

1. **IComponentName** - The public method accessible through `componentRef`. This should include the `focus` method, as well as getters for important values like `checked` in the case the component will be referenced and the value may be read manually. Example:

```tsx
export IComponentName {
  focus: () => void;
}
```

2. **IComponentNameProps** - The props for the component. This should include the `componentRef` prop for accessing the public interface, the `theme` prop (which will be injected by the `@customizable` decorator), as well as the `getStyles` function.

Example:

```tsx
export IComponentNameProps extends React.Props<ComponentNameBase> {

  componentRef?: (componentRef: IComponentName) => void;

  theme?: ITheme;

  getStyles?: IStyleFunction<IComponentNameStyleProps, IComponentNameStyles>;

}
```

3. **IComponentNameStyleProps** - The props needed to construct styles. This represents the simplified set of immutable things which control the class names. Note that things which were optional may be set to be required here, to simplify the style definitions:

```tsx
export interface IComponentNameStyleProps {
  theme: ITheme;
  disabled: boolean;
  checked: boolean;
}
```

4. **IComponentNameStyles** The styles which apply to each area of the component. Each area should be listed here required, as an `IStyle`, with `root` always representing the root element of the component:

```tsx
export interface IComponentNameStyles {
  root: IStyle;
  child1: IStyle;
  // etc.
}
```

In the style interface, always refer to the root element using the name `root`, for predictability in styling.

#### ComponentName.base.tsx contents

1. The component shoud be named `{ComponentName}Base`.
2. It should be decorated with the `customizable` decorator using `{ComponentName}Base` as the target name.
3. It should use the `classNameFunction` helper to create a className generation function.

Example:

```tsx
import { IComponentNameProps } from ='./ComponentName.props';

const getClassNames = classNameFunction<IComponentNameStyleProps, IComponentNameStyles>();

export class ComponentName extends React.Component<...> {
  public render() {
    const { getStyles, theme } = this.props;
    const classNames = getClassNames(getStyles, { theme: theme! });

    return (
      <div className={ classNames.root }>Hello</div>;
    );
  }
}
```

#### ComponentName.styles.ts

The styles file should export the `getStyles` function which takes in `IComponentNameStyleProps` and returns `IComponentNameStyles` (the default styling for the component.)

Note that the root element for styles should al

```tsx
export function getStyles(props: IComponentNameStyleProps): IComponentNameStyles {
  return {
    root: {},
    child: {},
    etc.
  };
}
```

#### Component.tsx

Tying the component to the style is made easy using the `styled` HOC wrapper, whch as input takes the base component and an object of 1 or more style function props.

```tsx
import { styled } from 'office-ui-fabric-react/lb/Styling';
import { CompoenentNameBase } from './ComponentName.base';
import { getStyles } from './ComponentName.styles';

// Create a Breadcrumb variant which uses these default styles.
export const ComponentName = styled(
  ComponentNameBase,
  {
    getStyles
  }
);
```

#### Support for customizable sub-component styling

The component may also intend to provide customized styling for nested components. For example, a `Breadcrumb` component may want to render `Crumb` components, which may take in their own styles.

The recommended approach is to avoid exposing the individual `Crumb` props at the `Breadcrumb` layer. Instead, expose a way to provide an alternative component for the `Crumb` using the `as` prop convention. This lets the caller create their own `Crumb` variant and use it within the `Breadcrumb`. It also opens up other scenarios such as providing additional default props on the sub-component, or replacing it completely.

In the `Breadcrumb` case, we would expose an optional `crumbAs` prop. This would allow the consumer to make a `Breadcrumb` variant which renders red `Crumb` components:

```
const RedCrumb = styled(Crumb, props => ({ root: background: 'red' }));

render() {
  return <Breadcrumb crumbAs={ RedCrumb } ... />
```

Additionally this "breadcrumb with red crumbs" scenario could be abstracted as its own component, by passing in the 3rd optional param to `styled`, which lets the caller provide new default prop values:

```
const RedBreadcrumb = styled(Breadcrumb, undefined, { crumbAs: RedCrumb });

render() {
  return <RedBreadcrumb ... />
}
```

#### ComponentName.test.tsx

The test file should include:

1. A snapshot test locking the component DOM structure down for the important states.
2. Tests which simulate clicking on things, changing the state of the compoent, and validating things still work.

#### Moving styles from scss to ts

Basic conversion just means copying styles from scss into ts, making prop names camelCased instead of kebab-cased, and stringifying everything except for pixel values.

In addition, all static classnames embedded within the tsx file inside of the `css` helper function calls can now move into the styles file.

Styles in scss:
```css
.list {
  white-space: nowrap;
  padding: 0;
  margin: 0;
  display: flex;
  align-items: stretch;
}
```

Converted to ts:
```ts
list: [
  'ms-Breadcrumb-list',
  {
    whiteSpace: 'nowrap',
    padding: 0,
    margin: 0,
    display: 'flex',
    alignItems: 'stretch'
  }
],
```

Some scss special cases:

##### mixins and includes

Sass mixins are simply an informal way of using functions. Translating them into actual javascript, where you can reuse and import/export them, is really easy.

If you find some fabric-core mixins are missing, consider adding them to the `@uifabric/styling` package if they are highly reusable. However keep in mind that the PLT1 bundle size WILL be affected, so do this sparingly only for very common things.

#### font-size-x variables

Use typesafe enums instead of the sass variables:

```ts
import { FontSizes } from 'office-ui-fabric-react/lib/Styling';

fontSize: FontSizes.small
```

##### Focus rectangles

The `styling` package has a helper to provide consistent focus rectangles.


#### Footnotes: Motivations for moving away from SCSS

SCSS a build time process of expanding a high level css-like language into raw css. Our pipeline to load the raw css goes through a javascript conversion process and gets loaded on the page via a javascript library called `load-themed-styles`. Effectively, we have a complex build process which takes rules, converts them into JavaScript, and loads them dynamically.

This process is complicated and adds a number of limitations.

#### We can't register classes dynamically

Scenarios like "make this area of the screen use a different theme" become really complicated if build time is the only time for evaluations.

#### Bundle size and css loading heft with scss

If a button has 20 different possible states, using scss you must load the css for all 20 of the states pre-emptively, so you end up loading way more rules than you will ever actually use. There is no "plt1 styles vs delay loaded styles". The best you can do is to partition your css to specific modules, and delay load the modules. But in this model, you will still preempt loading a lot of rules that aren't used.

Sass also encourages "mixins" as a way to have one definition of styles that can be used in multiple places. This completely fights against bundle size, since mixins simply stamp duplicates copies of the same rules whereever they're used, resulting in bloated (but highly compressable) style definitions. The compression helps but all of this could be avoided by using a different approach to defining our styling.

#### Constant battle with specificity

Perhaps the most difficult thing to resolve is css specificity. Countless hacks have been implemented to "slightly tweak" styling of a thing in a particular context. If your rule is equally specific as an existing rule, you have a race condition; last one to register wins, resulting in hacks that only work sometimes. And even if your rule is more specific than an existing rule, there are no gates that can catch an existing rule being changed to be more specific later, resulting in breaking the workarounds.

We want a system which allows users to pass in their overrides, which can create new permutations of classes which are only 1 level of specificity deep, providing a consistent safe way to override the defaults.

### Testing

Our tests are built using [Jest](https://facebook.github.io/jest/). This allows us to run tests in a node environment, and simulates the browser using jsdom.

For snapshot testing, we use `react-test-renderer` and Jest apis.

For creating React functional tests, we use [Enzyme](http://airbnb.io/enzyme/) to automate rendering. This gives us helpers for mounting a component, accessing elements rendered by it, and simulating clicks and keypresses.

Visual regression testing uses [Storybook](https://storybook.js.org/basics/introduction/) to document various UI states of components.

#### Running tests

In command prompt navigate to the appropriate package, for example `packages/office-ui-fabric-react`

To just validate everything works, run `npm run build`, which will build the project including running tslint and jest for tests.

If you *only* want to run jest, you can also run only the `jest` task by running `npm run build jest`.

#### Running tests in watch mode

When you are developing tests, use the watch mode to run the tests as you write them!

1. Go to the package folder where you want to run the tests.
2. Type `npm run start-test`.
3. Edit and saving tests should now cause the console to re-run the tests you have added/modified.

#### Debugging

To debug tests, you can use Visual Studio Code. Inside of a `*.test.ts` file, add a `debugger` statement where you want to break, and hit F5 to start debugging.

Note: Because of limitations with the current Node LTS version, breakpoints in VSCode will not hit until you're actively in the debugger using `debugger` statement. The latest Node version however has fixes that will enable breakpoints to resolve, so this workaround is temporary.

#### Writing tests

##### Simple unit testing

Tests in Jest are written similar to mocha tests, though Jest includes a number of assertions that work similar to chai. A basic test example:

```ts
describe('thing', () => {
  it('does something', () => {
    expect(thing.something()).toEqual(aValue);
  });
});
```

Note that you do not need to import the assertions or the Jest APIs; they should be available automatically through the included typings.

##### Snapshot testing

Jest enables you to create snapshot tests. Snapshots simply compare a JSON object with an expected output. The assertion `toMatchSnapshot` api will abstract loading a .snap file for the test to compare, or will create one if none exists.

```typescript
import * as React from 'react';
import { CommandBar } from './CommandBar';
import * as renderer from 'react-test-renderer';

describe('CommandBar', () => {
  it('renders commands correctly', () => {
    expect(renderer.create(
      <CommandBar
        items={ [
          { key: '1', name: 'name 1' },
          { key: '2', name: 'name 2' }
        ] }
      />
    ).toJSON()).toMatchSnapshot();
  });
});
```

If you ever break a snapshot, you can update all baselines either manually, or using the `npm run update-snapshots` command within a given package folder. Currently the `office-ui-fabric-react` and `experiments` packages both have snapshot testing enabled.

##### Functional testing

In cases where you need to automate a component and validate it performs correctly, you can use [Enzyme](http://airbnb.io/enzyme/) apis to mount components, evaluate dom structure, and simulate events.

```jsx
  it('opens a menu with IContextualMenuItem.subMenuProps.items property', () => {
    const commandBar = mount<CommandBar>(
      <CommandBar
        items={ [
          {
            name: 'TestText 1',
            key: 'TestKey1',
            className: 'MenuItem',
            subMenuProps: {
              items: [
                {
                  name: 'SubmenuText 1',
                  key: 'SubmenuKey1',
                  className: 'SubMenuClass'
                }
              ]
            }
          },
        ] }
      />
    );

    const menuItem = commandBar.find('.MenuItem button');
    expect(menuItem.length).toEqual(1);
    menuItem.simulate('click');
    expect(document.querySelector('.SubMenuClass')).toBeDefined();
  });
```

##### Visual regression testing

[Storybook](https://storybook.js.org/basics/introduction/) is a dev environment for UI components. We write 'stories' to capture different states of components. With every pull request, the stories are rendered by Screener to check for any visual changes. Screener posts a status to Github PRs where you can view the visual test report. If changes are found, the status will fail on Github until the regressions are fixed or an admin approves the changes.

Stories are found at `./apps/vr-tests/src/stories`. Most stories are written with a `FabricDecorator` that wraps the components with consistent padding. [Screener](https://github.com/screener-io/screener-storybook) steps are added to crop to a specific CSS class (most stories should crop to the `.testWrapper` class of the `FabricDecorator`) and to simulate different events, such as `hover` and `click`.

```jsx
import * as React from 'react';
import Screener, { Steps } from 'screener-storybook/src/screener';
import { storiesOf } from '@storybook/react';
import { FabricDecorator } from '../utilities';
import { Link, ILinkProps } from 'office-ui-fabric-react';

storiesOf('Link', module)
  .addDecorator(FabricDecorator)
  .addDecorator(story => (
    <Screener
      steps={ new Steps()
        .snapshot('default', { cropTo: '.testWrapper' })
        .hover('.ms-Link')
        .snapshot('hover', { cropTo: '.testWrapper' })
        .click('.ms-Link')
        .hover('.ms-Link') // Always add a 'hover' step after 'click'
        .snapshot('click', { cropTo: '.testWrapper' })
        .end() // Every set of Screener steps should finish with 'end()'
      }
    >
      { story() }
    </Screener>
  ))
  .add('Root', () => (<Link href='#'>I'm a link</Link>))
  .add('Disabled', () => (<Link href='#' disabled>I'm a disabled link</Link>))
  .add('No Href', () => (<Link>I'm rendered as a button because I have no href</Link>));
```

Certain components may be written with a custom decorator/wrapper, and you may crop to a different CSS class or omit the `cropTo` option altogether. Components that render outside its container, require specific styles on its parent, or render on a different layer, such as Callout, are cases where you would customize the decorators.

```jsx
storiesOf('Slider', module)
  .addDecorator(story => (
    // Vertical slider requires its parent to have a height specified
    <div style={ { width: '300px', height: '200px', display: 'flex' } }>
      { story() } // Render story (component) inside this container
    </div>
  ))
  .addDecorator(FabricDecorator)
  .addDecorator(story => (
    <Screener
      steps={ new Screener.Steps()
        .snapshot('default', { cropTo: '.testWrapper' })
        .hover('.ms-Slider-line')
        .snapshot('hover', { cropTo: '.testWrapper' })
        .end()
      }
    >
      { story() }
    </Screener>
  )).add('Vertical', () => (
    <Slider
      label='Basic example:'
      min={ 1 }
      max={ 3 }
      step={ 1 }
      defaultValue={ 2 }
      showValue={ true }
      vertical={ true }
    />
  ));
```
#### Test Utilities & Helpers
##### shallowUntilTarget Function

Enzyme has a method called Shallow Rendering that allows you to constrain yourself to testing a component as a unit, and to ensure that your tests aren't indirectly asserting on behavior of child components. If you would like to know more about Shallow Rendering in general then you can view the main Enzyme documentation [here](http://airbnb.io/enzyme/docs/api/shallow.html).

The `shallowUntilTarget()` function is a work around due to a conflict with decorated components, and considering at the time of writing this we are planning to use the **@customizable** decorater on all Fabric components it is likely that the built into Enzyme `shallow()` function will not yield the correct results because it's being applied to the customized component.

```jsx
it('renders the result of onRenderData', () => {
    const initialData = { content: 5 };
    const renderedDataId = 'onRenderDataId';
    const onRenderData = (data: any) => <div id={ renderedDataId }> Rendered data: { data.content }</div >;

    const wrapper = shallow<IResizeGroupProps, IResizeGroupState>(
      <ResizeGroup
        data={ initialData }
        onReduceData={ onReduceScalingData }
        onRenderData={ onRenderData }
      />
    );

    expect(wrapper.containsMatchingElement(onRenderData(initialData))).toEqual(true);
  });
```
For example above you expect shallow to return a ShallowWrapper of **ResizeGroup** but actually it will return a ShallowWrapper of **ComponentWithInjectedProps** - the customized component returned from the **@customizable** decorator function. **ResizeGroup** is a child of the customized component.  `shallowUntilTarget()` will allow you so specify which component you want to target for your test using a string - in this case we want **'ResizeGroupBase'** because the same effect happens to components using the `styled` function.

We just need to import the function from the common folder.
```jsx
import { shallowUntilTarget } from '../../common/shallowUntilTarget';
```
Usage is exactly the same as shallow with an added argument containing a string name of target component.
```jsx
it('renders the result of onRenderData', () => {
    const initialData = { content: 5 };
    const renderedDataId = 'onRenderDataId';
    const onRenderData = (data: any) => <div id={ renderedDataId }> Rendered data: { data.content }</div >;

    const wrapper = shallowUntilTarget<IResizeGroupProps, IResizeGroupState>(
      <ResizeGroup
        data={ initialData }
        onReduceData={ onReduceScalingData }
        onRenderData={ onRenderData }
      />
      , 'ResizeGroupBase');

    expect(wrapper.containsMatchingElement(onRenderData(initialData))).toEqual(true);
  });
```

#### FAQ

*Q. Browser methods aren't working.*

A. Using browser methods like getBoundingClientRect won't work when using enzyme to render a document fragment. It's possible to mock this method out if you need, see the `FocusZone` unit tests as an example.

*Q. My event isn't being triggered.*

A. Make sure to use Enzyme `simulate` api to simulate React events. For example: `menuItem.simulate('click');`
