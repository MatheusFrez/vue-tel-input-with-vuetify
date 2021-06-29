<template>
  <div :class="['vue-tel-input-vuetify', wrapperClasses]">
    <div class="country-code">
      <v-select
        :label="selectCountryLabel"
        v-model="countryCode"
        @change="onChangeCountryCode"
        v-bind="$attrs"
        :items="sortedCountries"
        :disabled="disabledSelectCountry"
        item-text="name"
        item-value="iso2"
        return-object
        :clearable="false"
      >
        <template v-slot:selection>
          <div :class="activeCountry.iso2.toLowerCase()" class="vti__flag" />
        </template>
        <template v-slot:item="data">
          <span :class="data.item.iso2.toLowerCase()" class="vti__flag" />
          <span>{{ data.item.name }} {{ `+${data.item.dialCode}` }}</span>
        </template>
      </v-select>
    </div>
    <v-text-field
      ref="input"
      type="tel"
      :value="value"
      v-model="phone"
      v-bind="$attrs"
      v-on="$listeners"
      @input="onInput"
      @change="onChange"
    >
      <template #message="{ key, message }">
        <slot name="label" v-bind="{ key, message }" />
      </template>
    </v-text-field>
  </div>
</template>

<script>
import PhoneNumber from 'awesome-phonenumber';
import { getCountry, setCaretPosition } from './utils';
import allCountries from './all-countries';

// Polyfill for Event.path in IE 11: https://stackoverflow.com/a/46093727
function getParents(node, memo) {
  const parsedMemo = memo || [];
  const { parentNode } = node;
  if (!parentNode) return parsedMemo;
  return getParents(parentNode, parsedMemo.concat(parentNode));
}

