
 # Introduction

This guide will introduce you to the concept of offline remotes in a federated application. Offline remotes allow a federated application to continue functioning even when a remote is offline. This is achieved by using a custom runtime plugin that provides a fallback module when a remote is offline.

In this guide, we will use a simple example of two federated applications, `app1` and `app2`, where `app1` loads a button component from `app2`. We will implement a custom runtime plugin that provides a fallback module when `app2` is offline.

# Key Concepts

* **Federated Application**: A federated application is a single application that is composed of multiple smaller applications, each with its own set of dependencies and functionality.
* **Remote**: A remote is a smaller application that is loaded by a federated application. A remote can expose its functionality to the federated application through a shared module.
* **Custom Runtime Plugin**: A custom runtime plugin is a plugin that can be used to customize the behavior of a federated application. In this guide, we will use a custom runtime plugin to provide a fallback module when a remote is offline.
* **Fallback Module**: A fallback module is a module that is used when a remote is offline. The fallback module can provide a default implementation of the functionality that is exposed by the remote.

# Implementation Guide

To implement offline remotes in a federated application, we will use a custom runtime plugin that provides a fallback module when a remote is offline. The following steps will guide you through the implementation:

1. Install the required dependencies:
```
npm install @module-federation/runtime @module-federation/enhanced
```
2. Create a custom runtime plugin in `app1/offline-remote.ts`:
```typescript
import { FederationRuntimePlugin } from '@module-federation/runtime/types';

export default function ():FederationRuntimePlugin {

  const getErrorMessage = (id, error) => `remote ${id} is offline due to error: ${error}`;

  const getModule = (pg, from) => {
    if (from === 'build') {
      return () => ({
        __esModule: true,
        default: pg,
      });
    } else {
      return {
        default: pg,
      };
    }
  };

  return {
    name: 'offline-remote-plugin',
    errorLoadRemote({id, error, from, origin}) {
      console.error(id, 'offline');
      const pg = function () {
        console.error(id, 'offline', error);
        return getErrorMessage(id, error);
      };

      return getModule(pg, from);
    },
  };
}
```
3. Update the `app1/rspack.config.js` and `app1/webpack.config.js` files to include the custom runtime plugin:
```javascript
plugins: [
  new ModuleFederationPlugin({
    name: 'app1',
    filename: 'remoteEntry.js',
    remotes: {
      app2: 'app2@http://localhost:3002/remoteEntry.js',
    },
    runtimePlugins: [require.resolve('./offline-remote.ts')],
    exposes: {
      './Button': './src/Button',
    },
    shared: {
      ...deps,
      react: {
        singleton: true,
      },
      'react-dom': {
        singleton: true,
      },
      lodash: {},
    },
  }),
  new HtmlRspackPlugin({
    template: './public/index.html',
  }),
],
```
4. Update the `app2/webpack.config.js` file to expose the button component:
```javascript
plugins: [
  new ModuleFederationPlugin({
    name: 'app2',
    filename: 'remoteEntry.js',
    exposes: {
      './Button': './src/Button',
    },
    shared: {
      ...deps,
      react: {
        singleton: true,
      },
      'react-dom': {
        singleton: true,
      },
      lodash: {},
    },
  }),
  new HtmlWebpackPlugin({
    template: './public/index.html',
  }),
],
```
5. Update the `app1/src/App.js` file to load the button component from `app2`:
```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import lodash from 'lodash';

import LocalButton from './Button';
import RemoteButton from 'app2/Button';

// ...

const App = () => (
  <div>
    <h1>Offline Remote</h1>
    <h2>Remotes currently in use</h2>
    {/* Display the names of the remotes loaded by the CustomPlugin */}
    {__FEDERATION__.__INSTANCES__.map(inst => (
      <span
        style={{
          padding: 10,
          color: '#fff',
          background: getColorFromString(inst.name.split().reverse().join('')),
        }}
        key={inst.name}
      >
        {inst.name}
      </span>
    ))}
    <p>
      Click The second button. This will cause the <i>pick-remote.ts</i> to load remoteEntry urls
      from a mock api call.
    </p>
    {/* LocalButton is a button component from the local app */}
    <LocalButton />
    {/* RemoteButton is a button component loaded from a remote app */}
    <React.Suspense fallback="Loading Button">
      <RemoteButton />
    </React.Suspense>
    {/* The Reset button clears the 'button' item from localStorage */}
    <button
      onClick={() => {
        localStorage.clear('button');
        window.location.reload();
      }}
    >
      Reset{' '}
    </button>
  </div>
);

export default App;
```
6. Start the `app1` and `app2` applications:
```
npm start
```
7. Open the `app1` application in a web browser and click the "App 2 Button - CLICK ME" button. This will cause the `app2` remote to go offline.
8. Observe that the `app1` application continues to function even though the `app2` remote is offline. The fallback module provided by the custom runtime plugin is used instead of the remote module.

# Summary

In this guide, we have learned how to implement offline remotes in a federated application using a custom runtime plugin. We have seen how to create a custom runtime plugin that provides a fallback module when a remote is offline. We have also seen how to update the `rspack.config.js` and `webpack.config.js` files to include the custom runtime plugin and how to load a remote module from a federated application.

Offline remotes are a powerful feature of federated applications that can improve the reliability and availability of an application. By using a custom runtime plugin, we can provide a fallback module when a remote is offline, ensuring that the application continues to function even when a remote is unavailable.