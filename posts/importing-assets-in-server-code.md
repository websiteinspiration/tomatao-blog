---
Title:  Importing Assets In Server Code
Author: Thomas `tomatao` Hudspith-Tatham
Tags:   universal, react, css
Date:   April 3, 2016
---

# Importing Assets In Server Code

## Server don't do dat!

So you're building a universal JS application and you're taking advantage of Webpack for it's ability to require CSS files directly into the component. You may even be requiring in a SVG, PNG or markdown file, ooh the list goes on!

```js
import './banner.scss'
import banner from './banner.jpg'
// be happy ^_^
```

The first time I got that working in my client app... I felt pretty good! So good, that when I elegantly placed it in my universal app, I was expecting unicorns!... But no. The server side render just stared back at me with a vacant expression (in the form of an error). 

NodeJS is designed for importing JS files, when it puts together your client app to make a HTML page for rendering, it simply doesn't understand. Poor NodeJS, it just wants to know...

> *"Excuse me programmer, what is .PNG?"*.

### Roll Up Node-Hook

One way to alleviate this pain of the server render is to place a filter around the module require system in NodeJS. Some colleagues and myself have released a module that will do this for you, [node-hook-filename](https://github.com/tomatau/node-hook-filename). It allows you to pass an array of string values that will be tested against each require and then *stub* the require out.

```js
import nhf from 'node-hook-filename'
nhf(['.scss'], (filename) => {
  // return something else
})
```

The above code will catch any require/import that matches `.scss` and invoke an optional callback function with the matching file-name. It also allows you to return a new value that will be used instead of the module - this is very helpful for testing as you can supply spy objects. 

### What about image files? We need URLs in server render!

Node-hook-filename isn't enough. We need to import multiple asset types: images need to expose a path; SVGs might want to expose their mark-up; style files might wish to expose class names al a CSS modules.

### And cache busting...

Another consideration here is cache busting of assets. A nice feature of Webpack is to add a random hash to each compile of the build output. To get this working, the server needs to know when the compile has completed so it can perform the server render with knowledge of the necessary assets.

## Isomorphic-Tools

There is a fantastic tool that can allow your app to meet all of these criteria called [webpack-isomorphic-tools](https://www.npmjs.com/package/webpack-isomorphic-tools). It plugs into Webpack to generate a JSON file that contains a map between each asset you require and the value you need; be it a path, mark-up or object.

```js
// wepack config
import isomorphicConfig from 'config/isomorphic.config'
import IsomorphicToolsPlugin from 'webpack-isomorphic-tools/plugin'

const webpackConfig = {
  // ...
  plugins: [
    // ...
    new IsomorphicToolsPlugin(isomorphicConfig)
  ]
}
```

Add the plugin to your Webpack config. Notice the isomorphic.config, this is where you define how to interpret assets.

```js
// asset file generated by webpack plugin
{
  "javascript": {
    "main": "/main.d301e1ff3d5a11c2098c.js"
  },
  "assets": {
    "./src/assets/avatar.jpeg": "data:image/jpeg;base64,/...",
    "./src/app/icons/status.svg": "<svg viewBox=\"0 0 56 52\" width=\"56\" height=\"52\"></svg>"
  }
}
```

The above JSON is the map between assets and values that your server render can use.

For the server render to make use of this object, webpack-isomorphic-tools also provides a function that you should wrap your server render in.

```js
import IsomorphicTools from 'webpack-isomorphic-tools'
import isomorphicConfig from 'config/isomorphic.config'

const isomorphicTools = new IsomorphicTools(isomorphicConfig)
isomorphicTools.server(ROOT_DIRECTORY, () => {
  // do server render here
  const { rootRouter, setRoutes } = require(`server/router`)
  setRoutes(isomorphicTools.assets())

  app.use(function *() {
    yield rootRouter.routes()
  })
})
```

The above function `isomorphicTools.server()`, takes two arguments, the second is a callback that is invoked once the Webpack build is finished and the said JSON file is generated! **Just make sure the server render happens inside the callback!**

It then also replaces all of your asset requires in the server render with those defined in the JSON file... MAGIC! You can even access the asset object for building the link/script tags in your html page.

### Svg markup? Sure!

So how do we get webpack-isomorphic-tools to understand different asset types? It's all in the config you supply to the plug-in and the tool. It has some presets for fonts and images -- and you can configure custom asset types. Here is an example for exporting SVG mark-up:

```js
// config/isomorphic.config.js
export default {
  // ... config
  assets: {
    // ... other assets
    asset_type: {
      extension: 'svg',
      parser(module) {
        if (module.source) {
          // regex to grab content
          const regex = /module\.exports = "((.|\n)*)"/;
          // test regex against module source
          const match = module.source.match(regex);
          // either return the content or nothing
          return (match ? match[1] : '').replace(/\\/g, '');
        }
      }
    }
  }
}
```

The above creates a custom asset type for SVG on the server render that builds the mark-up into a string, just the same way that SVG-inline-loader would for webpack.

### What about CSS modules? We need class names!

For CSS Modules, we want our import statements to get an object containing the class names needed for our server render. While this is possible with webpack-isomophic-tools... it's can get quite messy with parser and filters in the config. But... there is another solution!

## CSS modules require hook!

There is another module purely for CSS module imports, named, [css-modules-require-hook](https://github.com/css-modules/css-modules-require-hook)! Put it before your server side render and it will do exactly what it says on the tin!

There are a few gotchas, one is random hash class-names that are very useful to prevent global class collisions... both Webpack and the hook need to generate the exact same hashes. To make this easier, both Webpack and the hook can simply generate class names based on the file location instead.

Another gotcha, is preprocessors. The hook comes with support for postCSS built in, which is great, but preprocessors such as SCSS require a little more work.

Here's an example set-up for the modules hook.

```js
import cssModulesHook from 'css-modules-require-hook'
import sass from 'node-sass'
import loaderUtils from 'loader-utils'
import autoprefixer from 'autoprefixer'

cssModulesHook({
  extensions: [ '.scss', '.css' ],
  // setup postCSS
  prepend: [ autoprefixer({ browsers: [ 'last 2 versions' ] }) ],
  // build class-names based on location
  generateScopedName(exportedName, exportedPath) {
    const path = exportedPath
      .replace(`${ROOT}/`, '') // strip absolute path
      .replace(/^\//, '') // strip leading slash
      .replace(/\.s?css$/, '') // strip extension
      .replace(/\/|\./g, '-') // replace bars with hyphens
    return `${path}-${exportedName}`
  },
  preprocessCss(css, filename) {
    return sass.renderSync({ // convert SCSS to CSS
      includePaths: [ `${ROOT}/node_modules`, STYLES ],
      data: css,
      file: filename,
      importer(url) {
        return { file: loaderUtils.urlToRequest(url) }
      },
    }).css
  },
})
```

Put the above code in your server to get CSS modules working in your server render!

You will also need Webpack to be configured to generate similar CSS class names.

```js
// webpack config
const webpackConfig = {
  // config...
  loaders: [ {
      test: /\.s?css$/,
      loaders: [
        'style',
        'postcss',
        'css' +
          '?modules' +
          '&localIdentName=[path][name]-[local]',
        'sass' +
          '?outputStyle=expanded',
      ],
    } ],
}
```

And that's all there is too it! Now you can cover pretty much any type of asset require!

Stay tuned for more posts!

- tomatao


