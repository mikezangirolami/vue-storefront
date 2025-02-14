## Composition API

> If you already have an experience with Vue Composition API, it's safe to skip this section and start with [Vue Storefront Composables](#what-are-vue-storefront-composables)

Composition API is a new way to abstract and reuse the logic added in Vue 3.0. It allows you to create and observe a reactive state both inside the Vue component and outside it as a standalone function.

Let's try to build functionality for submitting a form with two fields: user name and password.

```html
<template>
  <form>
    <input type="text" placeholder="Username" />
    <input type="password" placeholder="Password" />
    <button type="submit">Submit</button>
  </form>
</template>
```

In the Vue component, Composition API methods should be called inside the new component option called `setup`:

```html
<script>
  export default {
    setup() {
      // here we will start using Composition API
    },
    data() {
      return {
        /* ... */
      };
    },
    methods: {
      /* ... */
    }
  };
</script>
```

First of all, we need two reactive values that will be bound to form fields. They can be created with Vue `ref` method:

```js
import { ref } from 'vue';

export default {
  setup() {
    const username = ref('');
    const password = ref('');

    return { username, password };
  }
};
```

The argument passed to `ref` is an _initial value_ of our reactive state properties. Under the hood, `ref` creates an object with a single property `value`:

```js
setup() {
  const username = ref('');
  const password = ref('');

  console.log(username.value) // => ''

  return { username, password };
}
```

After we returned reactive properties from the `setup`, they become available in Vue component options (such as `data` or `methods`) and in component template. `refs` returned from setup are automatically unwrapped - this means you don't access the `.value` anymore:

```html
<template>
  <form @submit="submitForm">
    <input v-model="username" type="text" placeholder="Username" />
    <input v-model="password" type="password" placeholder="Password" />
    <button type="submit">Submit</button>
  </form>
</template>

<script>
  import { ref } from 'vue';

  export default {
    setup() {
      const username = ref('');
      const password = ref('');

      return { username, password };
    }
    methods: {
      submitForm() {
        console.log(this.username)
      }
    }
  };
</script>
```

Methods can be also created inside the `setup` option:

```js{6-11}
export default {
  setup() {
    const username = ref('');
    const password = ref('');

    const submitForm = () => {
      // remember that inside `setup` you need to access ref's value
      console.log(username.value);
    };

    return { username, password, submitForm };
  }
};
```

Let's disable a button when username or password is empty. In order to do this, we can create a computed property - and this can be also done with the Composition API inside the `setup`:

```html{5,15-17}
<template>
  <form @submit="submitForm">
    <input v-model="username" type="text" placeholder="Username" />
    <input v-model="password" type="password" placeholder="Password" />
    <button :disabled="!isValid" type="submit">Submit</button>
  </form>
</template>

<script>
  import { ref, computed } from 'vue';

  export default {
    setup() {
      /*...*/
      const isValid = computed(
        () => username.value.length > 0 && password.value.length > 0
      );

      return { username, password, submitForm, isValid };
    }
  };
</script>
```

Now, we can make a final step and abstract all the form functionality _outside_ the component - in a standalone function:

```js
// useForm.js

import { ref, computed } from 'vue';

export function useForm() {
  const username = ref('');
  const password = ref('');

  const submitForm = () => {
    console.log(username.value);
  };

  const isValid = computed(
    () => username.value.length > 0 && password.value.length > 0
  );

  return { username, password, submitForm, isValid };
}
```

This function is what we call a **composable**: a reusable piece of logic with the reactive state. Composables can be imported and used in Vue components:

```js
// SomeComponent.vue

import { useForm } from './useForm.js';

export default {
  setup() {
    const { username, password, submitForm, isValid } = useForm();

    return { username, password, submitForm, isValid };
  }
};
```

Vue Storefront uses composables as its main API. We will take a look over them in the next section.

## What are Vue Storefront composables

