---
layout: post
title: Vue Quicktip - Fetch on Created, Then Watch
---

A common pattern you see in Vue is calling a function on created, then watch a property for changes and recall that function. Whilst this is not a bad pattern, I will show you a quick tip how you can improve this code.
```js
created () {
  this.fetchUsers();
}
watch: {
  searchText () {
    this.fetchUsers();
  },
}
```
Did you know that a watcher accepts method names, it can be a string referencing the function. A watcher does not have to be a function. This can be found in the documentation but is easy to miss. We can rewrite the watcher from before like this.

```js
watch: {
  searchText: 'fetchUsers',
}
```
Another neat feature in Vue watchers is `immediate: true`. This calls the function specified on the watcher on the creation of the component. Before we can use this we need to adjust the watchers, we create an object for that watcher like this.

```js
watch: {
  searchText: {
    handler: 'fetchUsers',
    immediate: true,
  },
}
```
As you can see we specified our function, as a string reference, as a handler and used immediate true. This watcher does exactly the same as the first code example of this post with less code. Fetch on created, then watch.
