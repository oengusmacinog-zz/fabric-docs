## Using Icons

By default, the Fabric icons are not added to your bundle, in order to save bytes for scenarios where you don't care about icons, or you only care about a subset.

To make them available, you may initialize them as such:

```tsx
import { initializeIcons } from 'office-ui-fabric-react/lib/Icons';

initializeIcons(/* optional base url */);
```

### Alternative CDN options

By default, the icon font will be pulled from the SharePoint CDN.

If you would like the icons to be served from your cdn, simply copy the files from the package's `fonts` folder to your cdn, and in `initializeIcons`, provide the base url to access those fonts (Note that it will require a trailing slash.)

### What does `initializeIcons` do?

It registers a map of icon names, which define how to render icons. Icons can be rendered either through JSX components, or as a font character. The icon code will register the font-face definition only when a given icon from a subset is referenced.

What we're trying to optimize here is download size. We define a map of icon codes which map to a font-face. When the `Icon` component renders the `Upload` icon, we determine if the font-face has yet been registered, an if not, we add it to the page, causing the subset containing the `Upload` icon to be downloaded.

The `@uifabric/icons` packages can resolve over 1000 different icons, and will download from the 10+ font partitions, minimizing download overhead. We also include the most common icons in the first partition, optimizing for the basic scenarios. If there are commonly used icons missing in there, please file an issue so that we can evaluate adjusting the primary partition.

### Disabling generated warnings

When icons are rendered using the `Icon` component, but have not yet been registered, you will see console errors indicating so. In most cases, this can be addressed by registering the icons. But there are 2 cases, where this isn't desirable:

**Test scenarios**

In test scenarios, you may want to simply disable the warnings and avoid registering icons. To do this:

```tsx
import { setIconOptions } from 'office-ui-fabric-react/lib/Styling';

// Suppress icon warnings.
setIconOptions({
  disableWarnings: true
});
```

**Library icon registration**

If your code is running in an environment where icons may have already been registered, you may need to disable the warnings. (By default, registering the same icon twice will ignore subsequent registrations.) To initialize icons and avoid duplication warnings, pass options into `initializeIcons`:

```tsx
import { initializeIcons } from '@uifabric/icons';

initializeIcons(undefined, { disableWarnings: true });
```


