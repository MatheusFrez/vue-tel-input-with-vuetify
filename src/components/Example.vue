<template>
  <v-container>
    <v-row justify="center">
      <v-col cols="6">
        <vue-tel-input-vuetify
          ref="tel"
          v-model="myPhone"
          :placeholder="placeholder"
          :only-countries="onlyCountries"
          default-country="br"
          select-label="Code"
          outlined
          disabled-fetching-country
          @input="onInput"
          @country-changed="changeCountry($event)"
          :rules="[(v) => this.validate(v)]"
        />
      </v-col>
    </v-row>
    <v-row justify="center">
      <v-col cols="6">
        <div v-if="(phone || {}).number" style="color: #e83e8c">
          <span>
            Number:
            <strong>{{ (phone || {}).number }}</strong
            >,&nbsp;
          </span>
          <span>
            Is valid:
            <strong>{{ phone.valid }}</strong
            >,&nbsp;
          </span>
          <span>
            Country:
            <strong>{{ phone.country }}</strong>
          </span>
        </div>
      </v-col>
    </v-row>
    <v-row justify="center">
      <v-col cols="6">
        <v-btn @click="myPhone = '+55 22999999999'">Set phone</v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import VueTelInputVuetify from '../../lib/vue-tel-input-vuetify.vue';
import PhoneNumber from 'awesome-phonenumber';

export default {
  name: 'Example',
  components: {
    VueTelInputVuetify
  },
  data: () => ({
    myPhone: '',
    country: 'BR',
    placeholder: 'Put your cellphone number here',
    phone: {
      number: '',
      valid: false,
      country: undefined
    },
    onlyCountries: ['br', 'ar', 'cl', 'co', 'mx', 'es', 'uy'],
    selectedCountry: {}
  }),
  methods: {
    onInput(formattedNumber, phoneObject) {
      this.phone.number = ((phoneObject || {}).number || {}).international || '';
      this.phone.valid = (phoneObject || {}).valid;
      this.phone.country = ((phoneObject || {}).country || {}).name;
    },
    changeCountry(country) {
      this.selectedCountry = country;
    },
    validate(v) {
      const pn = new PhoneNumber(v, this.country);
      return (v && ((pn || {}).a || {}).valid) || 'Telephone invalid';
    }
  }
};
</script>
