React Router Redux Middleware
=============================

Middleware for React + Router + Redux with Server Side Rendering.

# Attention, this is an experimental project. It requires basic knowledge of involved frameworks.

Usage:

```bash
npm install react-router-redux-middleware --save-dev
```

We assume that you should already have `express`, `webpack` and `webpack-dev-server`. You will need them.

Server rendering basically takes your application, applies same Babel transformations as Webpack do, then renders it
to a string. Because of that you will also need a `babel-cli` since it's the easiest way to run the server with Babel:

```bash
npm install babel-cli --save-dev
```

We assume that you either have `.babelrc` file or `babel` section in your `package.json`.

Alter your `package.json` and add to `scripts` section:

```json
{
  "scripts": {
    "start": "NODE_ENV=development babel-node ./index.js",
    "server": "NODE_ENV=production babel-node ./index.js"
  }
}
```

Now it's time to configure your `router.js`:

```js
import React from "react";
import {Router, Route} from "react-router";
import NotFound from './NotFound';

function def(promise) {
    return promise.then(cmp => cmp.default);
}

export default function(history) {

    return <Router history={history}>
        <Route path='/' getComponent={() => def(import('./App'))}/>
        <Route path='*' component={NotFound}/>
    </Router>;

}
```

And the way how you render your app, this hackery is especially required if you use async routes:

```js
import React from "react";
import {render} from "react-dom";
import {Provider} from "react-redux";
import {browserHistory, match, Router} from "react-router";

import createRouter from "./router";
import createStore from "./reduxStore";

const mountNode = document.getElementById('app'); // !!!!! PLEASE NOTE ID OF MOUNT NODE !!!!!
const store = createStore(window.__PRELOADED_STATE__); // !!!!! PLEASE NOTE THE NAME OF VARIABLE !!!!!

function renderRoutes(routes, store, mountNode) {

    match({history: browserHistory, routes}, (error, redirectLocation, renderProps) => {

        render((
            <Provider store={store}>
                <Router {...renderProps} />
            </Provider>
        ), mountNode);

    });

}

renderRoutes(createRouter(browserHistory), store, mountNode);
```

Here is the sample `index.html` that should be a part of webpack build, e.g. emitted to you output path. It could be a
real file or generated by `HtmlWebpackPlugin`, but it has to be known by Webpack.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>App</title>
<body>
<div id="app"></div><!-- !!!!! MUST EXACTLY MATCH THE SERVER CONFIG !!!!! -->
</body>
</html>
```

Now let's set up the `server.js`, you can add more customizations if needed:

```js
import path from "path";
import Express from "express";
import webpack from "webpack";
import Server from "webpack-dev-server";
import createRouter from "./src/router"; // same file as in client side
import createStore from "./src/reduxStore"; // same file as in client side
import config from "./webpack.config";
import {createExpressMiddleware, createWebpackMiddleware, skipRequireExtensions} from "react-router-redux-middleware";

skipRequireExtensions(); // this may be omitted but then you need to manually teach Node to ignore non-js files

const port = process.env.PORT || 3000;

const options = {
    createRouter: createRouter,
    createStore: ({req, res}) => (createStore({
        foo: Date.now() // pre-populate something right here
    })),
    initialStateKey: '__PRELOADED_STATE__', // !!!!! MUST MATCH THE CLIENT
    template: ({template, html}) => (template.replace(
        // !!!!! MUST MATCH THE INDEX.HTML
        `<div id="app"></div>`,
        `<div id="app">${html}</div>`
    )),
    templatePath: path.join(config.output.path, 'index.html'),
    outputPath: config.output.path
};

if (process.env.NODE_ENV !== 'production') {

    const compiler = webpack(config);
    const middleware = createWebpackMiddleware(compiler, config);

    config.devServer.setup = function(app) {
        app.use(middleware(options));
    };

    new Server(compiler, config.devServer).listen(port, '0.0.0.0', listen);

} else {

    const app = Express();

    app.use(createExpressMiddleware(options));
    app.use(Express.static(config.output.path));

    app.listen(port, listen);

}

function listen(err) {
    if (err) throw err;
    console.log('Listening %s', port);
}
```