<p align="center">
  <img src="https://raw.githubusercontent.com/aspect-apps/redux-ql/master/assets/thumbnail-dark-redux-ql.png" width="190" height="190">
  <br />
  <a href="https://enjoy.gitstore.app/repositories/aspect-apps/redux-nl"><img src="https://enjoy.gitstore.app/repositories/badge-aspect-apps/redux-nl.svg"></a>
  <hr />
</p>

# Redux Network Layer (Redux NL)

Redux Network Layer is a network layer for your application powered by redux and redux-saga heavily inspired by GraphQL clients such as react-relay. 

## Prerequisites

This requires that you have both [redux](https://redux.js.org/) and [redux-saga](https://redux-saga.js.org/) setup in your project.

## Setup

```sh
npm install redux-nl
```

For ReduxNL to work correctly you need to setup both the redux-nl saga and action reducer. 

The ReduxNL saga powers your networks layer. This can be spawned into your root reducer with the secound parameter of the spawn function as your API base url e.g. `https://my-example-api/`. Configuring the URL in this manner allows you to setup different URLs for local, sandbox and production environments. The third parameter is your API specification, example below. **We are looking at ways to auto-generate the API specification at compile-time from a `/spec` endpoint defined in your API**.

```js
import ReduxNLSaga from "redux-nl";
import { spawn } from "redux-saga";
import APISpecification from "../example-api.js";

export function* rootSaga() {
  yield all([
    // ReduxNL Network Sagas
    spawn(ReduxNLSaga, ApiUrl, APISpecification),
    ...
```

The ReduxNL action reducer records a temporary instance of you latest action fired into the redux store, this allows us to provide the smart ReduxNL callbacks inside our React components. You need to add the following to your `combineReducers` function:

```js
import { ActionReducer } from "../libs/redux-nl";

const rootReducer = combineReducers({
    action: ActionReducer,
    ...
```

Lastly, we need to configure ReduxNL by running the setup function after you intialie your store. This takes a few parameters which will allow you to customise your network layer.

```js
const store = createStore(rootReducer, middleware);

ReduxNL.setup({ 
  store,
  delay: 1000, <--- adds a network delay for testing slow connections
  isDev: false <--- Things like delay and console.warns will be ignored when this is false
  errorMessage: ".." <--- Custom fallback message
});
```

**Note: the ideal in the future is to unify the above three steps into a single step redux middleware**

## Usage

ReduxNL allows you to make request from your React components (or outside your react components) and listen to the status of that request... the only difference is that ReduxNL dispatches the request result to the Redux store, allowing you to also update your global state.

The below example allows you to update your component in response to the fired request. You can pass paramters via the `payload` and `meta` properties, these will be used in your network request, more details below. The libary follows the [Flux Standard Action](https://github.com/redux-utilities/flux-standard-action) specification for redux actions.


```js
ReduxNL.post("/user/brands/{slug}", {
  payload: { slug },
  meta: {
    apiToken: authToken
  },
  onSuccess: (action) => {

  },
  onFailure: (action) => {

  },
  onFinal: (action) => {

  }
});

// -- You can also write the call as a promise --

ReduxNl.promise.post("/user/brands/{slug}").then(...).catch();

// OR...

try {
  const action = await ReduxNl.promise.post("/user/brands/{slug}");
} catch(action){
  // Handle error
} finally {
  // Do something
}

```

The above example will dispatch a `CREATE_USER_BRANDS_SLUG_RESPONSE` to the store once the request has completed (success or failure). You can listen to these actions in your reducer by using some redux-ql utilities:

```js
const CreateBrandResponse = ReduxNL.response.type.post("/user/brands/{slug}");
const InitialState = {
  data: [],
  updatedAt: null
};

export default (state = InitialState, action) => {
  switch (action.type) {
  case CreateBrandResponse: {
    ...
```

Available methods for fetching the action type string:

```js
const CreateBrandResponse = ReduxNL.response.type.post("/user/brands/{slug}") -> CREATE_USER_BRANDS_SLUG_RESPONSE
const UpdateBrandResponse = ReduxNL.response.type.patch("/user/brands/{slug}") -> UPDATE_USER_BRANDS_SLUG_RESPONSE
const DeleteBrandResponse = ReduxNL.response.type.delete("/user/brands/{slug}") -> DELETE_USER_BRANDS_SLUG_RESPONSE
const FetchBrandResponse = ReduxNL.response.type.get("/user/brands/{slug}") -> FETCH_USER_BRANDS_SLUG_RESPONSE

const CreateBrandRequest = ReduxNL.request.type.post("/user/brands/{slug}") -> CREATE_USER_BRANDS_SLUG_REQUEST
const UpdateBrandRequest = ReduxNL.request.type.patch("/user/brands/{slug}") -> UPDATE_USER_BRANDS_SLUG_REQUEST
const DeleteBrandRequest = ReduxNL.request.type.delete("/user/brands/{slug}") -> DELETE_USER_BRANDS_SLUG_REQUEST
const FetchBrandRequest = ReduxNL.request.type.get("/user/brands/{slug}") -> FETCH_USER_BRANDS_SLUG_REQUEST
```

#### Request Parameters

All paramters in `payload` are passed as data to POST, UPDATE and DELETE request.

#### Query Parameters

Query parameters e.g. `https://my-example-api/user?api_token=..."` are automatically added to the URL via the `meta` key.

#### Route Parameters

Route parameters are defined in your api specification as so `/user/brands/{slug}`. In this example, when a value with the key of `slug` is passed thorugh then it is automatically replaced in the URL e.g. `/user/brands/{slug}` -> `/user/brands/apple`.

#### Headers

```
{
  ...
  meta: {
    headers: {
      ... <---
    }
  }
}
```

## API Spec Example

```js
// example-api.js

export default [
    { path: "/auth/login", method: "POST" },
    { path: "/auth/password-reset", method: "POST" },
    { path: "/user/verification", method: "POST" }, 
    { path: "/user", method: "PATCH" }, 
    { path: "/user", method: "GET" }, 
    { path: "/organisations/{slug}", method: "GET" }, 
    { path: "/user/brands/all", method: "GET" }, 
    { path: "/user/credits", method: "GET" }, 
    { path: "/user/brands/{slug}", method: "POST" },
    { path: "/user/brands/{slug}", method: "DELETE" }, 
    { path: "/user/orders", method: "GET" }, 
    { path: "/user/orders/{id}", method: "GET" },
    { path: "/checkout/{slug}/stripe", method: "POST" },
    { path: "/user/orders/{id}", method: "PATCH" }, 
    { path: "/user/stripe/card", method: "GET" }, 
    { path: "/user/stripe/card", method: "POST" }, 
    { path: "/user/stripe/card", method: "DELETE" }, 
    { path: "/user/subscription", method: "GET" }, 
];
```

## Limitations

- Only supports a single API connection.
- Only supports single path parameter e.g. `"/{user}/orders/{id}"` would break the module.
