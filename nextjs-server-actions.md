---
title: >
  NextJS 13.4: Server Actions
subtitle: Combining Client-Side Interactivity with Server-Side Data Fetching with the Help of Server Actions
date: Oct 10, 2023
---

# NextJS 13.4: Combining Client-Side Interactivity with Server-Side Data Fetching with the Help of Server Actions

## Server Actions

With the release of Next.js 13.4, an exciting new feature called server actions has been introduced, enabling the combination of client-side interactivity and server-side data fetching. Server Actions are an alpha feature in Next.js, built on top of React Actions. They enable server-side data mutations, reduced client-side JavaScript, and progressively enhanced forms. They can be defined inside Server Components and/or called from Client Components:

>"The React ecosystem has seen a lot of innovation and exploration of ideas around forms, managing form state, and caching and revalidating of data. Over time, React has become more opinionated about some of these patterns. For example, recommended “uncontrolled components” for form state.
>
>The current ecosystem of solutions has either been reusable client-side solutions or primitives built into frameworks. Until now, there hasn't been a way to compose server mutations and data primitives. The React team has been working on a first-party solution for mutations.
>
>We're excited to announce support for experimental Server Actions in Next.js, enabling you to mutate data on the server, calling functions directly without needing to create an in-between API layer."
>
>[**Next.js Blog**](https://nextjs.org/blog/next-13-4#server-actions-alpha)


To be able to use Server Actions in your Next.js project you have to enable the experimental serverActions flag.

```next.config.js
module.exports = {
  experimental: {
    serverActions: true,
  },
}
```
Server Actions can be defined in two places:
- Inside the component that uses it (Server Components only)
- In a separate file (Client and Server Components), for reusability. You can define multiple Server Actions in a single file.

## Problem

In the development of my ecommerce website **[BeatsXchange](https://beatsxchange.vercel.app)**, I encountered a problem when implementing the cart functionality: I needed to have access to the prismaClient and use callbacks in the same component. Specifically, I had to implement the addToCart functionality in app/[product]/page.tsx and the resetCart functionality in the app/cart/page.tsx and have the cart page being refreshed everytime one of this actions were executed.

```app/[product]/page.tsx
import Button from '@/components/Button'

async function getProduct(productId: string) {
  // get the product from the database
  // ...
  return product
}

async function addProductToCart(productId: string) {
  // add product to the cart of the user in the database
}

export default async function Page({params}: { params: { product: string } }) {
  const product: Product | null = await getProduct(params.product)
  return product === null ? (
    <div>
      Page not found
    </div>
  ) : (
    <div>
      // ...
      <Button onClick={() => addProductToCart()} content='Add to cart'/>
    </div>
  )
}
```
```app/cart/page.tsx
import Button from "@/components/Button"

async function getCartItems(userId: string) {
  // ...
  return cartItems
}

async function removeAllCartItems(userId: string) {
  await prisma.cartItem.deleteMany({
    where: {
      userId: userId
    }
  })
}

export default async function Page() {
  const {userId} = auth(); // auth object from ClerkClient
  if (userId === null) return
  return (
    <div>
      // displayed the cart items
      // ...
      <Button onClick={() => removeAllCartItems()} params={userId} content='Reset Cart'/>
    </div>
  )
}
```

Both routes use the `<Button/>` component, which faced limitations when trying to pass event handlers as props.
```components/Button.tsx
'use client'

export default function Button({onClick, content}: {onClick: Promise<void>, content: string}) {
  return <button onClick={onClick}>{content}</button>
}
```

I tried to fidget with this approach and make it work but I had two problems that constantly came up:
1. You can't pass functions (event handlers) as props
```
ERROR: Event handlers cannot be passed to Client Component props.
  <... onClick={function} params=... content=...>
               ^^^^^^^^^^
If you need interactivity, consider converting part of this to a Client Component.
```
2. The cart page did not refresh and show the new content after an action was performed.

## Solution

As I searched for a solution for this issue I discovered Server Actions. By transforming the functions into server actions (by adding the 'use server' directive at the top of the body) and then passing them and their parameters to the \<Button/\> component I resolved the issue of passing event handlers as props `ERROR: Event handlers cannot be passed to Client Component props`. Additionally, by utilizing the **useTransition()** hook in the client component and revalidatePath('/cart') in the server actions, the cart page would now be automatically refreshed with the updated content.
```components/Button.tsx
'use client'

import { useTransition } from "react"

export default function Button({onClick, params, content}: {onClick: Function, params: any, content: string}) {
  let [isPending, startTransition] = useTransition()
  return <button onClick={() => startTransition(() => onClick(params))}>{content}</button>
}
```
```app/cart/page.tsx
async function removeAllCartItems(userId: string) {
  'use server'
  await prisma.cartItem.deleteMany({
    where: {
      userId: userId
    }
  })
  revalidatePath('/cart')
}

export default async function Page() {
  const {userId} = auth();
  if (userId === null) return
  const items = await getCartItems(userId)
  return (
    <div>
      {items.map(item => <p>{item.product.name}</p>}
      <Button onClick={removeAllCartItems} params={userId} content='Reset Cart'/>
    </div>
  )
}
```

## Caveat

When calling Server Actions from a different page, revalidatePath('/cart') only worked the first time and did not refresh the cart page in subsequent calls. This is a known issue that is currently being fixed. As a temporary workaround, I accessed the cart page using the old anchor instead of using the next/Link component.
```components/Nav.tsx
<a href='/cart'>Cart</a>
```
## Conclusion

Server actions allow for server-side data mutations, reduced client-side JavaScript, and enhanced form management. By enabling server actions, you can perform server-side tasks without an API endpoint and integrate those with client specific interactivity.

Although there are a few advantages compared to using API endpoints, like being able to integrate multiple platforms with the same API, server actions can definetely be the best option in some instances like the one above.

It's important to note that server actions are currently in an alpha stage, which means there is ongoing development and refinement happening. Although they are not yet stable, the potential for improvement and the addition of new features is something to be excited about.

If you're interested, you can explore the code for the project mentioned above on [**GitHub**](https://github.com/programmeriosif/beatsXchange). Additionally, feel free to connect with me on [**LinkedIn**](https://www.linkedin.com/in/iosif-ioan-buliga). And remember, a smile is the best error handler.
