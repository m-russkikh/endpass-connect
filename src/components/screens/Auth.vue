<template>
  <screen>
    <v-frame :loading="!inited">
      <create-account-form
        v-if="authorized && isAccountsEmpty"
        :closable="isDialog"
        @request="handleAccountRequest"
        @cancel="handleAuthCancel"
      />
      <otp-form 
        v-else-if="otpEmail" 
        :closable="isDialog"
        :loading="loading"
        :error="error"
        @submit="handleOtpSubmit"
        @cancel="handleAuthCancel"
      />
      <message-form 
        v-else-if="!authorized && sent"
        :closable="isDialog"
        message="An email with authorization link was sent on your address. Open it in the same browser to sign in."
        @cancel="handleAuthCancel"
      />
      <message-form 
        v-else-if="authorized && sent"
        message="You are successfully authorized. You will be redicected in a few seconds."
      />
      <auth-form
        v-else
        :inited="inited"
        :loading="loading"
        :error="error"
        :closable="isDialog"
        @cancel="handleAuthCancel"
        @submit="handleAuthSubmit"
      />
    </v-frame> 
  </screen>
</template>

<script>
import isEmpty from 'lodash/isEmpty';
import { mapActions, mapGetters, mapState } from 'vuex';
import Screen from '../Screen.vue';
import VFrame from '../VFrame.vue';
import AuthForm from '../AuthForm.vue';
import OtpForm from '../OtpForm.vue';
import MessageForm from '../MessageForm.vue';
import CreateAccountForm from '../CreateAccountForm.vue';

export default {
  name: 'Auth',

  data: () => ({
    error: null,
    needAccount: false,
  }),

  computed: {
    ...mapState({
      inited: state => state.core.inited,
      loading: state => state.core.loading,
      sent: state => state.accounts.linkSent,
      otpEmail: state => state.accounts.otpEmail,
      accounts: state => state.accounts.accounts,
    }),
    ...mapGetters(['isDialog']),

    authorized() {
      return !!this.accounts;
    },

    confirmed() {
      return this.authorized && this.sent;
    },

    isAccountsEmpty() {
      return isEmpty(this.accounts);
    },
  },

  watch: {
    authorized: {
      handler() {
        this.handleAuthorizationDataChange();
      },
      immediate: true,
    },

    accounts: {
      handler() {
        this.handleAuthorizationDataChange();
      },
      immediate: true,
    },
  },

  methods: {
    ...mapActions([
      'auth',
      'cancelAuth',
      'confirmAuth',
      'confirmAuthViaOtp',
      'awaitAuthMessage',
      'awaitAuthConfirm',
      'awaitAccountCreate',
      'openCreateAccountPage',
    ]),

    async handleOtpSubmit(code) {
      try {
        await this.confirmAuthViaOtp({
          email: this.otpEmail,
          code,
        });
      } catch (err) {
        console.error('handle error', err);
        this.handleAuthError(err);
      }
    },

    async handleAuthSubmit(email) {
      try {
        await this.auth(email);
        await this.awaitAuthConfirm();
      } catch (err) {
        console.error(err);
        this.handleAuthError(err);
      }
    },

    handleAuthorizationDataChange() {
      const {
        authorized,
        awaitAccountCreate,
        isAccountsEmpty,
        confirmAuth,
      } = this;

      if (authorized && isAccountsEmpty) {
        awaitAccountCreate();
      } else if (authorized && !isAccountsEmpty) {
        confirmAuth();
      }
    },

    async handleAccountRequest() {
      this.openCreateAccountPage();
    },

    handleAuthCancel() {
      this.cancelAuth();
    },

    handleWindowClose() {
      this.cancelAuth();
    },

    handleAuthError(error) {
      this.error = error.message || 'Unexpected error, try login later';
    },
  },

  async created() {
    if (this.isDialog) {
      window.addEventListener('beforeunload', this.handleWindowClose);
    }
  },

  components: {
    Screen,
    VFrame,
    AuthForm,
    OtpForm,
    MessageForm,
    CreateAccountForm,
  },
};
</script>
