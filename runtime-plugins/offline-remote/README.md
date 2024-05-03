 # Introduction

This guide will introduce you to the concept of offline remotes in a federated application using rspack and webpack. Offline remotes allow a federated application to continue functioning even when one or more of its remote applications are offline. This is achieved by implementing a custom runtime plugin that provides a fallback module when a remote application fails to load.

In this guide, we will use two federated applications, `app1` and `app2`, to demonstrate the implementation of offline remotes. Each application will expose a `Button` component, and `app1` will also consume the `Button` component from `app2`.

# Key Concepts

* Offline remotes: A technique for handling remote application failures in a federated application.
* Custom runtime plugin: A plugin that extends the functionality of the federation runtime, allowing for custom behavior when loading remote applications.
* Fallback module: A module that is used when a remote application fails to load, allowing the federated application to continue functioning.

# Implementation Guide

## 1. Set up the project structure

Create two directories, `app1` and `app2`, and initialize each directory as a new npm project:
```
mkdir app1 app2
cd app1
npm init -y
cd ../app2
npm init -y
```

## 2. Install dependencies

Install the required dependencies for both `app1` and `app2`:
```
npm install @babel/core @babel/preset-react babel-loader html-webpack-plugin serve @rspack/core @rspack/cli @rspack/dev-server webpack webpack-cli webpack-dev-server @module-federation/runtime @module-federation/enhanced
```

## 3. Configure rspack

Create a `rspack.config.js` file in the `app1` directory with the following contents:
```javascript
const {
  HtmlRspackPlugin,
  container: { ModuleFederationPlugin },
} = require('@rspack/core');
const path = require('path');

// adds all your dependencies as shared modules
// version is inferred from package.json in the dependencies
// requiredVersion is used from your package.json
// dependencies will automatically use the highest available package
// in the federated app, based on version requirement in package.json
// multiple different versions might coexist in the federated app
// Note that this will not affect nested paths like "lodash/pluck"
// Note that this will disable some optimization on these packages
// with might lead the bundle size problems
const deps = require('./package.json').dependencies;

module.exports = {
  entry: './src/index',
  mode: 'development',
  devServer: {
    static: {
      directory: path.join(__dirname, 'dist'),
    },
    hot: true,
    port: 3001,
    liveReload: true,
  },
  target: 'web',
  output: {
    publicPath: 'auto',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        include: path.resolve(__dirname, 'src'),
        use: {
          loader: 'builtin:swc-loader',
          options: {
            jsc: {
              parser: {
                syntax: 'ecmascript',
                jsx: true,
              },
              transform: {
                react: {
                  runtime: 'automatic',
                },
              },
            },
          },
        },
      },
      {
        test: /\.ts$/,
        use: {
          loader: 'builtin:swc-loader',
          options: {
            jsc: {
              parser: {
                syntax: 'typescript',
                jsx: true,
              },
              transform: {
                react: {
                  runtime: 'automatic',
                },
              },
            },
          },
        },
      },
    ],
  },
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
};
```

## 4. Configure webpack

Create a `webpack.config.js` file in the `app1` directory with the following contents:
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { ModuleFederationPlugin } = require('@module-federation/enhanced');
const path = require('path');

// adds all your dependencies as shared modules
// version is inferred from package.json in the dependencies
// requiredVersion is used from your package.json
// dependencies will automatically use the highest available package
// in the federated app, based on version requirement in package.json
// multiple different versions might coexist in the federated app
// Note that this will not affect nested paths like "lodash/pluck"
// Note that this will disable some optimization on these packages
// with might lead the bundle size problems
const deps = require('./package.json').dependencies;

module.exports = {
  entry: './src/index',
  cache: false,
  mode: 'development',
  devServer: {
    static: {
      directory: path.join(__dirname, 'dist'),
    },
    port: 3001,
  },
  target: 'web',
  output: {
    publicPath: 'auto',
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: ['@babel/preset-react'],
        },
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js',
      },
      runtimePlugins: [require.resolve('./offline-remote.js')],
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
      },
    }),
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
};
```

## 5. Implement the custom runtime plugin

Create an `offline-remote.ts` file in the `app1` directory with the following contents:
```javascript
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

## 6. Implement the `Button` component

Create a `src/Button.js` file in the `app2` directory with the following contents:
```javascript
import React from 'react';
import * as lodash from 'lodash';

const style = {
  background: '#00c',
  color: '#fff',
  padding: 12,
};

const Button = () => (
  <button
    onClick={() => {
      window.localStorage.setItem('button', 'green');
      window.location.reload();
    }}
    style={style}
  >
    App 2 Button - CLICK ME
  </button>
);

export default Button;
```

## 7. Implement the `App` component

Create a `src/App.js` file in the `app1` directory with the following contents:
```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import lodash from 'lodash';

import LocalButton from './Button';
import RemoteButton from 'app2/Button';

// A function to generate a color from a string
const getColorFromString = str => {
  // Prime numbers used for generating a hash
  let primes = [1, 2, 3, 5, 7, 11, 13, 17, 19, 23];
  let hash = 0;

  // Generate a hash from the string
  for (let i = 0; i < str.length; i++) {
    hash += str.charCodeAt(i) * primes[i % primes.length];
  }

  // Convert the hash to a color
  let color = '#';
  for (let i = 0; i < 3; i++) {
    const value = (hash >> (i * 8)) & 0xff;
    color += ('00' + value.toString(16)).substr(-2);
  }

  return color;
};

// The main App component
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

## 8. Run the applications

Start both `app1` and `app2` using the following commands:
```
cd app1
npm start
cd ../app2
npm start
```

# Summary

In this guide, we have demonstrated how to implement offline remotes in a federated application using rspack and webpack. By using a custom runtime plugin, we were able to provide a fallback module when a remote application fails to load, allowing the federated application to continue functioning. This technique can be useful in scenarios where remote applications may be offline or unavailable, and can help improve the resilience and reliability of a federated application.