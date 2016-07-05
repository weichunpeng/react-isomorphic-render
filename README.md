# react-isomorphic-render

[![npm version](https://img.shields.io/npm/v/react-isomorphic-render.svg?style=flat-square)](https://www.npmjs.com/package/react-isomorphic-render)
[![npm downloads](https://img.shields.io/npm/dm/react-isomorphic-render.svg?style=flat-square)](https://www.npmjs.com/package/react-isomorphic-render)

Is a module providing support for isomorphic (universal) rendering with React, React-router, Redux, Redux-router. Also allows for Webpack "hot reload". Allows for locale detection and therefore internationalization of the app. Provides isomorphic HTTP client for REST API. Also it handles Http Cookies mutation correctly (both on client and server). And when paired with Redux it knows how to preload web pages when rendering them on the server.

## Installation

```bash
$ npm install react-isomorphic-render --save
```

## Usage

See [webapp](https://github.com/halt-hammerzeit/webapp) and [webpack-react-redux-isomorphic-render-example](https://github.com/halt-hammerzeit/webpack-react-redux-isomorphic-render-example) as references.

The following usage example is for `React + React-router + Redux` setup.

Create your `react-isomorphic-render.js` set up file (it will be used both on client and server)

```javascript
export default
{
  // Redux reducer
  // (either an object or a function returning an object)
  reducer: require('./src/client/redux/reducer'),

  // React-router routes
  // (either a `<Route/>` element or a `function({ store })` returning a `<Route/>` element)
  routes: require('./src/client/routes'),
  
  // Wraps React page component with arbitrary elements (e.g. <Provider/>, etc; see an example below)
  wrapper: require('./src/client/wrapper')
}
```

Start webpage rendering server

```javascript
import webpage_server from 'react-isomorphic-render/page-server'
import settings from './react-isomorphic-render'

// Create webpage rendering server
const server = webpage_server
({
  // enable/disable development mode
  development: true,

  // Http host and port for executing all client-side ajax requests on server-side.
  // This is the host and port on which the web application is run.
  application:
  {
    host: '192.168.0.1',
    port: 80
  },

  // URLs of javascript and CSS files
  // which will be insterted into the <head/> element of the resulting Html webpage
  // (as <script src="..."/> and <link rel="style" href="..."/>)
  //
  assets:
  {
    javascript: '/assets/application.js',
    style: '/assets/application.css'
  }
},
settings)

// Start webpage rendering server
// (`server.listen(port, [host], [callback])`)
server.listen(3000, function(error)
{
  if (error)
  {
    throw error
  }

  console.log(`Webpage rendering server is listening at http://localhost:${port}`)
})
```

Create your client-side main file `application.js`

```javascript
import { render } from 'react-isomorphic-render/redux'
import settings from './react-isomorphic-render'

// Renders the page in web browser
render
({
  // enable/disable development mode
  development: true
},
settings)
```

An example of a `wrapper` component:

```javascript
// Can be also a "React pure component" (i.e. a function)
export default class Wrapper extends React.Component
{
  // Wraps React page component with arbitrary elements (e.g. Redux Provider)
  render()
  {
    const { store, children } = this.props

    return <Provider store={store}>{children}</Provider>
  }
}
```

And, finally, React pages would look like this (optionally use `@preload()` helper to load the neccessary data before the page is rendered)

```javascript
import { title }              from 'react-isomorphic-render'
import { preload }            from 'react-isomorphic-render/redux'
import { connect }            from 'react-redux'
import { bindActionCreators } from 'redux'

// fetches the list of users from the server
function fetch_users()
{
  return {
    promise: http => http.get('/api/users').then(ids => Promise.map(ids, id => http.get(`/api/users/${id}`))),
    events: ['GET_USERS_PENDING', 'GET_USERS_SUCCESS', 'GET_USERS_FAILURE']
  }
}

@preload(dispatch => dispatch(fetch_users))
@connect
(
  state    => ({ users: state.users.users }),
  dispatch => bindActionCreators({ fetch_users }, dispatch)
)
export default class Page extends Component
{
  static propTypes =
  {
    users       : PropTypes.array.isRequired,
    fetch_users : PropTypes.func.isRequired
  }

  render()
  {
    return (
      <div>
        <title("Users")/>
        <ul>{this.props.users.map(user => <li>{user.name}</li>)}</ul>
        <button onClick={this.props.fetch_users}>Refresh</button>
      </div>
    )
  }

  // Observing action result example (advanced).
  //
  // Suppose you make a `delete_users()` function 
  // analagous to the `fetch_users()` function.
  //
  // Then you can call it like this:
  //
  // <button onClick={::this.delete_users}>Delete all users</button>
  //
  // (async/await Babel syntax is used here; can be rewritten as usual Promises)
  //
  async delete_users()
  {
    try
    {
      const count = await this.props.delete_users()
      alert(`Deleted ${count} users`)
    }
    catch (error)
    {
      alert(error)
    }
  }
}
```

The final step is to set up the main web server (`192.168.0.1:80` in this example) to proxy all HTTP requests for webpages to the webpage rendering server you've set up.

An example of how HTTP request routing on your main web server can be set up:

 * all HTTP GET requests starting with `/assets` return static files from your `assets` folder
 * all HTTP requests starting with `/api` call your REST API methods
 * all the other HTTP GET requests are proxied to `http://localhost:3000` for webpage rendering

(proxying can be easily set up with [http-proxy](https://github.com/nodejitsu/node-http-proxy))

(see the aforementioned example projects for reference)

## HTTP response status code

To set custom HTTP response status code for a specific route set the `status` property of that `<Route/>`.

```javascript
export default (
  <Route path="/" component={Layout}>
    <IndexRoute component={Home}/>
    <Route path="blog"  component={Blog}/>
    <Route path="about" component={About}/>
    <Route path="*"     component={Page_not_found} status={404}/>
  </Route>
)
```

## Page preloading

As you have noticed in the (Redux) example above, `@preload()` helper is called to preload a web page before display. It is used to preload pages before rendering them (both on the server side and on the client side).

```javascript
@preload(function(dispatch, getState, location, params) { return Promise })
```

Note: `@preload()` decorator should be placed on top of all other decorators in order to work. The reason is that it adds a static method to your `Route`'s `component` and further decorators on top of it may not retain that static method.

On the client side, when a user navigates a link, first it changes the Url in the address bar, then it waits for the next page to preload, and when the page is fully loaded it displays the page to the user. If preloading a page can take some time one may want to add a "spinner" to inform the user that the navigation process needs some time. It can be done by adding a Redux reducer listening to these three Redux events:

```javascript
import { Preload_started, Preload_finished, Preload_failed } from 'react-isomorphic-render/redux'

export default function(state = {}, event = {})
{
  switch (event.type)
  {
    case Preload_started  : return { ...state, pending: true,  error: false }
    case Preload_finished : return { ...state, pending: false }
    case Preload_failed   : return { ...state, pending: false, error: event.error }
    default               : return state
  }
}
```

```javascript
import React       from 'react'
import { connect } from 'react-redux'

export default connect(model => ({ pending: model.preload.pending, error: model.preload.error }))
(function Spinner(props)
{
  return <div className={"preloading " + (props.pending ? "preloading-show" : "")}/>
})
```

```css
.preloading
{
  position: fixed;

  top    : 0;
  left   : 0;
  right  : 0;
  bottom : 0;

  z-index: 1;

  background-color: rgba(0, 0, 0, 0.2);

  display: none;
}

.preloading-show
{
  display: block;
  cursor: wait;
}
```

## Utilities

### Setting webpage title, description, <meta/> tags

Uses [react-helmet](https://github.com/nfl/react-helmet) under the hood.

```javascript
import { head, title, meta } from 'react-isomorphic-render'

const meta =
[
  // <meta charset="utf-8"/>
  { charset: 'utf-8' },

  // <meta name="..." content="..."/>
  { name: 'viewport', content: 'width=device-width, initial-scale=1.0, user-scalable=no' },

  // <meta property="..." content="..."/>
  { property: 'og:title',       content: 'International Bodybuilders Club' },
  { property: 'og:description', content: 'Do some push ups' },
  { property: 'og:locale',      content: 'ru-RU' }
]

// sets specific webpage <head/> tags
head('WebApp', meta)

// webpage title will be replaced with this one
title('New webpage title')

// will add additional <meta/> tags to the webpage <head/>
meta({ ... same `meta` as above ... })
```

### Locale detection

This library performs the following locale detection steps for each webpage rendering HTTP request:

 * Checks the `locale` query parameter (if it's an HTTP GET request)
 * Checks the `locale` cookie
 * Checks the `Accept-Language` HTTP header

(for more info see [`koa-locale`](https://www.npmjs.com/package/koa-locale))
 
The resulting locale is passed as `preferred_locale` parameter into `localize()` function of the webpage rendering server which then returns `{ locale, messages }`.

Later, on the client side, that `locale` returned from the `localize()` function on the server side is fed into `load_translation(locale)` function, and when translation is loaded the application is rendered with `locale` and `messages` properties passed to the `wrapper`.

### Determining current location

```javascript
@connect(model => ({ location: model.router.location }))
class Component extends React.Component
{
  render()
  {
    return <span>{this.props.location}</span>
  }
}
```

### Changing current location

```javascript
import { goto, redirect } from 'react-isomorphic-render'

// usage example
// (`goto` navigates to a URL while adding a new entry in browsing history,
//  `redirect` does the same without modifying browsing history)
// this.props.dispatch(goto('/items/1?color=red'))
```

## Miscellaneous react-isomorphic-render.js settings

```javascript
{
  // (optional)
  // handles errors occurring inside `@preload()` of pages on the client side
  // (can redirect to special error pages if needed)
  on_preload_error: (error, { url, redirect }) => redirect(`/error?url=${encode(url)}&error=${error.code}`)

  // (optional)
  // user can add his own middleware to the `middleware` list
  redux_middleware: middleware => middleware

  // (optional)
  // is called when Redux store has been created
  // (can be used for setting up Webpack Hot Module Replacement)
  on_store_created: ({ reload_reducer }) => {}
}
```

## Miscellaneous webpage rendering server options

```javascript
{
  ...

  // URLs of javascript and CSS files
  // which will be insterted into the <head/> element of the resulting Html webpage
  // (as <script src="..."/> and <link rel="style" href="..."/>)
  //
  // Also a website "favicon" URL, if any.
  //
  // Can be an `object` or a `function(url)`.
  //
  // `javascript` and `style` can be strings or objects.
  // If they are objects then also provide an `entry` parameter.
  // The objects may also contain `common` entry
  // which will also be included on the page.
  //
  assets: (url) =>
  {
    return {
      javascript: '/assets/main.js',

      // (optional)
      style: '/assets/main.css',

      // (optional)
      // URL of your "favicon".
      // If you're using Webpack then the URL is the result of a require() call.
      icon: '/assets/icon.png',

      // (only required when `javascript` and `style` are objects)
      entry: 'webpack entry key' // e.g. 'main'
    }
  },

  // (optional)
  // Is called when an error happens on the server side
  // (can redirect to special "500 Error" pages)
  on_error: (error, { url, redirect }) => redirect(`/error?url=${encode(url)}&error=${error.code}`)

  // (optional)
  // Returns React element an array of React elements
  // which will be inserted into server rendered webpage's <head/>
  // (in case of an array use `key`s to prevent React warning)
  head: () => React element or an array of React elements

  // (optional)
  // Allows for wrapping React page component with arbitrary elements
  // (or doing whatever else can be done with a React element).
  // Returns either a React element or an array of React elements
  // which will be inserted into server rendered webpage's <body/>
  body: react_page_element => react_page_element

  // (optional)
  // Returns React element or an array of React elements.
  // Allows adding arbitrary React elements to the start of the <body/>
  // (use `key`s to prevent React warning when returning an array of React elements)
  body_start: () => React element or an array of React elements

  // (optional)
  // Returns React element or an array of React elements.
  // Allows adding arbitrary React elements to the end of the <body/>
  // (use `key`s to prevent React warning when returning an array of React elements)
  body_end: () => React element or an array of React elements

  // (optional)
  // (is used only in development mode - removes Ctrl + R (F5) flicker)
  // This function returns CSS text which will be inserted 
  // into server rendered webpage's <head/> <style/> tag.
  // If you're using Webpack then the CSS text is the result of a require() call.
  style: () => require('../assets/style.scss').toString()

  // (optional)
  // Preloads data before performing page rendering.
  // If this function returns an object then this object
  // will be merged into Redux store.
  preload: async (http_client) => {}
  // (or same without `async`: (http_client) => Promise.resolve({})

  // (optional)
  // Based on the `preferred_locale`,
  // returns the suitable `locale` and `messages` for it.
  localize: async (store, preferred_locale) => { locale, messages }
  // (or same without `async`: (store, preferred_locale) => Promise.resolve({ locale, messages }))
}
```

## Miscellaneous client-side rendering options

```javascript
{
  ...

  // (optional)
  // Enables/disables Redux development tools (true/false)
  development_tools: _development_tools_,

  // (optional)
  // Loads localized messages (asynchronously)
  load_translation: async locale => messages
  // (or same without `async`: locale => Promise.resolve(messages))
}
```

## Gotchas

This library is build system agnostic: you can use your favourite Grunt, Gulp, Browserify, RequireJS, Webpack, etc.

If you're using Webpack then make sure you either build your server-side code with Webpack too (so that asset `require()` calls (images, styles, fonts, etc) inside React components work, see [universal-webpack](https://github.com/halt-hammerzeit/universal-webpack)) or use [webpack-isomorphic-tools](https://github.com/halt-hammerzeit/webpack-isomorphic-tools).

## Contributing

After cloning this repo, ensure dependencies are installed by running:

```sh
npm install
```

This module is written in ES6 and uses [Babel](http://babeljs.io/) for ES5
transpilation. Widely consumable JavaScript can be produced by running:

```sh
npm run build
```

Once `npm run build` has run, you may `import` or `require()` directly from
node.

After developing, the full test suite can be evaluated by running:

```sh
npm test
```

While actively developing, one can use (personally I don't use it)

```sh
npm run watch
```

in a terminal. This will watch the file system and run tests automatically 
whenever you save a js file.

When you're ready to test your new functionality on a real project, you can run

```sh
npm pack
```

It will `build`, `test` and then create a `.tgz` archive which you can then install in your project folder

```sh
npm install [module name with version].tar.gz
```

## License

[MIT](LICENSE)