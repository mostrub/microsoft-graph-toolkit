# Utilities for SharePoint Framework Web Parts using Microsoft Graph Toolkit

[![npm](https://img.shields.io/npm/v/@microsoft/mgt-spfx-utils?style=for-the-badge)](https://www.npmjs.com/package/@microsoft/mgt-spfx-utils)

![SPFx 1.16.1](https://img.shields.io/badge/SPFx-1.16.1-green.svg?style=for-the-badge)

Helper functions to simplify lazy loading of Microsoft Graph Toolkit components when using disambiguated web components in SharePoint Framework web parts. For more information on disambiguation in MGT see https://github.com/microsoftgraph/microsoft-graph-toolkit/tree/main/packages/mgt-components#disambiguation

## Installation

To lazy load Microsoft Graph Toolkit components in a React web part add both the `@microsoft/mgt-spfx-utils` and `@microsoft/mgt-react` packages as dependencies to your SharePoint Framework project:

```bash
npm install @microsoft/mgt-spfx-utils @microsoft/mgt-react
```

or

```bash
yarn add @microsoft/mgt-spfx-utils @microsoft/mgt-react
```

## Usage

Disambiguation is intended to provide developers with a mechanism to use a specific version of MGT in their solution without encountering collisions with other solutions that may be using MGT. `mgt-spfx` will allow all SPFx solutions in a tenant to use a single shared version, either v2.x or v3.x, of MGT. Currently multiple versions of `mgt-spfx` cannot be used in the same tenant. This is a limitation of the SharePoint Framework.

By disambiguating tag names of Microsoft Graph Toolkit components, you can use your own version of MGT rather than using the centrally deployed `@microsoft/mgt-spfx` package. This allows you to avoid colliding with SharePoint Framework components built by other developers. When disambiguating tag names, MGT is included in the generated SPFx bundle, increasing its size. It is strongly recommended that you use a disambiguation value unique to your organization and solution to avoid collisions with other solutions, e.g. `contoso-hr-extensions`.

> **Important:** Since a given web component tag can only be registered once this approach **must** be used along with the `customElementHelper.withDisambiguation('foo')` as this allows developers to create disambiguated tag names.

### When using no framework web parts

When building SharePoint Framework web parts without a JavaScript framework the `@microsoft/mgt-components` library must be asynchronously loaded after configuring the disambiguation setting. No helper library is necessary as dynamic imports are sufficient. After the browser has loaded the `@microsoft/mgt-components` library any tags which had already been rendered to the DOM without an existing custom element registration will automatically have the loaded custom behavior attached to them.

Below is a minimal example web part that demonstrates how to use MGT with disambiguation in SharePoint Framework Web parts. A more complete example is available in the [No Framework Web Part Sample](https://github.com/microsoftgraph/microsoft-graph-toolkit/blob/main/samples/sp-mgt/src/webparts/helloWorld/HelloWorldWebPart.ts).

```ts
import { BaseClientSideWebPart } from '@microsoft/sp-webpart-base';
import { Providers } from '@microsoft/mgt-element';
import { SharePointProvider } from '@microsoft/mgt-sharepoint-provider';
import { customElementHelper } from '@microsoft/mgt-element/dist/es6/components/customElementHelper';

export default class MgtWebPart extends BaseClientSideWebPart<Record<string, unknown>> {
  protected onInit(): Promise<void> {
    if (!Providers.globalProvider) {
      Providers.globalProvider = new SharePointProvider(this.context);
    }
    // Use the solution name to ensure unique tag names
    customElementHelper.withDisambiguation('spfx-solution-name');
    return import('@microsoft/mgt-components').then(() => super.onInit());
  }

  public render(): void {

    this.domElement.innerHTML = `
    <section class="${styles.helloWorld} ${this.context.sdks.microsoftTeams ? styles.teams : ''}">
      <mgt-spfx-solution-name-login></mgt-spfx-solution-name-login>
    </section>`;
  }
}
```

### When using React to build web parts

When building SharePoint Framework web parts using React any component that imports from the `@microsoft/mgt-react` library must be asynchronously loaded after configuring the disambiguation setting. The `lazyLoadComponent` helper function exists to facilitate using `React.lazy` and `React.Suspense` to lazy load these components from the top level web part.

Below is a minimal example web part that demonstrates how to use MGT with disambiguation in React based SharePoint Framework Web parts. A complete example is available in the [React SharePoint Web Part Sample](https://github.com/microsoftgraph/microsoft-graph-toolkit/blob/main/samples/sp-webpart/src/webparts/mgtDemo/MgtDemoWebPart.ts).

```ts
// [...] trimmed for brevity
import { Providers } from '@microsoft/mgt-element/dist/es6/providers/Providers';
import { customElementHelper } from '@microsoft/mgt-element/dist/es6/components/customElementHelper';
import { SharePointProvider } from '@microsoft/mgt-sharepoint-provider/dist/es6/SharePointProvider';
import { lazyLoadComponent } from '@microsoft/mgt-spfx-utils';

// Async import of component that imports the React Components
const MgtDemo = React.lazy(() => import('./components/MgtDemo'));

export interface IMgtDemoWebPartProps {
  description: string;
}
// set the disambiguation before initializing any webpart
// Use the solution name to ensure unique tag names
customElementHelper.withDisambiguation('spfx-solution-name');

export default class MgtDemoWebPart extends BaseClientSideWebPart<IMgtDemoWebPartProps> {
  // set the global provider
  protected async onInit() {
    if (!Providers.globalProvider) {
      Providers.globalProvider = new SharePointProvider(this.context);
    }
  }

  public render(): void {
    const element = lazyLoadComponent(MgtDemo, { description: this.properties.description });

    ReactDom.render(element, this.domElement);
  }

  // [...] trimmed for brevity
}
```

The underlying components can then use MGT components from the `@microsoft/mgt-react` package as usual. Because of the earlier setup steps the the MGT React components will render html using the disambiguated tag names:

```tsx
import { Person } from '@microsoft/mgt-react';

// [...] trimmed for brevity

export default class MgtReact extends React.Component<IMgtReactProps, {}> {
  public render(): React.ReactElement<IMgtReactProps> {
    return (
      <div className={ styles.mgtReact }>
        <Person personQuery="me" />
      </div>
    );
  }
}
```