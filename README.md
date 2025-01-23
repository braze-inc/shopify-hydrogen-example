# Braze integration for Shopify Hydrogen

Discover how to connect Braze to your Shopify Hydrogen storefront.

## Onsite tracking

The first step is to initialize the Braze Web SDK. We recommend doing that by installing our NPM package and importing it in your `root.jsx` file. You will also need to include this setting in your vite.config.js file. 

Once imported, you must initialize the SDK within a “useEffect” hook:

```
import * as braze from "@braze/web-sdk"; // add this to the top of root.jsx file

export function Layout({children}) {
  const nonce = useNonce();
  /** @type {RootLoader} */
  const data = useRouteLoaderData('root');
  
  useEffect(() => {
    braze.initialize(data.brazeApiKey, {
      baseUrl: data.brazeApiUrl,
      enableLogging: true,
    });
    braze.openSession()    
  }, [])

  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        <Meta />
        <Links />
      </head>
      <body>
        {data ? (
          <Analytics.Provider
            cart={data.cart}
            shop={data.shop}
            consent={data.consent}
          >
            <PageLayout {...data}>{children}</PageLayout>
          </Analytics.Provider>
        ) : (
          children
        )}
        <ScrollRestoration nonce={nonce} />
        <Scripts nonce={nonce} />
      </body>
    </html>
  );
}
```

The values “data.brazeApiKey” and “data.brazeApiUrl” need to be included in the component loader. We recommend adding them as environment variables in your application:

```
export async function loader(args) {
  // Start fetching non-critical data without blocking time to first byte
  const deferredData = loadDeferredData(args);

  // Await the critical data required to render initial state of the page
  const criticalData = await loadCriticalData(args);

  const {storefront, env} = args.context;

  return defer({
    ...deferredData,
    ...criticalData,
    publicStoreDomain: env.PUBLIC_STORE_DOMAIN,
    brazeApiKey: env.BRAZE_API_KEY,
    brazeApiUrl: env.BRAZE_API_URL,
    customerData: customerData,
    shop: getShopAnalytics({
      storefront,
      publicStorefrontId: env.PUBLIC_STOREFRONT_ID,
    }),
    consent: {
      checkoutDomain: env.PUBLIC_CHECKOUT_DOMAIN,
      storefrontAccessToken: env.PUBLIC_STOREFRONT_API_TOKEN,
      withPrivacyBanner: false,
      // localize the privacy banner
      country: args.context.storefront.i18n.country,
      language: args.context.storefront.i18n.language,
    },
  });
}
```

## Track Shopify Account Login Event

Track when a shopper signs into their account and sync their user information to Braze. 

```
export function trackCustomerLogin(customerData, storefrontUrl) {
  const braze = window.braze;
  
  const customerId = customerData.id.substring(customerData.id.lastIndexOf('/') + 1)
  const customerSessionKey = `ab.shopify.shopify_customer_${customerId}`;
  const alreadySetCustomerInfo = sessionStorage.getItem(customerSessionKey);
  
  if(!alreadySetCustomerInfo) {
    const user = braze.getUser()
    braze.changeUser(customerId)
    user.setFirstName(customerData.firstName);
    user.setLastName(customerData.lastName);
    user.setEmail(customerData.emailAddress.emailAddress);
    user.setPhoneNumber(customerData.phoneNumber.phoneNumber);
    braze.logCustomEvent(
      "shopify_account_login",
      { source: storefrontUrl }
    )
    sessionStorage.setItem(customerSessionKey, customerId);
  }
}

```

Then, in the same useEffect hook that initializes the Braze SDK you can add the call to this function:

```
useEffect(() => {
  braze.initialize(data.brazeApiKey, {
    baseUrl: data.brazeApiUrl,
    enableLogging: true,
  });
  braze.openSession()
  
  data.isLoggedIn.then((isLoggedIn) => {
    if(isLoggedIn) {
      trackCustomerLogin(data.customerData, data.publicStoreDomain)
    }
  })
}, [])
```

Last part is to load the customer data in your loader function:

```
export async function loader(args) {
  // Start fetching non-critical data without blocking time to first byte
  const deferredData = loadDeferredData(args);

  // Await the critical data required to render initial state of the page
  const criticalData = await loadCriticalData(args);

  const {storefront, env} = args.context;

  const isLoggedIn = await deferredData.isLoggedIn;
  let customerData;
  if (isLoggedIn) {
    const { data, errors } = await args.context.customerAccount.query(
        CUSTOMER_DETAILS_QUERY,
    );
    customerData = data.customer
  } else {
    customerData = {}
  }

  return defer({
    ...deferredData,
    ...criticalData,
    publicStoreDomain: env.PUBLIC_STORE_DOMAIN,
    brazeApiKey: env.BRAZE_API_KEY,
    brazeApiUrl: env.BRAZE_API_URL,
    customerData: customerData,
    shop: getShopAnalytics({
      storefront,
      publicStorefrontId: env.PUBLIC_STOREFRONT_ID,
    }),
    consent: {
      checkoutDomain: env.PUBLIC_CHECKOUT_DOMAIN,
      storefrontAccessToken: env.PUBLIC_STOREFRONT_API_TOKEN,
      withPrivacyBanner: false,
      // localize the privacy banner
      country: args.context.storefront.i18n.country,
      language: args.context.storefront.i18n.language,
    },
  });
}
```

