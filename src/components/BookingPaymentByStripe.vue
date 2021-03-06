<template>
  <div>
    <v-row no-gutters="">
      <v-col cols="12">
        <div class="card-number" ref="cardNumber"></div>
      </v-col>
    </v-row>
    <v-row class="mt-n05" no-gutters="">
      <v-col cols="6">
        <div class="expire-number" ref="expireNumber"></div>
      </v-col>
      <v-col cols="6">
        <div class="cvv-number" ref="cvvNumber"></div>
      </v-col>
    </v-row>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import store from '../store';
import { formatDate } from '../helpers';
import { InternalMessagePassing } from '../types';
// stripe setup example: https://github.com/stripe-samples/accept-a-card-payment/blob/master/without-webhooks/client/web/script.js
// theme examples https://stripe.dev/elements-examples/

export default Vue.extend({
  name: 'booking-payment-by-stripe',
  data() {
    return {
      stripe: {} as any,
      card: {} as any
    };
  },
  mounted() {
    this.cleanup();
    this.init();
  },
  methods: {
    cleanup() {
      (this as any).$store.dispatch('payment/updateIsPaymentLoading', false);
      (this as any).$store.dispatch('payment/updatePaymentError', '');
    },
    async init() {
      try {
        await this.getStripeKey();
        await this.waitForStripeScript();
        await this.createStripeComponent(this.stripeKey, this.accountId);
      } catch (error) {
        (this as any).$store.dispatch('payment/updatePaymentError', error.message);
      }
    },
    async waitForStripeScript() {
      return new Promise((resolve, reject) => {
        const stripeScriptTag = document.getElementById('stripe-script');
        if (!stripeScriptTag) {
          reject('Stripe not available');
          return;
        }

        // @ts-ignore
        if (window.GLOBAL_stripeIsLoaded) {
          resolve();
          return;
        }
        const onload = () => {
          resolve();
        };
        stripeScriptTag.addEventListener('load', onload);

        this.$once('hook:destroyed', () => {
          stripeScriptTag.removeEventListener('load', onload);
        });

        setTimeout(() => {
          reject('Error in loading Stripe');
        }, 20000);
      });
    },
    async submit(): Promise<InternalMessagePassing> {
      let result;
      try {
        result = await this.payByStripe();
      } catch (error) {
        result = { error: true, message: error.message };
      }
      return result;
    },
    async createStripeComponent(stripeKey, accountId) {
      // @ts-ignore
      this.stripe = window.Stripe(await stripeKey, {
        stripeAccount: accountId
      });
      // @ts-ignore
      const themes = this.$vuetify.theme.themes;

      var elementStyles = {
        base: {
          color: themes.dark.light,
          fontSmoothing: 'antialiased',
          fontSize: '16px',
          fontFamily: 'Montserrat, Segoe UI, sans-serif',
          ':focus': {
            color: themes.dark.light
          },
          '::placeholder': {
            color: themes.dark.light
          }
        },
        invalid: {
          color: themes.dark.error,
          iconColor: themes.dark.error
        }
      };
      const elements = this.stripe.elements({
        fonts: [
          {
            cssSrc: 'https://fonts.googleapis.com/css?family=Montserrat'
          }
        ],
        locale: 'auto'
      });

      const elementClasses = {
        focus: 'focus',
        empty: 'empty',
        invalid: 'invalid'
      };

      var cardNumber = elements.create('cardNumber', {
        style: elementStyles,
        classes: elementClasses,
        placeholder: 'Card number'
      });
      cardNumber.mount(this.$refs.cardNumber);

      // sending only one input to confirmCardPayment() is enough
      this.card = cardNumber;

      var cardExpiry = elements.create('cardExpiry', {
        style: elementStyles,
        classes: elementClasses,
        placeholder: 'Expiration'
      });
      cardExpiry.mount(this.$refs.expireNumber);

      var cardCvc = elements.create('cardCvc', {
        style: elementStyles,
        classes: elementClasses,
        placeholder: 'CVV'
      });
      cardCvc.mount(this.$refs.cvvNumber);
    },
    payByStripe() {
      const that = this;
      return (this as any).$store.dispatch('payment/payByStripe', {
        stripe: this.stripe,
        clientSecret: this.clientSecret,
        card: this.card
      });
    },
    getStripeKey() {
      return (this as any).$store.dispatch('payment/getStripeKey');
    }
  },
  computed: {
    stripeKey() {
      return (this as any).$store.getters['payment/stripeKey'];
    },
    accountId() {
      return (this as any).$store.getters['payment/accountId'];
    },
    customBookingInfo() {
      return (this as any).$store.getters['booking/customBookingInfo'];
    },
    clientSecret() {
      return (this as any).$store.getters['payment/clientSecret'];
    },
    reservationId() {
      return (this as any).$store.getters['booking/reservationId'];
    }
  }
});
</script>

<style lang="scss">
@import '@/styles/utility.scss';
</style>

<style lang="scss" scoped>
.StripeElement {
  height: rem(56px);
  padding-top: rem(18px);
  padding-left: rem(10px);
  padding-right: rem(10px);
  width: 100%;
  border: 1px solid map-get($grey, 'lighten-1');
  background-color: transparent;
  color: map-get($grey, 'lighten-1');
  &.invalid {
    border-color: map-get($red, 'base');
    border-width: rem(2px);
  }
  &.card-number {
    border-radius: $border-radius-root $border-radius-root 0 0;
    &.invalid {
      border-bottom-width: rem(1px);
    }
  }
  &.expire-number {
    border-radius: 0 0 0 $border-radius-root;
    border-right-width: rem(1px);
    &.invalid {
      border-right-width: rem(1px);
    }
  }
  &.cvv-number {
    border-radius: 0 0 $border-radius-root 0;
    border-left-width: 0;
    &.invalid {
      border-left-width: rem(1px);
    }
  }
}
</style>
