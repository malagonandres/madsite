# Angular website with SSR from firebase #

## Create the app ##

Build a new app. (I prefer to setup with some flags)

```
#!bash

ng new <YOUR_PROJECT_NAME> -s -S --routing --style=styl
```

Get inside de folder that has been created


```
#!bash
cd <YOUR_PROJECT_NAME>

```

## Add Universal SSR to the app ##
We are going to follow the steps presented in [angularfire](https://github.com/angular/angularfire) with some updates


```
#!bash

ng generate universal --client-project <YOUR_PROJECT_NAME>

npm install --save-dev @nguniversal/express-engine @nguniversal/module-map-ngfactory-loader express webpack-cli ts-loader ws xhr2

```

Create a file called server.ts in the root of you project.


```
#!javascript

import 'zone.js/dist/zone-node';
import 'reflect-metadata';

import { enableProdMode } from '@angular/core';
import { ngExpressEngine } from '@nguniversal/express-engine';
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

import * as express from 'express';
import { join } from 'path';
import { readFileSync } from 'fs';

(global as any).WebSocket = require('ws');
(global as any).XMLHttpRequest = require('xhr2');

enableProdMode();

export const app = express();

const DIST_FOLDER = join(process.cwd(), 'dist');
const APP_NAME = 'YOUR_PROJECT_NAME'; // TODO: replace me!

const {
  AppServerModuleNgFactory,
  LAZY_MODULE_MAP
} = require(`./dist/${APP_NAME}-server/main`);

const template = readFileSync(
  join(DIST_FOLDER, APP_NAME, 'index.html')
).toString();

app.engine(
  'html',
  ngExpressEngine({
    bootstrap: AppServerModuleNgFactory,
    providers: [provideModuleMap(LAZY_MODULE_MAP)]
  })
);

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, APP_NAME));

app.get('*.*', express.static(join(DIST_FOLDER, APP_NAME)));
app.get('*', (req, res) => {
  res.render(join(DIST_FOLDER, APP_NAME, 'index.html'), { req });
});

if (!process.env.FUNCTION_NAME) {
  const PORT = process.env.PORT || 4000;
  app.listen(PORT, () => {
    console.log(`Node server listening on http://localhost:${PORT}`);
  });
}
```

Create a new file named webpack.server.config.js to bundle the express app from previous step.


```
#!javascript

const path = require('path');
const webpack = require('webpack');

const APP_NAME = 'YOUR_PROJECT_NAME'; // TODO: replace me!

module.exports = {
  entry: { server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  mode: 'development',
  target: 'node',
  externals: [/^firebase/, /^bufferutil/, /^utf-8-validate/],
  output: {
    path: path.join(__dirname, `dist/${APP_NAME}-webpack`),
    library: 'app',
    libraryTarget: 'umd',
    filename: '[name].js'
  },
  module: {
    rules: [{ test: /\.ts$/, loader: 'ts-loader' }]
  },
  plugins: [
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'),
      {}
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
};
```

Update your package.json with the following build scripts, replacing YOUR_PROJECT_NAME with the name of your project.


```
#!json

"scripts": {
  // ... omitted
  "build": "ng build --prod && npm run build:ssr",
  "build:ssr": "ng run YOUR_PROJECT_NAME:server && npm run webpack:ssr",
  "webpack:ssr": "webpack --config webpack.server.config.js",
  "serve:ssr": "node dist/YOUR_PROJECT_NAME-webpack/server.js"
},
```

Test your app locally by running npm run build && npm run serve:ssr.


```
#!bash

npm run build && npm run serve:ssr
```



## Setup firebase in the app ##

Then inside your project root, setup your Firebase CLI project:


```
#!bash

firebase init

```

Configure whichever features you'd want to manage but make sure to select at least functions and hosting. Choose Typescript for Cloud Functions and use the default public directory for Hosting.


```
#!bash

? Which Firebase CLI features do you want to set up for this folder? Press Space to select features, then Enter to confirm your choices. Functions: Configure and deploy Cloud Functions, Hosting: Configure and deploy Firebase Hosting sites

? Please select an option: Use an existing project
? Select a default Firebase project for this directory: <YOUR_PROJECT_NAME:server>
? What language would you like to use to write Cloud Functions? TypeScript
? Do you want to use TSLint to catch probable bugs and enforce style? Yes
? Do you want to install dependencies with npm now? Yes
? What do you want to use as your public directory? public
? Configure as a single-page app (rewrite all urls to /index.html)? Yes
```

After you're configured, you should now see a firebase.json file in your project root. Let's add the following rewrites directive to it:


```
#!json

"hosting": {
    "public": "public",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "function": "universal"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(eot|otf|ttf|ttc|woff|font.css)",
        "headers": [
          {
            "key": "Access-Control-Allow-Origin",
            "value": "*"
          }
        ]
      },
      {
        "source": "**/*.@(jpg|jpeg|gif|png)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=7200"
          }
        ]
      },
      {
        "source": "404.html",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=300"
          }
        ]
      }
    ]
  }
```

Let's go ahead and modify your package.json to build for Cloud Functions:


```
#!json

"scripts": {
  // ... omitted
  "build": "ng build --prod && npm run copy:hosting && npm run build:ssr && npm run build:functions",
  "copy:hosting": "cp -r ./dist/YOUR_PROJECT_NAME/* ./public && rm ./public/index.html",
  "build:functions": "npm run --prefix functions build"
},
```

Change the build script in your functions/package.json to the following:


```
#!json

"scripts": {
    // ... omitted
    "build": "rm -r ./dist && cp -r ../dist . && tsc",
}
```

Create an empty folder inside functions folder named dist


```
#!bash

mkdir -p functions/dist
```

Finally, add the following to your functions/src/index.ts:


```
#!javascript

export const universal = functions.https.onRequest((request, response) => {
  require(`${process.cwd()}/dist/YOUR_PROJECT_NAME-webpack/server`).app(request, response);
});
```

We you should now be able to run npm run build to build your project for Firebase Hosting and Cloud Functions.

To test, spin up the emulator with firebase serve. 

```
#!bash
run npm run build


firebase serve
```

Once you've confirmed it's working go ahead and firebase deploy.

```
#!bash
firebase deploy

```
