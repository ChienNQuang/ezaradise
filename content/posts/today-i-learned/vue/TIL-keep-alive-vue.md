---
title: "TIL: how to keep state when user navigate back in Vue"
date: 2023-12-24T01:42:41+07:00
draft: true
---

It's almost 2AM, and I found THE solution to a problem I was having:

> How do I let the user navigates back and forth between pages and still keeps the state exactly where it is?

Okay okay, I did not read the full Vue's documentation. No need to hassle me. But what matters is here I am.

But really, how would you do it?

Of course it is to read the documentation! As the first time using Vue, and I did not have much experience with vue router, I saw the chance to have finally done what I was too lazy to do: to read everything on the vue-router's documentation page.

You know what I finally learned after reading it? 何も～ない

It's not that I am not that good at reading, or keeping them knowledge all in my head or anything. I genuinely did not find the thing I was looking for. Vue router only offers route parameters and query parameters at this moment. SOOOOO, I figured we can only use query params to do this now eh? But before that, I have to think of some way to self-implement this.

What I thought of in the first place, was, to use a store, say pinia, to save the state/any information needed to rerender/stay the same place as before. This approach is very naive of me, but I indeed saw the problem that implementing this is not easy (at least for me). I did not see how would I even use a full fledge DB to solve it, let alone a simple state manager/store.

Another alternative I thought of was to use some kind of a stack, that will keep track of what happen and store states in each stack item. But isn't this the same thing that browser history does?

What about URL fragments? Vue router does not support it.

Okay, so what next?

I had to implement a query params system that get passed to pages/containers and used it as default to render the exact same thing as the page before:

```vue
const props = defineProps<{
  targetId: string;
  projectId: string;
  tab?: string | undefined;
  itemId?: string | undefined;
}>();

watch(
  () => activeName.value,
  () => {
    router.push({
      query: {
        tab: activeName.value,
        itemId: props.itemId,
      },
    });
  },
);
```

This gave me a glimpse of hope, because it works! But it has some drawbacks:

- The URLs are ugly, things like `https://example.com/abc/xyz/:id?tab=desc&itemId=12` surely cannot be pretty in anyway, the more thing we need to keep track, the uglier the URL gets
- I have to keep track of the URL at every. single. container. that is inside one page, imagine having more than 3.

So even though this worked just fine, I still had belief that this is not the best way to do this.

Instead, I came through this built-in component: [KeepAlive](https://vuejs.org/guide/built-ins/keep-alive.html)

And oh my gosh is it a life saver to me. With such a handy, I put it at every layout page like this:

```Vue
<div class="container mx-auto overflow-auto">
  <router-view v-slot="{ Component }">
    <keep-alive>
      <component :is="Component" />
    </keep-alive>
  </router-view>
</div>
```

And that's it! Everything just works, now not only on one page, but on any page! What a magic, after this I have sworn to myself to read every single page in the documentation. But that was one of what I have learned today. Thanks for reading!