export default {
  name: 'VueTelInputVuetify',
  directives: {
    // Click-outside by BosNaufal: https://github.com/BosNaufal/vue-click-outside
    'click-outside': {
      bind(el, binding, vNode) {
        // Provided expression must evaluate to a function.
        if (typeof binding.value !== 'function') {
          const compName = vNode.context.name;
          let warn = `[Vue-click-outside:] provided expression ${binding.expression} is not a function, but has to be`;
          if (compName) {
            warn += `Found in component ${compName}`;
          }
          console.warn(warn);
        }
        // Define Handler and cache it on the element
        const { bubble } = binding.modifiers;
        const handler = (e) => {
          // Fall back to composedPath if e.path is undefined
          const path = e.path || (e.composedPath ? e.composedPath() : false) || getParents(e.target);
          if (bubble || (path.length && !el.contains(path[0]) && el !== path[0])) {
            binding.value(e);
          }
        };
        el.__vueClickOutside__ = handler;
        // add Event Listeners
        document.addEventListener('click', handler);
      },
      unbind(el) {
        // Remove Event Listeners
        document.removeEventListener('click', el.__vueClickOutside__);
        el.__vueClickOutside__ = null;
      }
    }
  },
  props: {
    value: { type: String, default: '' },
    selectCountryLabel: { type: String, default: '' },
    disabledFetchingCountry: { type: Boolean, default: false },
    disabledSelectCountry: { type: Boolean, default: false },
    mode: { type: String, default: '' },
    allCountries: { type: Array, default: () => allCountries },
    defaultCountry: { type: String, default: '' },
    preferredCountries: { type: Array, default: () => [] },
    onlyCountries: { type: Array, default: () => [] },
    ignoredCountries: { type: Array, default: () => [] },
    wrapperClasses: { type: [String, Array, Object], default: '' },
    inputOptions: { type: Object, default: () => {} }
  },
  data: () => ({
    phone: '',
    activeCountry: { iso2: '' },
    open: false,
    selectedIndex: null,
    typeToFindInput: '',
    typeToFindTimer: null,
    cursorPosition: 0,
    countryCode: null
  }),
  computed: {
    parsedMode() {
      if (this.mode) {
        if (!['international', 'national'].includes(this.mode)) {
          console.error('Invalid value of prop "mode"');
        } else {
          return this.mode;
        }
      }
      if (!this.phone || this.phone[0] !== '+') return 'national';

      return 'international';
    },
    filteredCountries() {
      // List countries after filtered
      if (this.onlyCountries.length) return this.getCountries(this.onlyCountries);

      if (this.ignoredCountries.length) {
        return this.allCountries.filter(
          ({ iso2 }) =>
            !this.ignoredCountries.includes(iso2.toUpperCase()) && !this.ignoredCountries.includes(iso2.toLowerCase())
        );
      }
      return this.allCountries;
    },
    sortedCountries() {
      // Sort the list countries: from preferred countries to all countries
      const preferredCountries = this.getCountries(this.preferredCountries).map((country) => ({
        ...country,
        preferred: true
      }));
      return [...preferredCountries, ...this.filteredCountries];
    },
    phoneObject() {
      const result = PhoneNumber(this.phone || '', this.activeCountry.iso2).toJSON();
      Object.assign(result, {
        isValid: result.valid,
        country: this.activeCountry
      });
      if (!this.phone) {
        return {
          ...result,
          number: {
            input: ''
          }
        };
      }
      return result;
    },
    phoneText() {
      let key = 'input';
      if (this.phoneObject.valid) key = this.parsedMode;
      return (this.phoneObject || {}).number[key] || '';
    }
  },
  watch: {
    // eslint-disable-next-line func-names
    'phoneObject.valid': function(value) {
      if (value) this.phone = this.phoneText;
      this.$emit('validate', this.phoneObject);
    },
    value() {
      this.phone = this.value;
    },
    open(isDropdownOpened) {
      // Emit open and close events
      if (isDropdownOpened) this.$emit('open');
      else this.$emit('close');
    },
    phone(newValue, oldValue) {
      if (newValue) {
        if (newValue[0] === '+') {
          const code = PhoneNumber(newValue).getRegionCode();
          if (code) this.activeCountry = this.findCountry(code) || this.activeCountry;
        }
      }
      // Reset the cursor to current position if it's not the last character.
      if (oldValue && this.cursorPosition < oldValue.length) {
        this.$nextTick(() => setCaretPosition(this.$refs.input, this.cursorPosition));
      }

      if (this.phoneText) this.$emit('input', this.phoneText, this.phoneObject);
    },
    activeCountry(value) {
      if (value && value.iso2) this.$emit('country-changed', value);
    }
  },
  mounted() {
    this.initializeCountry()
      .then(() => {
        if (!this.phone && this.inputOptions && this.inputOptions.showDialCode && this.activeCountry.dialCode) {
          this.phone = `+${this.activeCountry.dialCode}`;
        }
        this.countryCode = this.activeCountry;
        this.$emit('validate', this.phoneObject);
      })
      .catch(console.error);
  },
  created() {
    if (this.value) this.phone = this.value.trim();
  },
  methods: {
    initializeCountry() {
      return new Promise((resolve) => {
        /**
         * 1. If the phone included prefix (+12), try to get the country and set it
         */
        if (this.phone && this.phone[0] === '+') {
          const activeCountry = PhoneNumber(this.phone).getRegionCode();
          if (activeCountry) {
            this.choose(activeCountry);
            resolve();
            return;
          }
        }
        /**
         * 2. Use default country if passed from parent
         */
        if (this.defaultCountry) {
          const defaultCountry = this.findCountry(this.defaultCountry);
          if (defaultCountry) {
            this.choose(defaultCountry);
            resolve();
            return;
          }
        }
        const fallbackCountry = this.findCountry(this.preferredCountries[0]) || this.filteredCountries[0];
        /**
         * 3. Check if fetching country based on user's IP is allowed, set it as the default country
         */
        if (!this.disabledFetchingCountry) {
          getCountry()
            .then((res) => {
              if (this.phone === '') this.activeCountry = this.findCountry(res) || this.activeCountry;
            })
            .catch((error) => {
              console.warn(error);
              /**
               * 4. Use the first country from preferred list (if available) or all countries list
               */
              this.choose(fallbackCountry);
            })
            .finally(() => resolve());
        } else {
          /**
           * 4. Use the first country from preferred list (if available) or all countries list
           */
          this.choose(fallbackCountry);
          resolve();
        }
      });
    },
    /**
     * Get the list of countries from the list of iso2 code
     */
    getCountries(list = []) {
      return list.map((countryCode) => this.findCountry(countryCode)).filter(Boolean);
    },
    findCountry(iso = '') {
      return this.allCountries.find((country) => country.iso2 === iso.toUpperCase());
    },
    getItemClass(index, iso2) {
      const highlighted = this.selectedIndex === index;
      const lastPreferred = index === this.preferredCountries.length - 1;
      const preferred = this.preferredCountries.some((c) => c.toUpperCase() === iso2);
      return {
        highlighted,
        'last-preferred': lastPreferred,
        preferred
      };
    },
    choose(country, toEmitInputEvent = false) {
      this.activeCountry = country || this.activeCountry || {};
      if (this.phone && this.phone[0] === '+' && this.activeCountry.iso2 && this.phoneObject.number.significant) {
        // Attach the current phone number with the newly selected country
        this.phone = PhoneNumber(this.phoneObject.number.significant, this.activeCountry.iso2).getNumber(
          'international'
        );
      } else if (this.inputOptions && this.inputOptions.showDialCode && country) {
        // Reset phone if the showDialCode is set
        this.phone = `+${country.dialCode}`;
      }
      if (toEmitInputEvent) {
        if (this.phoneText) this.$emit('input', this.phoneText, this.phoneObject);
      }
    },
    onInput(e) {
      // Returns response.number to assign it to v-model (if being used)
      // Returns full response for cases @input is used
      // and parent wants to return the whole response.
      if (this.phoneText) this.$emit('input', this.phoneText, this.phoneObject);
      // Keep the current cursor position just in case the input reformatted
      // and it gets moved to the last character.
      if (e && e.target) this.cursorPosition = e.target.selectionStart;
    },
    onChange(value) {
      if (value) this.$emit('change', value);
    },
    focus() {
      this.$refs.input.focus();
    },
    toggleDropdown() {
      if (this.disabled) return;
      this.open = !this.open;
    },
    clickedOutside() {
      this.open = false;
    },
    keyboardNav(e) {
      if (e.keyCode === 40) {
        // down arrow
        e.preventDefault();
        this.open = true;

        if (this.selectedIndex === null) this.selectedIndex = 0;
        else this.selectedIndex = Math.min(this.sortedCountries.length - 1, this.selectedIndex + 1);

        const selEle = this.$refs.list.children[this.selectedIndex];
        if (selEle.offsetTop + selEle.clientHeight > this.$refs.list.scrollTop + this.$refs.list.clientHeight) {
          this.$refs.list.scrollTop = selEle.offsetTop - this.$refs.list.clientHeight + selEle.clientHeight;
        }
      } else if (e.keyCode === 38) {
        // up arrow
        e.preventDefault();
        this.open = true;

        if (this.selectedIndex === null) this.selectedIndex = this.sortedCountries.length - 1;
        else this.selectedIndex = Math.max(0, this.selectedIndex - 1);

        const selEle = this.$refs.list.children[this.selectedIndex];
        if (selEle.offsetTop < this.$refs.list.scrollTop) this.$refs.list.scrollTop = selEle.offsetTop;
      } else if (e.keyCode === 13) {
        // enter key
        if (this.selectedIndex !== null) this.choose(this.sortedCountries[this.selectedIndex]);
        this.open = !this.open;
      } else {
        // typing a country's name
        this.typeToFindInput += e.key;
        clearTimeout(this.typeToFindTimer);
        this.typeToFindTimer = setTimeout(() => {
          this.typeToFindInput = '';
        }, 700);
        // don't include preferred countries so we jump to the right place in the alphabet
        const typedCountryI = this.sortedCountries
          .slice(this.preferredCountries.length)
          .findIndex((c) => c.name.toLowerCase().startsWith(this.typeToFindInput));
        if (typedCountryI >= 0) {
          this.selectedIndex = this.preferredCountries.length + typedCountryI;
          const selEle = this.$refs.list.children[this.selectedIndex];
          const needToScrollTop = selEle.offsetTop < this.$refs.list.scrollTop;
          const needToScrollBottom =
            selEle.offsetTop + selEle.clientHeight > this.$refs.list.scrollTop + this.$refs.list.clientHeight;
          if (needToScrollTop || needToScrollBottom)
            this.$refs.list.scrollTop = selEle.offsetTop - this.$refs.list.clientHeight / 2;
        }
      }
    },
    reset() {
      this.selectedIndex = this.sortedCountries.map((c) => c.iso2).indexOf(this.activeCountry.iso2);
      this.open = false;
    },
    onChangeCountryCode() {
      this.choose(this.countryCode, true);
    }
  }
};
</script>

<style src="./sprite.css"></style>
<style lang="scss">
.vti__flag {
  margin-right: 8px;
}

.vue-tel-input-vuetify {
  display: flex;
  align-items: center;

  .country-code {
    width: 75px;
  }

  li.last-preferred {
    border-bottom: 1px solid #cacaca;
  }

  .v-text-field {
    .v-select__selections {
      position: relative;
      .vti__flag {
        position: absolute;
        margin-left: 18px;
      }
    }
    &--outlined {
      .v-select__selections {
        .vti__flag {
          margin-left: auto;
        }
      }
    }
  }
}
</style>
