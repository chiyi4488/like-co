<template>
  <BaseDialogV2
    ref="dialog"
    :is-show="true"
    :is-show-backdrop="true"
    :is-show-header="true"
    :is-closable="true"
    @update:isShow="onUpdateIsShow"
    @click-outside="onClosed"
  >
    <auth-core-register
      :is-sign-in="true"
      :is-fix-contact="true"
      :language="getCurrentLocale"
      :email="getUserInfo.email"
      @success="signInWithAuthCore"
    />
  </BaseDialogV2>
</template>


<script>
import { mapActions, mapGetters } from 'vuex';

import AuthCoreRegister from '~/components/AuthCore/Register';
import BaseDialogV2 from '~/components/dialogs/BaseDialogV2';

export default {
  name: 're-auth-dialog',
  data() {
    return {
    };
  },
  components: {
    BaseDialogV2,
    AuthCoreRegister,
  },
  computed: {
    ...mapGetters([
      'getUserInfo',
      'getIsShowReAuthDialog',
      'getCurrentLocale',
    ]),
  },
  methods: {
    ...mapActions([
      'setReAuthDialogShow',
      'setAuthCoreToken',
    ]),
    onUpdateIsShow(isShow) {
      if (isShow !== this.getIsShowReAuthDialog) {
        this.setIsShow(isShow);
      }
    },
    onToggleSignIn() {
      this.toggleAuthDialogIsSignIn();
    },
    onConfirm() {
      this.setIsShow(false);
      this.$emit('confirm');
      this.onClosed();
    },
    onCancel() {
      this.setIsShow(false);
      this.$emit('cancel');
      this.onClosed();
    },
    onClosed() {
      this.$emit('closed');
    },
    setIsShow(isShow) {
      this.setReAuthDialogShow(isShow);
    },
    close() {
      this.setIsShow(false);
    },
    async signInWithAuthCore({ accessToken }) {
      await this.setAuthCoreToken(accessToken);
      this.setIsShow(false);
    },
  },
};
</script>
