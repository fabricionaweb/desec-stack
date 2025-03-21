<template>
  <div>
    <v-alert v-if="done" type="success">
      <p>
        We hereby confirm your donation to deSEC. We would like to <b>thank you</b> for
        your support. If you have any questions concerning your donation, how
        we use your money, or if we can do anything else for you: Please do not
        hesitate to contact us anytime.
      </p>
      <p>
        We will debit {{ amount }} € from your account within the next
        two weeks. Your mandate reference number is {{ mref }}; our
        creditor identifier is {{ creditorid }}.
      </p>
      <p>
        Please note that the payment is handled by {{ creditorname }}, which
        may be the name appearing on your bank statement.
      </p>
      <p>Again, thank you so much.</p>
      <v-btn depressed outlined block :to="{name: 'home'}">Done</v-btn>
    </v-alert>
    <v-form v-if="!done" @submit.prevent="donate" ref="form">
      <error-alert v-bind:errors="errors"></error-alert>

      <v-radio-group
          v-model="interval"
          mandatory
          row
      >
        <v-radio label="Just once" :value="0"></v-radio>
        <v-radio label="Monthly" :value="1"></v-radio>
        <v-radio label="Quarterly" :value="3"></v-radio>
      </v-radio-group>

      <v-text-field
              v-model="name"
              label="Full Name of the Account Holder"
              prepend-icon="mdi-account"
              outline
              required
              :disabled="working"
              :rules="name_rules"
              :error-messages="name_errors"
      />

      <v-text-field
              v-model="iban"
              label="IBAN"
              prepend-icon="mdi-bank"
              outline
              required
              :disabled="working"
              :rules="iban_rules"
              :error-messages="iban_errors"
              validate-on-blur
      />

      <v-text-field
              v-model="amount"
              label="Amount in Euros"
              prepend-icon="mdi-cash-100"
              outline
              required
              :disabled="working"
              :rules="amount_rules"
              :error-messages="amount_errors"
      />

      <v-text-field
              v-model="message"
              label="Message (optional)"
              prepend-icon="mdi-message-text-outline"
              outline
              :disabled="working"
              validate-on-blur
      />

      <v-text-field
              v-model="email"
              label="Email Address (optional)"
              prepend-icon="mdi-email"
              outline
              :disabled="working"
              :rules="email_rules"
              :error-messages="email_errors"
              validate-on-blur
      />

      <v-btn
              depressed
              block
              color="primary"
              type="submit"
              :disabled="working"
              :loading="working"
      >Donate Now</v-btn>
    </v-form>
  </div>
</template>

<script>
  import axios from 'axios';
  import {email_pattern} from '@/validation';
  import {digestError} from '@/utils';
  import ErrorAlert from '@/components/ErrorAlert';

  const HTTP = axios.create({
    baseURL: '/api/v1/',
    headers: {
    },
  });

  export default {
    name: 'DonateDirectDebitForm',
    components: {
      ErrorAlert,
    },
    data: () => ({
      valid: false,
      working: false,
      done: false,
      errors: [],

      /* from env */
      creditorid: process.env.VUE_APP_DESECSTACK_API_SEPA_CREDITOR_ID,
      creditorname: process.env.VUE_APP_DESECSTACK_API_SEPA_CREDITOR_NAME,

      /* account holder name field */
      name: '',
      name_rules: [v => !!v || 'We need the account holder\'s name to debit an account.'],
      name_errors: [],

      /* IBAN field */
      iban: '',
      iban_rules: [v => !!v || 'For direct debit, an IBAN is required. If you do not have an IBAN, please consider using alternative donation methods.'],
      iban_errors: [],

      /* amount field */
      amount: 10,
      amount_rules: [
              v => !!v || 'Please specify the amount you want to donate, in Euros.',
              v => !isNaN(v) || 'Please specify the amount as a decimal number.'
      ],
      amount_errors: [],

      /* message field */
      message: '',

      /* email field */
      email: '',
      email_rules: [v => v === '' || !!email_pattern.test(v || '') || 'This is not an email address.'],
      email_errors: [],

      /* donation interval (every N months) */
      interval: 1,

      /* sent by server */
      mref: '',
    }),
    methods: {
      async reset() {
        this.$refs.form.reset();
      },
      async donate() {
        if (!this.$refs.form.validate()) {
          return;
        }
        this.working = true;
        this.errors.splice(0, this.errors.length);
        try {
          let response = await HTTP.post('donation/', {
            amount: this.amount,
            name: this.name,
            iban: this.iban,
            bic: "",
            message: this.message,
            email: this.email,
            interval: this.interval,
          });
          this.mref = response.data.mref;
          this.done = true;
        } catch (ex) {
          let errors = await digestError(ex);
          for (const c in errors) {
            if (c === 'email') {
              this.email_errors = errors[c];
            } else if (c === 'amount') {
              this.amount_errors = errors[c];
            } else {
              this.errors.push(...errors[c]);
            }
          }
        }
        this.working = false;
      },
    },
  };
</script>
