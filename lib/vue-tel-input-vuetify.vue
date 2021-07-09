<template>
  <div :class="['vue-tel-input-vuetify', wrapperClasses]">
    <div class="country-code">
      <v-select
        v-model="countryCode"
        :label="selectCountryLabel"
        v-bind="attrsToSelectField"
        :items="sortedCountries"
        :disabled="disabledSelectCountry"
        item-text="name"
        item-value="iso2"
        return-object
        :clearable="false"
        @change="onChangeCountryCode"
      >
        <template #selection>
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
      v-model="phone"
      type="tel"
      :value="value"
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
import { getCountry } from './utils';
import allCountries from './all-countries';

export default {
  name: 'VueTelInputVuetify',
  props: {
    value: { type: String, default: '' },
    selectCountryLabel: { type: String, default: '' },
    disabledFetchingCountry: { type: Boolean, default: false },
    disabledSelectCountry: { type: Boolean, default: false },
    mode: { type: String, default: '' },
    allCountries: { type: Array, default: () => allCountries },
    defaultCountry: { type: String, default: 'br' },
    preferredCountries: { type: Array, default: () => [] },
    onlyCountries: { type: Array, default: () => [] },
    ignoredCountries: { type: Array, default: () => [] },
    wrapperClasses: { type: [String, Array, Object], default: '' },
    inputOptions: { type: Object, default: () => {} }
  },
  data: () => ({
    phone: '',
    activeCountry: { iso2: '' },
    countryCode: null
  }),
  computed: {
    attrsToSelectField() {
      let attrsReady = { ...this.$attrs };
      delete attrsReady['rules'];
      return attrsReady;
    },
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
    'phoneObject.valid': function(value) {
      if (value) this.phone = this.phoneText;
      this.$emit('validate', this.phoneObject);
    },
    value() {
      this.phone = this.value;
    },
    phone(newValue) {
      if (newValue) {
        if (newValue[0] === '+') {
          const code = PhoneNumber(newValue).getRegionCode();
          if (code) this.activeCountry = this.findCountry(code) || this.activeCountry;
        }
      }
      this.$emit('input', this.phoneText, this.phoneObject);
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
            return resolve();
          }
        }
        /**
         * 2. Use default country if passed from parent
         */
        if (this.defaultCountry) {
          const defaultCountry = this.findCountry(this.defaultCountry);
          if (defaultCountry) {
            this.choose(defaultCountry);
            return resolve();
          }
        }
        const fallbackCountry = this.findCountry(this.preferredCountries[0]) || this.filteredCountries[0];

        if (!this.disabledFetchingCountry) {
          getCountry()
            .then((res) => {
              if (this.phone === '') this.activeCountry = this.findCountry(res) || this.activeCountry;
            })
            .catch((error) => {
              console.warn(error);
              this.choose(fallbackCountry);
            })
            .finally(() => resolve());
        } else {
          this.choose(fallbackCountry);
          return resolve();
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
      if (toEmitInputEvent) this.$emit('input', this.phoneText, this.phoneObject);
    },
    onInput() {
      // Returns response.number to assign it to v-model (if being used)
      // Returns full response for cases @input is used
      // and parent wants to return the whole response.
      this.$emit('input', this.phoneText, this.phoneObject);
    },
    onChange(value) {
      if (value) this.$emit('change', value);
    },
    focus() {
      this.$refs.input.focus();
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
