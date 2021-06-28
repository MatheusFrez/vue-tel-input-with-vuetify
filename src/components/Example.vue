<template>
  <v-container>
    <v-row justify="center">
      <v-col cols="6">
        <vue-tel-input-vuetify
          v-model="myPhone"
          :valid-characters-only="true"
          :default-country="'br'"
          select-label="Code"
          @input="onInput"
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

export default {
  name: 'Example',
  components: {
    VueTelInputVuetify
  },
  data: () => ({
    myPhone: '',
    phone: {
      number: '',
      valid: false,
      country: undefined
    }
  }),
  methods: {
    onInput(formattedNumber, phoneObject) {
      this.phone.number = ((phoneObject || {}).number || {}).international || '';
      this.phone.valid = (phoneObject || {}).valid;
      this.phone.country = ((phoneObject || {}).country || {}).name;
    }
  }
};
</script>