Composable is a function that uses [Composition API](#composition-api) under the hood. Composables are the main public API of Vue Storefront and, in most cases, the only API except configuration you will work with.

You can treat composables as independent micro-applications. They manage their own state, handle server-side rendering, and rarely interact with each other (except [useUser](/composables/use-user.html) and [useCart](/composables/use-cart.html)). No matter what integration you are using, your application will always have the same set of composables with the same interfaces.

To use a composable, you need to import it from an integration you are using, and call it on the component `setup` option:

```js
import { useProduct } from '{INTEGRATION}';
import { onSSR } from '@vue-storefront/core`

export default {
  setup() {
    const { products, search} = useProduct('<UNIQUE_ID>');

    onSSR(async () => {
      await search(searchParams)
    })

    return {
      products
    };
  }
};
```

> `onSSR` is used to perform an asynchronous request on the server side and convey the received data to the client. You will learn more about it very soon.

For some composables (like `useProduct` ) you will need to pass a unique ID as a parameter (it can be a product ID, category ID etc.). Others (like `useCart`) do not require an ID passed. You can always check a composable signature in the [API Reference](TODO)

## Anatomy of a composable

Every Vue Storefront composable usually returns three main pieces:

- **Main data object**. A read-only object that represents data returned by the composable. For example, in `useProduct` it's a `products` object, in `useCategory` it's `category` etc.
- **Supportive data objects**. These properties depend directly on indirectly on the main data object. For example, `loading`, `error` or `isAuthenticated` from `useUser` depend on a `user`.
- **Main function that interacts with data object**. This function usually calls the API and updates the main data object. For example in `useProduct` and `useCategory` it's a `search` method,in `useCart` it's a `load` method. The rule of thumb here is to use `search` when you need to pass some search parameters. `load` is usually called when you need to load some content based on cookies or `localStorage`

```js
import { useProduct } from '{INTEGRATION}';

const { products, search, loading } = useProduct('<UNIQUE_ID>');

search({ slug: 'super-t-shirt' });

return { products, search, loading };
```

However, the example above won't work without the `onSSR` helper. Let's take a look at it.

### Using `onSSR` for server-side rendering

By default, Vue Storefront supports conveying server-side data to the client with Nuxt.js 2 where `setup` function is synchronous. Because of that, we can't use asynchronous functions if their results depend on each other (e.g. by loading `products` using `id` of a category that you have to fetch first).

To solve this issue, we provide a temporary solution - `onSSR`:

```js
import { useProduct } from '{INTEGRATION}';
import { onSSR } from '@vue-storefront/core';

export default {
  setup() {
    const { products, search, loading } = useProduct('<UNIQUE_ID>');

    onSSR(async () => {
      await search({ slug: 'super-t-shirt' });
    });

    return {
      products,
      loading
    };
  }
};
```

`onSSR` accepts a callback where we should call our `search` or `load` method asynchronously. This will change `loading` state to `true`. Once the API call is done, main data object (`products` in this case) will be populated with the result, and `loading` will become `false` again.

In the future, Vue 3 will provide an async setup and `onSSR` won't be needed anymore.

## What composables I can use?

Vue Storefront integrations are exposing the following composables:

#### Product Catalog

- `useProduct` - Managing a single product with variants (or a list).
- `useCategory` - Managing category lists (but not category products).
- `useFacet` - Complex catalog search with filtering.
- `useReview` - Product reviews.

#### User Profile and Authorization

- `useUser` - Managing user sessions, credentials and registration.
- `useUserShipping` - Managing shipping addresses.
- `useUserBilling` - Managing billing addresses.
- `useUserOrder` - Managing past and active user orders.

#### Shopping Cart

- `useCart` - Loading the cart, adding/removing products and discounts.

#### Wishlist/Favourite

- `useWishlist` - Loading the wishlist, adding/removing products.

#### CMS Content

- `useContent` - Fetching the CMS content. It is usually used in combination with `<RenderContent>`component.

#### Checkout

- `useShipping` - Saving the shipping address for a current order.
- `useShippingProvider` - Choosing a shipping method for a current order. Shares data with `VsfShippingProvider` component.
- `useBilling` - Saving the billing address for a current order.
- `usePaymentProvider` - Choosing a payment method for a current order. Shares data with `VsfPaymentProvider` component
- `useMakeOrder` - Placing the order.

#### Other

- `useVSFContext` - Accessing the integration API methods and client instances.
In a real-world application, these composables will most likely use different backend services under the hood yet still leverage the same frontend API. For instance within the same application `useProduct` and `useCategory` could use `commercetools`, `useCart` some ERP system, `useFacet` - Algolia etc.

## Error handling

A flexible way of error handling is essential for a framework like Vue Storefront. We decided to put all errors in reactive objects so they can be easily managed in the template.

Each composable returns an `error` computed property. It is an object which has names of async functions from composable as keys and Error instance or `null` as a value.

```vue
<template>
  <button @click="addItem(product)">Add to cart</button>
  <div v-if="error.addItem">{{ error.addItem.message }}</div>
</template>

<script>
export default {
  setup() {
    const { addItem, error } = useCart();

    return {
      addItem,
      error
    };
  }
};
</script>
```

There is a dedicated TypeScript interface for every composable. Take a look at this one from `useCart`:

```ts
export interface UseCartErrors {
  addItem: Error;
  removeItem: Error;
  updateItemQty: Error;
  load: Error;
  clear: Error;
  applyCoupon: Error;
  removeCoupon: Error;
}
```

:::details Where does the error come from?

Inside each async method, we are catching errors when they occur and save them to the reactive property called `error` under the key corresponding to the triggered method:

```ts
const { addItem, error } = useCart();

addItem({ product: null }); // triggers an error
error.value.addItem; // here you have error raised by addItem function
```

:::

:::details Where can I find an interface of the error property from a certain composable?

When you are writing a code inside a script part of the Vue component, your IDE should give you hints on every different type of the composable. That's why you probably do not need to check these interfaces in the core's code.

However, if you still want to check interfaces, you could find them inside [`packages/core/core/src/types.ts`](https://github.com/vuestorefront/vue-storefront/blob/next/packages/core/core/src/types.ts).

[//]: # 'TODO: This should be added to API reference'

:::

### How to listen for errors?

Let's imagine you have some global component for error notifications. You want to send information about every new error to this component. But how to know when a new error appears? You can observe an error object with a watcher:

```ts
import { useUiNotification } from '~/composables';

const { cart, error } = useCart();
const { send } = useUiNotification();

watch(() => ({...error.value}), (error, prevError) => {
  if (error.addItem && error.addItem !== prevError.addItem)
    send({ type: 'danger', message: error.addItem.message });
  if (
    error.removeItem &&
    error.removeItem !== prevError.removeItem
  )
    send({ type: 'danger', message: error.removeItem.message });
});
```

In this example, we are using `useUiNotification` - a composable that handles notifications state. You can read more about it in the API reference.

[//]: # 'TODO: This should be added to API reference'

### How to customize graphql queries?

If your integration uses GraphQL API, you may need to change the default query that is being sent to fetch the data. That's quite a common case and Vue Storefront also provides the mechanism for this.

Since the communication with the API goes through our middleware, all queries also are defined there.

To customize or even totally override the original (default) queries you need to follow two steps.

Firstly, you need to use a dedicated parameter: `customQuery` that tells the app, what to do with a query.
This parameter is an object that has a name of the queries as keys, and the name of the queries function under the values.


```ts
const { search } = useProduct();

search({ customQuery: { products: 'my-products-query' } }); 
```

In the example above, we are changing `products` query, and our function that will take care of this overriding is `my-products-query`. As a second step, we need to define that function.


Each custom query lives in the `middleware.config.js`, so it's the place where we should define `my-products-query`:

```js
module.exports = {
  integrations: {
    ct: {
      location: '@vue-storefront/commercetools-api/server',
      configuration: { /* ... */ },
      customQueries: {
        'my-products-query': ({ query, variables }) => {

          variables.locale = 'en'

          return { query, variables }
        }
      }
    }
  }
};
```

The custom query function always has in the arguments the default query and default variables and must return the query and its variables as well. In the body you can do anything you want with those parameters - you can override them or even change to the new ones.
