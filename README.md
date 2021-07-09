# vue-telephone-input-vuetify
International Telephone Input with Vuetify.

<p align="center">
<img width="600px" alt="International Telephone Input with Vuetify." src="https://media.giphy.com/media/eNe0TdR03G7MMNkFab/giphy.gif"/>
</p>


**Table of Contents**

- [Getting started](#getting-started)
- [Installation](#installation)
  - [vue-cli](#vue-cli)
  - [vuetify](#vuetify)
  - [npm](#npm)
- [Usage](#usage)
  - [Props](#props)
  - [Events](#events)
- [Credits & Contributors](#Credits-&-Contributors)

## Installation

### Example Repository

You might want to follow and use the basic example usage of this library in this repository [Example](https://github.com/MatheusFrez/vue-telephone-input-with-vuetify)

OR try from scratch with below steps

### vue-cli
- create new vue project using `vue-cli`:

```bash
  vue create my-app
  cd my-app
```

### vuetify
- install `vuetify` to newly created vue project

```bash
  vue add vuetify
```

### npm
- install `vue-telephone-input-vuetify` to newly created vue project

```bash
  npm i vue-telephone-input-vuetify
```

Install the plugin into Vue:

With vuetify loader:

```javascript
  // vue.config.js

  "transpileDependencies": [
    "vuetify",
    "vue-telephone-input-vuetify"
  ]
```

```javascript
  // src/plugins/vuetify.js

  import Vue from 'vue';
  import Vuetify from 'vuetify/lib';

  Vue.use(Vuetify);

  export default new Vuetify({
  });

```

Without vuetify loader:

```javascript
  // vue.config.js

  "transpileDependencies": [
    "vuetify",
    "vue-telephone-input-vuetify"
  ]
```

```javascript
  // src/plugins/vuetify.js

  import Vue from 'vue';
  import Vuetify from 'vuetify';
  import 'vuetify/dist/vuetify.min.css'

  Vue.use(Vuetify);

  export default new Vuetify({
  });

```

```javascript
// src/main.js

import Vue from 'vue';
import VueTelephoneInputVuetify from 'vue-telephone-input-vuetify/lib';
import vuetify from "./plugins/vuetify";

Vue.use(VueTelephoneInputVuetify, {
  vuetify,
});

new Vue({
  vuetify,
  render: (h) => h(App),
}).$mount("#app");

```

> View all available options in [Props](#props).

Use the `vue-telephone-input-vuetify` component:

  ```html
  <template>
    <vue-telephone-input-vuetify v-model="phone"></vue-telephone-input-vuetify>
  <template>

  <script>
  export default {
    data() {
      return {
        phone: null
      }
    }
  };
  </script>
  ```
## Credits & Contributors

**Credits**
- Telephone Number parsing, validation by [awesome-phonenumber](https://www.npmjs.com/package/awesome-phonenumber).
- Country Codes data from [intl-tel-input](https://github.com/jackocnr/intl-tel-input/blob/master/src/js/data.js).
- User's country by [ip2c.org](https://ip2c.org/s), request using [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API).

Adapted based on (https://github.com/yogakurniawan/vue-tel-input-vuetify).
