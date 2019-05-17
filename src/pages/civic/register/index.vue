<template lang="pug">
  .civic-register-page
    PageHeader
      template
        SiteNavBar.text-like-green

      main.page-content.page-content--narrow
        .civic-register-page__header
          LockIcon

        .text-12.text-center.font-600.leading-1_5.mt-16.px-32
          .text-gray-4a
            | {{ $t('CivicRegisterPage.chooseYourPaymentMethod') }}
          .text-gray-9b.mt-8
            | {{ $t('CivicRegisterPage.extraCharge') }}

        .civic-register-page__payment-method-list
          div
            LcLoadingIndicator.text-like-green(v-if="!stripe")

            template(v-if="hasPaymentAPI")
              button(ref="paymentRequestButton")

              .text-gray-4a.text-center.text-12.font-600.my-16
                | {{ $t('CivicRegisterPage.or') }}

            header.text-like-green.text-14.text-center.font-600(v-if="stripe")
              | {{ $t('CivicRegisterPage.creditCard') }}

            form.civic-register-page__payment-method-list-item.civic-register-page__credit-card-form(
              id="payment-form"
              @submit.prevent="onSubmitPayment"
            )
              .text-error.text-12.mt-8(v-if="error") {{ error }}
              .mt-8(ref="card")
              button.btn.btn--outlined.mt-24(v-if="card")
                | {{ $t('confirm') }}

</template>

<script>
import PageHeader from '~/components/PageHeader';
import SiteNavBar from '~/components/SiteNavBar';

import LockIcon from '~/assets/icons/lock-w-circle.svg';

import { getStripePaymentAPI } from '~/util/api';

export default {
  name: 'CivicRegisterPage',
  components: {
    PageHeader,
    SiteNavBar,
    LockIcon,
  },
  middleware: 'authenticated',
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
          content: this.$t('CivicPage.title'),
        },
        {
          hid: 'og:description',
          property: 'og:description',
          content: this.$t('CivicPage.title'),
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

<style lang="scss">
.civic-register-page {
  &__header {
    @apply flex;
    @apply justify-center;
    @apply items-center;

    @apply text-like-green;

    @apply mt-4;

    > svg {
      @apply fill-current;
    }
  }

  &__payment-method-list {
    @apply rounded;

    @apply mt-24;
    @apply p-24;
    @apply pb-32;

    @apply bg-white;

    > div {
      @apply flex;
      @apply flex-col;
      @apply items-center;
    }

    &-item {
      max-width: 320px;

      @apply w-full;
    }
  }

  &__credit-card-form {
    @apply flex;
    @apply flex-col;
    @apply mt-16;
  }
}
</style>
