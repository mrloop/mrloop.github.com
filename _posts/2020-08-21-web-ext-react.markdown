---
layout: post
title: "ReactJS WebExtensions"
comments: true
categories: [JavaScript, ReactJS]
---

I wanted to write a [WebExtension](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions) for Firefox and Chrome in [ReactJS](https://reactjs.org), with little configuration in the simplest possible manner, using [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html#create-react-app) and [web-ext](https://github.com/mozilla/web-ext), I couldn't find any guides or instructions online so this is the setup I used. [create-react-app](https://reactjs.org/docs/create-a-new-react-app.html#create-react-app) is the recommended tool for creating new single page applications in React.

Lets create an app, first make sure you've got [Node.js](https://nodejs.org/en/) installed then:

```sh
npx create-react-app web-ext-react-hello
cd web-ext-react-hello
npm start
```

We have a basic react app running. The next step is to bundle it as a web extension. For this we will use [`web-ext-react`](https://github.com/mrloop/web-ext-react), the library I extracted from [`race-ext-react`](https://github.com/mrloop/race-ext-react) to help bundle react apps as web extensions.

```sh
yarn add -D web-ext web-ext-react
```

A web extension can have multiple different javascripts, HTML and CSS for different parts. For example your web extension may have a sidebar or a popup, each with its own javascripts, HTML and CSS. As `create-react-app` is designed to output a single app and not multiple, we need to conditionally invoke different components of our single react app depending on the context, be it the sidebar, popup, content script or background script. In this case we'll add a browser action popup. The `App` component will be conditionally rendered if invoked from the browser action context.

### src/index.js

```js
if (document.isBrowserAction) {
  ReactDOM.render(
    <React.StrictMode>
      <App />
    </React.StrictMode>,
    document.getElementById("root")
  )
}
```

The extension needs a `manifest.json`, create `extension/manifest.json` and copy the logo to the extension directory `cp public/logo192.png extension`

### extension/manifest.json

```json
{

  "description": "Bundle ReactJS as web extension",
  "manifest_version": 2,
  "name": "web-ext-react-hello",
  "version": "1.0",
  "homepage_url": "https://github.com/mrloop/web-ext-react-hello",
  "icons": {
    "192": "logo192.png"
  },

  "browser_action": {
    "default_icon": "logo192.png",
    "default_title": "Hello WebExt",
    "default_popup": "popup.html"
  }
}
```

This manifest declares a browser action with the react logo. This will appear in the browser toolbar when the extension is run. Clicking the icon and you'll see the popup running the `App` component.

To start the extension scripts can be added to `package.json`

### package.json

```json
"scripts": {
  "start:firefox": "web-ext-react run | xargs -L1 web-ext run -u http://www.example.org/ -s",
  "start:chrome": "web-ext-react run | xargs -L1 web-ext run -u http://www.example.org/ -t chromium -s",
}
```

Tweak the styling, add `padding` and changing `min-height`.

### src/App.css

```css
.App-header {
  padding: 1em;
  background-color: #282c34;
  min-height: 200px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-size: calc(10px + 2vmin);
  color: white;
}
```

Now run `yarn start:firefox`

<picture>
  <source media="(min-width: 461px)" srcset="/images/web-ext-react-hello-740w-fs8.png" />
  <source media="(max-width: 460px)" srcset="/images/web-ext-react-hello-460w-fs8.png" />
  <img src="/images/web-ext-react-hello-375w-fs8.png" alt="screenshot" />
</picture>

We now have the default `create-react-app` running as a web extension! Try editing the app and live reload still works.

For complete source code please visit [https://github.com/mrloop/web-ext-react-hello](https://github.com/mrloop/web-ext-react-hello)

