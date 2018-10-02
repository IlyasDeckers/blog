---
layout: post
title: Vue.js - Create a global event bus
---

The event bus / publish-subscribe pattern, despite the bad press it sometimes gets, is still an excellent way of getting unrelated sections of your application to talk to each other. But wait! Before you go waste your time on another library, why not use Vue’s powerful built-in event bus?

## Creating the event bus

The first thing you’ll need to do is create the event bus and make the bus available to each Vue instance by defining them on the prototype.
```js
Object.defineProperty(Vue.prototype, '$bus', {
  get () {
    return this.$root.bus
  }
})
```
In your Vue component where you want to receive on the $bus put the following code. This listens to incoming requests send over the event bus.
```js

  mounted () {
    this.$bus.$on('funcName', data => {
      // do something
    })
  }
```
To trigger the defined event you simply use the following on any of your Vue components.
```js

  this.$bus.$emit('funcName', data)
```
## Use case

In the following example, I trigger the visibility of a button located next to my breadcrumbs. Depending on what page is shown I want to manipulate the button text and function it triggers.

```js
Breadcrumbs.vue

<template>
  <div>
    <button v-if="button.visible" 
            @click="$bus.$emit(button.func, true)" 
            class="btn btn-info d-none d-lg-block m-l-15">
       <i class="fa fa-plus-circle"></i> {{ button.text }}
    </button>
  </div>
</template>

export default {
  name: 'Breadcrumbs',

  data () {
    return {
      pageTitle: this.$route.name,
      button: {
        visible: false,
        func: '',
        text: ''
      }
    }
  },

  mounted () {
    this.$bus.$on('breadButton', data => {
      this.button = data
    })
  }
}
```
Now you can trigger the button's visibility and action from any of your Vue components.
```js

export default {
  mounted () {
    ...
    this.$bus.$emit('breadButton', {
      visible: true, 
      text: 'New Package',
      func: 'showOrderPage'
    })
    ...
  }
}
```

In more complex cases, you should consider employing a dedicated [state-management pattern](https://vuejs.org/v2/guide/state-management.html). 
[Full documentation](https://vuejs.org/v2/guide/components.html#Non-Parent-Child-Communication)
