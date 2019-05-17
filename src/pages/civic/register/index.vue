<template>
  <div>
    <div v-if="!stripe">Loading... (or maybe stripe is broken)</div>
    <button v-if="hasPaymentAPI" ref="paymentRequestButton" />
    <form id="payment-form" @submit.prevent="onSubmitPayment">
      <div class="form-row">
        <div ref="card" />

        <!-- Used to display form errors. -->
        <div v-if="error">{{ error }}</div>
      </div>
      <button v-if="card">Pay</button>
    </form>
  </div>
</template>

<script>
import { getStripePaymentAPI } from '~/util/api';

export default {
  middleware: 'authenticated',
  layout: 'dialog',
  data() {
    return {
      stripe: null,
      card: null,
      hasPaymentAPI: false,
      from: undefined,
      referrer: undefined,
      error: '',
    };
  },
  head() {
    return {
      title: this.$t('CivicPage.title'),
      meta: [
        {
          hid: 'description',
          name: 'description',
          content: this.$t('CivicPage.slogan'),
        },
        {
          hid: 'og:description',
          property: 'og:description',
          content: this.$t('CivicPage.slogan'),
        },
        {
          hid: 'og:image',
          property: 'og:image',
          content: 'https://liker.land/images/og/civic.png',
        },
      ],
      script: [{ src: 'https://js.stripe.com/v3' }],
    };
  },
  mounted() {
    if (window.sessionStorage) {
      this.from = window.sessionStorage.getItem('civicLikerFrom');
      this.referrer = window.sessionStorage.getItem('civicLikerReferrer');
    }
    this.initStripe();
  },
  methods: {
    initStripe() {
      if (!window.Stripe || !process.env.STRIPE_PUBLIC_KEY) return;
      const stripe = window.Stripe(process.env.STRIPE_PUBLIC_KEY);
      this.stripe = stripe;
      const paymentRequest = stripe.paymentRequest({
        country: 'HK',
        currency: 'usd',
        total: {
          label: 'Civic Liker',
          amount: 500,
        },
        requestPayerName: true,
        requestPayerEmail: true,
      });
      const elements = stripe.elements();
      const prButton = elements.create('paymentRequestButton', {
        paymentRequest,
        style: {
          paymentRequestButton: {
            type: 'buy', // default: 'default'
            theme: 'light-outline', // default: 'dark'
            height: '64px', // default: '40px', the width is always '100%'
          },
        },
      });
      paymentRequest.canMakePayment().then(result => {
        if (result) {
          this.hasPaymentAPI = true;
          this.$nextTick(() => prButton.mount(this.$refs.paymentRequestButton)); // wait for refs to show
          paymentRequest.on('token', async e => {
            try {
              await this.$axios.$post(getStripePaymentAPI(), {
                from: this.from,
                referer: this.referrer,
                token: result.token,
              });
              e.complete('success');
            } catch (err) {
              console.error(result.error); // eslint-disable-line no-console
              e.complete('fail');
            }
          });
        }
      });
      this.card = elements.create('card', {
        style: {
          base: {
            fontFamily:
              'open-sans,source-han-sans-traditional,Open Sans,Source Sans Pro,Noto Sans,Microsoft JhengHei,Microsoft YaHei,Arial,sans-serif',
          },
        },
      });
      this.card.mount(this.$refs.card);
    },
    onSubmitPayment(event) {
      this.stripe.createToken(this.card).then(result => {
        if (result.error) {
          console.error(result.error); // eslint-disable-line no-console
          this.error = result.error.message;
        } else {
          // TODO: check status field of return payload
          this.$axios.$post(getStripePaymentAPI(), {
            from: this.from,
            referer: this.referrer,
            token: result.token,
          });
          this.$router.push({ name: 'civic-payment-success' });
        }
      });
    },
  },
};
</script>