## Add tracking for product viewed and cart updated events

First, define a function that will call the Braze SDK. You could create a new file (such as “Tracking.jsx”) and import it from your components:

```
export function trackProductViewed(product, storefrontUrl) {
  const braze = window.braze || [];
  const eventData = {
    product_id: product.id.substring(product.id.lastIndexOf('/') + 1),
    product_name: product.title,
    variant_id: product.selectedOrFirstAvailableVariant.id.substring(product.selectedOrFirstAvailableVariant.id.lastIndexOf('/') + 1),
    image_url: product.selectedOrFirstAvailableVariant.image.url,
    product_url: `${storefrontUrl}/products/${product.handle}`,
    price: product.selectedOrFirstAvailableVariant.price.amount,
    currency: product.selectedOrFirstAvailableVariant.price.currencyCode,
    source: storefrontUrl,
    metadata: {
    sku: product.selectedOrFirstAvailableVariant.sku
  }

  }
  braze.logCustomEvent(
    "ecommerce.product_viewed",
    eventData 
  )
}
```

After that, you'll need to call this function whenever a user visits a product page. You can do that by adding a “useEffect” hook in the Product component located in “app/routes/products.$handle.jsx” file:

```
export default function Product() {
  /** @type {LoaderReturnData} */
  const {product, storefrontUrl} = useLoaderData();

  useEffect(() => {
    trackProductViewed(product, storefrontUrl)
  }, [])

  return (...)
}
```

The value for “storefrontUrl” is not present in the component loader by default, so you will need to add it:

```
async function loadCriticalData({context, params, request}) {
  const {handle} = params;
  const {storefront} = context;

  if (!handle) {
    throw new Error('Expected product handle to be defined');
  }

  const [{product}] = await Promise.all([
    storefront.query(PRODUCT_QUERY, {
      variables: {handle, selectedOptions: getSelectedProductOptions(request)},
    }),
    // Add other queries here, so that they are loaded in parallel
  ]);

  if (!product?.id) {
    throw new Response(null, {status: 404});
  }

  return {
    product,
    storefrontUrl: context.env.PUBLIC_STORE_DOMAIN,
  };
}
```

## Cart updates

For cart updates, besides tracking the “cart_updated” event it's also necessary to send the cart token value over to Braze, as we use it for processing order webhooks received from Shopify. This is done by creating an user alias for the user with the Shopify cart token as its name. 


First step is to define functions for tracking “cart_updated” and setting the cart token:

```
export function trackCartUpdated(cart, storefrontUrl) {
  const braze = window.braze || [];
  const eventData = {
    cart_id: cart.id,
    total_value: cart.cost.totalAmount.amount,
    currency: cart.cost.totalAmount.currencyCode,

    products: cart.lines.nodes.map((line) => {
      return {
        product_id: line.merchandise.product.id.toString(),
        product_name: line.merchandise.product.title,
        variant_id: line.merchandise.id.toString(),
        image_url: line.merchandise.image.url,
        product_url: `${storefrontUrl}/products/${line.merchandise.product.handle}`,
        quantity: Number(line.quantity),
        price: Number(line.cost.totalAmount.amount / Number(line.quantity))
      }
    }),
    source: storefrontUrl,
    metadata: {},
  };
  
  braze.logCustomEvent(
    "ecommerce.cart_updated",
    eventData 
  )
}

export function setCartToken(cart) {
  const cartId = cart.id.substring(cart.id.lastIndexOf('/') + 1) 
  const cartToken = cartId.substring(0, cartId.indexOf("?key="));
  if (cartToken) {
    const cartSessionKey = `ab.shopify.shopify_cart_${cartToken}`;
    const alreadySetCartToken = sessionStorage.getItem(cartSessionKey);

    if (!alreadySetCartToken) {
      braze.getUser().addAlias("shopify_cart_token", `shopify_cart_${cartToken}`)
      braze.requestImmediateDataFlush();
      sessionStorage.setItem(cartSessionKey, cartToken);
    }
  }
}
```

Last step is to call this function whenever the user cart gets updated. Hydrogen stores usually define a `CartForm` component that manages the cart object state. What you need to do is add another “useEffect” hook in the AddToCartButton component, that will call “trackCartUpdated” function whenever the form fetcher state changes:

```
export function AddToCartButton({
  analytics,
  children,
  disabled,
  lines,
  onClick,
}) {

  const fetcher = useFetcher({ key: "add-to-cart-fetcher" });

  useEffect(() => {
    if(fetcher.state === "idle" && fetcher.data) {
      trackCartUpdated(fetcher.data.updatedCart, fetcher.data.storefrontUrl)
      setCartToken(fetcher.data.updatedCart);
    }
  }, [fetcher.state, fetcher.data])

  return (
    <CartForm route="/cart" inputs={{lines}} fetcherKey="add-to-cart-fetcher" action={CartForm.ACTIONS.LinesAdd}>
      {(fetcher) => (
        <>
          <input
            name="analytics"
            type="hidden"
            value={JSON.stringify(analytics)}
          />
          <button
            type="submit"
            onClick={onClick}
            disabled={disabled ?? fetcher.state !== 'idle'}
          >
            {children}
          </button>
        </>
      )}
    </CartForm>
  );
}
```