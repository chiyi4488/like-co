<template>
  <BaseDialogV2
    ref="dialog"
    v-bind="$testID('AuthDialog')"
    :class="{
      'auth-dialog': true,
      'auth-dialog--blocking': isBlocking,
    }"
    :is-show="shouldShowDialog"
    :is-show-backdrop="!isSinglePage"
    :is-show-header="shouldShowHeader"
    :is-closable="!isSinglePage"
    @update:isShow="onUpdateIsShow"
    @click-outside="onClosed"
  >

    <div
      v-if="!isBlocking || isSinglePage"
      slot="header-left"
      class="auth-dialog__header-left"
    >
      <a
        v-if="isSignIn && isUsingAuthCore && !isMobileClient && currentTab === 'portal'"
        class="auth-dialog__legacy-login-button"
        @click="onClickUseLegacyButton"
      >
        {{ $t('AuthDialog.SignInWithLegacy.title') }}
      </a>
      <a
        v-else-if="currentTab === 'portal' || currentTab === 'error'"
        @click="onClickBackButton"
      >
        {{ $t('General.back') }}
      </a>
    </div>

    <div
      slot="header-center"
      class="auth-dialog__header-center"
    >
      <div class="auth-dialog__logo">
        <lc-avatar
          v-if="avatar"
          :src="avatar"
          :halo="avatarHalo"
          size="large"
          is-full-width
        />
        <template v-else>
          <img :src="LikeCoinLogo">
        </template>
      </div>
      <i18n
        v-if="avatar && fromDisplayName"
        class="auth-dialog__support-creator"
        path="AuthDialog.supportCreator"
        tag="div"
      >
        <span
          class="lc-color-like-green"
          place="name"
        >{{ fromDisplayName }}</span>
      </i18n>
    </div>

    <div
      :style="contentStyle"
      :class="[
        'auth-dialog__content',
        {
          'auth-dialog__content--with-avatar': !!avatar,
        }
      ]"
    >
      <div class="auth-dialog__tab-container">
        <Transition
          :css="false"
          :mode="tabTransitionMode"
          appear
          @leave="onTabLeave"
          @before-enter="onTabBeforeEnter"
          @enter="onTabEnter"
        >

          <div
            v-if="currentTab === 'portal'"
            v-bind="tabProps"
            class="auth-dialog__tab auth-dialog__tab--index auth-dialog__tab--portal"
          >
            <auth-core-register
              v-if="isUsingAuthCore"
              :is-sign-in="isSignIn"
              :language="getCurrentLocale"
              :redirect-url="getAuthCoreRedirectUrl"
              :social-login-pane-option="socialLoginPaneOption"
              @loaded="onAuthCoreLoaded"
              @loginWidgetLoaded="onAuthCoreLoginWidgetLoaded"
              @registerStarted="onAuthCoreRegisterStarted"
              @oauthStarted="onAuthCoreOAuthStarted"
              @loginStarted="onAuthCoreLoginStarted"
              @navigation="onAuthCoreNavigation"
              @success="signInWithAuthCore"
            />
            <signin-portal
              v-else
              class="base-dialog-v2__corner-block"
              :is-sign-in="isSignIn"
              :is-show-close-button="closable"
              :avatar="avatar"
              @toggle-sign-in="onToggleSignIn"
              @close="setIsShow(false)"
              @submit="signInWithPlatform"
            />
          </div>

          <div
            v-else-if="currentTab === 'loading'"
            v-bind="tabProps"
            class="auth-dialog__tab lc-padding-vertical-64"
          >
            <div class="lc-dialog-container-1">
              <h1 class="lc-font-size-32 lc-margin-bottom-8 lc-text-align-center lc-mobile">
                {{ $t('AuthDialog.label.loading') }}
              </h1>
            </div>
          </div>
          <div
            v-else-if="currentTab === 'register'"
            v-bind="tabProps"
            class="auth-dialog__tab lc-padding-vertical-16"
          >
            <register-form
              :prefilled-data="signInPayload"
              :is-edit-email="isEmailEditable"
              :platform="platform"
              @register="register"
            />
          </div>

          <div
            v-else-if="currentTab === 'error'"
            v-bind="tabProps"
            class="auth-dialog__tab lc-padding-vertical-16"
          >
            <div class="lc-dialog-container-1">
              <h1 class="lc-font-size-32 lc-margin-bottom-16 lc-mobile">
                {{ errorTitle }}
              </h1>
              <p class="lc-font-size-16 lc-color-like-gray-4 lc-padding-bottom-32">
                {{ errorMessage }}
              </p>

              <!-- Suggest legacy user to login with wallet -->
              <i18n
                v-if="errorMessageForLegacyUser"
                :path="errorMessageForLegacyUser"
                class="lc-font-size-16 lc-color-like-gray-4 lc-margin-bottom-32"
                tag="p"
              >
                <span
                  class="lc-color-like-green"
                  place="id"
                >{{ signInPayload.user }}</span>
                <span
                  class="lc-color-like-green"
                  place="email"
                >{{ signInPayload.email }}</span>
                <a
                  class="lc-color-light-burgundy lc-underline"
                  place="signIn"
                  @click="onClickSignWithWalletInError"
                >{{ $t('AuthDialog.Register.signWithWallet') }}</a>
              </i18n>

            </div>

            <div class="lc-dialog-container-1 lc-button-group">
              <md-button
                class="md-likecoin"
                @click="onDismissError"
              >
                {{ errorConfirmTitle }}
              </md-button>
            </div>
          </div>

          <div
            v-else-if="currentTab === 'loginSuccessful'"
            v-bind="tabProps"
            class="auth-dialog__tab lc-padding-vertical-64"
          >
            <div class="lc-dialog-container-1">
              <h1 class="lc-font-size-32 lc-margin-bottom-8 lc-text-align-center lc-mobile">
                {{ $t('AuthDialog.label.loginSuccessful') }}
              </h1>
            </div>
          </div>

          <div
            v-else-if="currentTab.split('-')[0] === 'loginFailure'"
            v-bind="tabProps"
            class="auth-dialog__tab lc-padding-top-16 lc-padding-bottom-24"
          >
            <div class="lc-dialog-container-1">
              <h1 class="lc-font-size-32 lc-margin-bottom-8 lc-mobile">
                {{ $t('AuthDialog.Failure.SignIn.header') }}
              </h1>
              <p class="lc-font-size-16 lc-color-like-gray-4 lc-margin-bottom-32">
                {{ $t('AuthDialog.Failure.SignIn.message') }}
              </p>
            </div>
            <div class="lc-dialog-container-1 lc-button-group">
              <md-button
                class="md-likecoin"
                @click="signInWithPlatform(currentTab.split('-')[1], { isAllowRedirect: false })"
              >
                {{ $t('AuthDialog.Failure.SignIn.confirm') }}
              </md-button><br><a
                class="lc-color-light-burgundy lc-underline"
                @click="currentTab = 'portal'"
              >
                {{ $t('AuthDialog.Failure.SignIn.cancel') }}
              </a>
            </div>
          </div>

        </Transition>
      </div>
    </div>

  </BaseDialogV2>
</template>


<script>
import Vue from 'vue'; // eslint-disable-line import/no-extraneous-dependencies
import { mapActions, mapGetters } from 'vuex';
import { ResizeObserver } from 'resize-observer';
import URL from 'url-parse';

import {
  MIN_USER_ID_LENGTH,
  MAX_USER_ID_LENGTH,
  LOGIN_CONNECTION_LIST,
  EXTERNAL_URL,
} from '@/constant';

import {
  getAuthPlatformSignInURL,
  getAuthPlatformSignInPayload,
} from '@/util/auth';

import {
  apiLoginUser,
  apiCheckIsUser,
  apiGetUserMinById,
} from '@/util/api/api';

import { checkUserNameValid } from '@/util/ValidationHelper';


import AuthCoreRegister from '~/components/AuthCore/Register';
import BaseDialogV2 from '~/components/dialogs/BaseDialogV2';
import SigninPortal from './AuthDialogContent/SignInPortal';
// import EmailSigninForm from './AuthDialogContent/SignInWithEmail';
import RegisterForm from './AuthDialogContent/Register';
import EthMixin from '~/components/EthMixin';
import experimentsMixin from '~/util/mixins/experiments';

import User from '@/util/User';

import LikeCoinLogo from '~/assets/logo/icon-plain.svg';

import { logTrackerEvent, logTimingEvent } from '@/util/EventLogger';
import {
  tryPostLoginRedirect,
  checkIsMobileClient,
} from '~/util/client';

function getRandomPaddedDigits(length) {
  return String(Math.floor(Math.random() * (10 ** length))).padStart(length, '0');
}

function shouldWriteURLIntoSession(sourceURL) {
  if (!sourceURL) {
    return false;
  }
  try {
    const url = new URL(sourceURL);
    return url.host.split('.').slice(-2).join('.') !== 'like.co';
  } catch (err) {
    return true;
  }
}

// TODO: remove this.$sentry.captureException

export default {
  name: 'auth-dialog',
  components: {
    BaseDialogV2,
    AuthCoreRegister,
    SigninPortal,
    RegisterForm,
  },
  mixins: [
    EthMixin,
    experimentsMixin(
      'shouldTestSocialStyle',
      'signin-portal',
      'list',
    ),
  ],
  data() {
    return {
      LikeCoinLogo,

      avatar: undefined,
      avatarHalo: 'none',
      fromDisplayName: '',

      currentTab: 'loading',
      contentStyle: {},
      tabTransition: 'fade',

      platform: '',
      signInPayload: {
        user: '',
        email: '',
        isEmailVerified: false,
      },

      errorCode: '',
      error: undefined,
      isSignIn: this.$route.query.register !== '1',

      referrer: '',
      sourceURL: '',
      loggedEvents: {},

      hasClickSignWithWalletInError: false,
      isUsingAuthCore: true,
      isMobileClient: false,
    };
  },
  computed: {
    ...mapGetters([
      'getAuthDialogStatus',
      'getIsShowAuthDialog',
      'getCurrentLocale',
      'getMetamaskError',
      'getLocalWeb3Wallet',
      'getUserMinInfoById',
      'getAuthCoreAccessToken',
      'getUserIsAuthCore',
    ]),
    closable() {
      return !(this.isBlocking || this.isSinglePage);
    },
    isSinglePage() {
      return this.$route.name === 'in-register';
    },
    isBlocking() {
      return this.currentTab === 'loading';
    },
    shouldHideDialog() {
      return !!this.getMetamaskError;
    },
    shouldShowDialog() {
      return this.getIsShowAuthDialog && !this.shouldHideDialog;
    },
    isEmailEditable() {
      return (
        !(this.signInPayload.isEmailVerified && this.signInPayload.email)
        && this.platform !== 'authcore'
      );
    },
    shouldShowHeader() {
      return this.currentTab !== 'portal' || this.isUsingAuthCore;
    },
    tabKey() {
      return this.currentTab;
    },
    tabProps() {
      return {
        ref: 'tab',
        key: this.tabKey,
      };
    },
    tabTransitionMode() {
      switch (this.tabTransition) {
        case 'flip':
          return 'out-in';

        case 'fade':
        default:
          return undefined;
      }
    },
    errorTitle() {
      switch (this.errorCode) {
        case 'USER_ALREADY_EXIST':
        case 'EMAIL_ALREADY_USED':
        case 'USER_WALLET_INVALID':
          return this.$t('AuthDialog.Register.error');
        default:
          return this.$t('General.label.error');
      }
    },
    errorMessage() {
      if (this.errorCode && this.$i18n.te(`Error.${this.errorCode}`, 'en')) {
        return this.$t(`Error.${this.errorCode}`);
      }
      if (this.error && typeof this.error.toString === 'function') {
        this.$t('AuthDialog.label.signInErrorWithDetails', {
          details: this.error.toString(),
        });
      }
      return this.$t('AuthDialog.label.signInError');
    },
    errorMessageForLegacyUser() {
      switch (this.errorCode) {
        case 'USER_ALREADY_EXIST':
        case 'EMAIL_ALREADY_USED':
        case 'USER_WALLET_INVALID':
          return `AuthDialog.Register.suggestWalletSignIn.${
            this.errorCode === 'EMAIL_ALREADY_USED' ? 'email' : 'id'
          }`;
        default:
          return '';
      }
    },
    errorConfirmTitle() {
      switch (this.errorCode) {
        case 'USER_ALREADY_EXIST':
        case 'EMAIL_ALREADY_USED':
        case 'USER_WALLET_INVALID':
          return this.$t('General.retry');
        default:
          return this.$t('General.button.confirm');
      }
    },
    getAuthCoreRedirectUrl() {
      let url = `${EXTERNAL_URL}/in/register?`;
      url += 'redirect_sign_in=1&sign_in_platform=authcore';
      const { redirect } = this.$route.query;
      if (redirect) {
        url += `&redirect=${encodeURIComponent(redirect)}`;
      }
      return url;
    },
    socialLoginPaneOption() {
      if (this.shouldTestSocialStyle) return 'list';
      return 'grid';
    },
  },
  watch: {
    getAuthDialogStatus: {
      handler({ isShow, isSignIn }) {
        if (isShow) {
          this.currentTab = 'portal';

          const { redirect_sign_in: isRedirectSignIn } = this.$route.query;
          if (isRedirectSignIn) {
            this.currentTab = 'loading';
          }

          this.$nextTick(this.updateContentHeightForCurrentTab);
        }
        if (isSignIn !== this.isSignIn && !isSignIn && !this.loggedEvents.swapRegisterTab) {
          this.loggedEvents.swapRegisterTab = 1;
          logTrackerEvent(this, 'RegFlow', 'SwapToRegisterTab', 'SwapToRegisterTab', 1);
        }
        this.isSignIn = isSignIn;

        // Sync dialog display with query string if not in single page
        if (!this.isSinglePage) {
          const query = { ...this.$route.query };
          if (isShow) {
            // Add show_login=1 in query string
            query.show_login = '1';
            if (isSignIn) {
              delete query.register;
            } else {
              query.register = '1';
            }
          } else {
            // Remove show_login and register in query string
            delete query.show_login;
            delete query.register;
          }
          this.$router.replace({ path: this.$route.path, query });
        }
        this.logShowAuthDialog(isShow);
      },
      deep: true,
    },
    currentTab(tab) {
      if (tab === 'register' && !this.loggedEvents.register) {
        this.loggedEvents.register = 1;
        logTrackerEvent(this, 'RegFlow', 'ShowRegisterForm', 'ShowRegisterForm', 1);
      }

      this.$nextTick(this.updateResizeObserverForCurrentTab);
    },
    shouldShowDialog(value) {
      if (value) {
        this.$nextTick(this.updateResizeObserverForCurrentTab);
      }
    },
  },
  async mounted() {
    this.isMobileClient = checkIsMobileClient();
    this.currentTab = 'portal';

    this.loggedEvents = {};

    const { from } = this.$route.query;
    if (from) {
      try {
        if (!this.getUserMinInfoById(from)) {
          await this.fetchUserMinInfo(from);
        }
        const userInfo = this.getUserMinInfoById(from);
        this.avatar = userInfo.avatar;
        this.fromDisplayName = userInfo.displayName;
        this.avatarHalo = User.getAvatarHaloType(userInfo);
      } catch (err) {
        // noop
      }
    }

    // Listen to onClickReturnButton event of MetaMaskDialog
    this.$root.$on('MetaMaskDialog.onClickReturnButton', () => {
      this.stopWeb3Polling();
      this.currentTab = 'portal';
    });

    // Initialize content height
    this.updateContentHeightForCurrentTab();

    // Show dialog when show_login set to true in query string
    if (this.$route.query.show_login === '1') {
      this.setAuthDialog({
        isShow: true,
        isSignIn: this.$route.query.register !== '1',
      });
    }

    // Handle redirect sign in
    const {
      redirect_sign_in: isRedirectSignIn,
      sign_in_platform: signInPlatform,
      ...query
    } = this.$route.query;
    if (isRedirectSignIn) {
      this.logRegisterEvent(this, 'RegFlow', 'LoginRedirectDone', 'LoginRedirectDone', 1);
      this.currentTab = 'loading';

      try {
        if (signInPlatform) {
          if (signInPlatform === 'authcore') {
            const { code } = this.$route.query;
            const payload = await this.fetchAuthCoreAccessTokenAndUser(code);
            await this.signInWithAuthCore(payload);
          } else {
            const { code, state } = this.$route.query;
            this.platform = signInPlatform;
            this.signInPayload = await getAuthPlatformSignInPayload(
              signInPlatform,
              { code, state },
            );
            if (this.signInPayload) this.login();
          }
        }
      } catch (err) {
        // eslint-disable-next-line no-console
        console.error(err);
        this.setError(err.code, err);
        if (this.$sentry) this.$sentry.captureException(err);
      }
      this.$router.replace({
        name: this.$route.name,
        query,
      });
    } else {
      this.logShowAuthDialog(this.getIsShowAuthDialog);
    }

    const { referrer } = this.$route.query;
    if (from) this.referrer = from;
    this.sourceURL = referrer || document.referrer;
    try {
      if (window.sessionStorage) {
        // store data when first view page
        // restore data when it is lost. eg redirect sign in
        if (this.sourceURL && shouldWriteURLIntoSession(this.sourceURL)) {
          window.sessionStorage.setItem('registerDialogSourceURL', this.sourceURL);
        } else {
          this.sourceURL = window.sessionStorage.getItem('registerDialogSourceURL');
        }
      }
    } catch (err) {
      // eslint-disable-next-line no-console
      console.error(err);
    }

    this.contentResizeObserver = new ResizeObserver(this.onObservingContentResize);
    this.updateResizeObserverForCurrentTab();
  },
  beforeDestroy() {
    if (this.contentResizeObserver) {
      this.contentResizeObserver.disconnect();
    }
  },
  methods: {
    ...mapActions([
      'newUser',
      'refreshUser',
      'fetchUserMinInfo',
      'doPostAuthRedirect',
      'setAuthDialog',
      'setAuthDialogShow',
      'toggleAuthDialogIsSignIn',
      'setWalletNoticeDialog',
      'openPopupDialog',
      'fetchAuthCoreAccessTokenAndUser',
      'fetchAuthCoreUser',
      'setAuthCoreToken',
      'initAuthCoreCosmosWallet',
      'fetchAuthCoreCosmosWallet',
      'authCoreLogoutUser',
    ]),
    setContentStyle({ height }) {
      const style = {
        height: `${height}px`,
      };
      if (!this.shouldShowHeader) {
        style.marginTop = 0;
      }
      this.contentStyle = style;
    },
    updateContentHeightForCurrentTab() {
      const elem = this.$refs.tab;
      if (elem) this.setContentStyle({ height: elem.offsetHeight });
    },
    updateResizeObserverForCurrentTab() {
      if (!this.contentResizeObserver) return;
      this.contentResizeObserver.disconnect();
      const elem = this.$refs.tab;
      if (elem) this.contentResizeObserver.observe(elem, { box: 'border-box' });
    },
    setError(code, error) {
      this.currentTab = 'error';
      this.errorCode = code;
      this.error = error;
    },

    getDialogContentContainerElem() {
      if (
        this.$refs.dialog
        && this.$refs.dialog.$el
        && typeof this.$refs.dialog.$el.querySelector === 'function'
      ) {
        return this.$refs.dialog.$el.querySelector('.base-dialog-v2__content-container');
      }
      return undefined;
    },
    getTabTransitionEffectAndTarget(el) {
      let target = el;
      let effect = this.tabTransition;
      if (effect === 'flip') {
        const containerEl = this.getDialogContentContainerElem();
        if (containerEl) {
          target = containerEl;
        } else {
          effect = 'fade';
        }
      }
      return { effect, target };
    },
    onTabLeave(el, onComplete) {
      const { effect, target } = this.getTabTransitionEffectAndTarget(el);
      switch (effect) {
        case 'flip': {
          this.$gsap.TweenLite.to(target, 0.5, {
            rotationY: 90,
            z: 500,
            ease: 'easeInPower2',
            onComplete,
          });
          break;
        }

        case 'fade':
        default:
          this.$gsap.TweenLite.to(target, 1, {
            opacity: 0,
            onComplete,
          });
          break;
      }
    },
    onTabBeforeEnter(el) {
      const { effect, target } = this.getTabTransitionEffectAndTarget(el);
      switch (effect) {
        case 'flip':
          this.$gsap.TweenLite.set(target, {
            visibility: 'hidden',
            z: 500,
            rotationY: -90,
          });
          break;
        case 'fade':
        default:
          this.$gsap.TweenLite.set(target, {
            visibility: 'hidden',
            opacity: 0,
          });
          break;
      }
    },
    onTabEnter(el, onComplete) {
      const { effect, target } = this.getTabTransitionEffectAndTarget(el);
      switch (effect) {
        case 'flip': {
          this.$gsap.TweenLite.to(target, 1, {
            rotationY: 0,
            z: 0,
            visibility: 'visible',
            ease: 'easeOutPower2',
            onComplete,
          });
          break;
        }

        case 'fade':
        default:
          this.$gsap.TweenLite.to(target, 1, {
            opacity: 1,
            visibility: 'visible',
          }, {
            onComplete,
          });
          break;
      }
      this.updateContentHeightForCurrentTab();
    },
    onClickSignWithWalletInError() {
      this.hasClickSignWithWalletInError = true;
      this.signInWithPlatform('wallet');
    },
    onDismissError() {
      switch (this.errorCode) {
        case 'USER_REGISTER_ERROR':
        case 'EMAIL_ALREADY_USED':
        case 'USER_ALREADY_EXIST':
        case 'USER_WALLET_INVALID':
          this.currentTab = 'register';
          return;

        case 'USE_AUTHCORE_LOGIN':
        case 'WALLET_REGISTER_DEPRECATED':
          this.isUsingAuthCore = true;
          this.currentTab = 'portal';
          return;

        default:
      }
      this.resetAuthCoreStatus();
      this.close();
    },
    onUpdateIsShow(isShow) {
      if (!this.shouldHideDialog && isShow !== this.getIsShowAuthDialog) {
        this.setIsShow(isShow);
      }
    },
    onObservingContentResize(entries) {
      entries.forEach((entry) => {
        const { height } = entry.contentRect;
        this.setContentStyle({ height });
      });
    },
    onClickUseLegacyButton() {
      this.isUsingAuthCore = false;
      this.signInWithPlatform('wallet');
    },
    onClickBackButton() {
      if (!this.isUsingAuthCore) {
        this.isUsingAuthCore = true;
        return;
      }
      switch (this.currentTab) {
        case 'portal':
          if (this.isSinglePage) {
            if (window.opener) {
              window.close();
            } else {
              this.$router.go(-1);
            }
          } else {
            this.onCancel();
          }
          break;

        default:
          this.currentTab = 'portal';
          break;
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
      this.resetAuthCoreStatus();
      this.$emit('closed');
    },
    setIsShow(isShow) {
      this.setAuthDialogShow(isShow);
    },
    close() {
      this.setIsShow(false);
      this.$nextTick(() => {
        if (this.isSinglePage) {
          if (window.opener) {
            window.close();
          } else {
            this.$router.push({ name: 'index' });
          }
        }
      });
    },
    resetAuthCoreStatus() {
      this.isUsingAuthCore = true; // HACK: reset authcore flag
      if (this.getAuthCoreAccessToken && !this.getUserIsAuthCore) {
        this.authCoreLogoutUser();
      }
    },
    onAuthCoreLoaded() {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreDialogLoaded', 'AuthCoreDialogLoaded', 1);
    },
    onAuthCoreLoginWidgetLoaded() {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreLoginWidgetLoaded', 'AuthCoreLoginWidgetLoaded', 1);
    },
    onAuthCoreRegisterStarted(method) {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreRegisterTry', `AuthCoreRegisterTry(${method})`, 1);
    },
    onAuthCoreOAuthStarted(method) {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreRegisterTry', `AuthCoreRegisterTry(${method})`, 1);
    },
    onAuthCoreLoginStarted(method) {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreLoginTry', `AuthCoreLoginTry(${method})`, 1);
    },
    onAuthCoreNavigation(page) {
      this.logRegisterEvent(this, 'RegFlow', `AuthCoreSwitchTo${page}`, `AuthCoreSwitchTo${page}`, 1);
      this.setAuthDialog({
        isShow: true,
        isSignIn: page === 'SignIn',
      });
    },
    async signInWithAuthCore({ accessToken, currentUser, idToken }) {
      this.logRegisterEvent(this, 'RegFlow', 'AuthCoreSignInSuccess', 'AuthCoreSignInSuccess', 1);
      await this.setAuthCoreToken(accessToken);
      this.currentTab = 'loading';
      this.platform = 'authcore';
      let user = currentUser;
      if (!user || !user.profileName) {
        try {
          user = await this.fetchAuthCoreUser();
        } catch (err) {
          console.error(err);
        }
      }
      const {
        primaryEmail: email,
        suggestedName: username,
      } = user;
      this.signInPayload = {
        email,
        username,
        accessToken,
        idToken,
      };
      this.login();
    },
    async signInWithPlatform(platform) {
      this.platform = platform;
      this.logRegisterEvent(this, 'RegFlow', 'LoginTry', `LoginTry(${platform})`, 1);

      switch (platform) {
        case 'wallet':
          this.currentTab = 'loading';
          this.setWalletNoticeDialog({
            isShow: true,
            cancelTitle: this.$t('WalletNoticeDialog.allSignInOptions'),
            onCancel: () => {
              this.currentTab = 'portal';
            },
            onConfirm: () => this.startWeb3AndCb(this.signInWithWallet),
          });
          return;
        default: {
          if (LOGIN_CONNECTION_LIST.includes(platform)) {
            const { url } = await getAuthPlatformSignInURL(platform);
            if (url) window.location.href = url;
            break;
          }
          // eslint-disable-next-line no-console
          console.error('platform default not exist');
          if (this.$sentry) {
            this.$sentry.captureException(new Error('platform default not exist'));
          }
          this.currentTab = 'error';
        }
      }
    },
    async signInWithWallet() {
      this.currentTab = 'loading';

      // Determine whether the wallet has registered
      try {
        await apiCheckIsUser(this.getLocalWeb3Wallet);
      } catch (err) {
        if (err.response) {
          if (err.response.status === 404) {
            this.setError('WALLET_REGISTER_DEPRECATED', err);
            return;
          }
        }

        // eslint-disable-next-line no-console
        console.error(err);
        if (this.$sentry) this.$sentry.captureException(err);
        this.setError((err.response || {}).data, err);
        return;
      }

      try {
        this.signInPayload = await User.signLogin(this.getLocalWeb3Wallet);
        this.login();
      } catch (err) {
        if (err.message.indexOf('Invalid "from" address') >= 0) {
          // User has logout MetaMask after EthHelper initialization
          this.currentTab = 'loading';
          this.startWeb3AndCb(this.signInWithWallet, true);
        } else if (err.message.indexOf('User denied message signature') >= 0) {
          // Return to login portal if user denied signing
          this.currentTab = 'portal';
        } else {
          // eslint-disable-next-line no-console
          console.error(err);
          if (this.$sentry) this.$sentry.captureException(err);
          this.setError(err.message, err);
        }
      }
    },
    async login() {
      this.currentTab = 'loading';
      try {
        this.logRegisterEvent(this, 'RegFlow', 'OAuthSuccess', `OAuthSuccess(${this.platform})`, 1);
        await apiLoginUser({
          locale: this.getCurrentLocale,
          platform: this.platform,
          ...this.signInPayload,
        });
        this.logRegisterEvent(this, 'RegFlow', 'LoginSuccessWhenRegister', 'LoginSuccessWhenRegister', 1);
        this.redirectAfterSignIn();
      } catch (err) {
        if (err.response) {
          if (err.response.status === 404) {
            if (this.isSignIn) logTrackerEvent(this, 'RegFlow', 'LoginRedirectToRegister', 'LoginRedirectToRegister', 1);
            this.preRegister();
            return;
          }
        }
        this.logRegisterEvent(this, 'RegFlow', 'LoginFail', 'LoginFail', 1);
        // eslint-disable-next-line no-console
        console.error(err);
        if (this.$sentry) this.$sentry.captureException(err);
        this.setError((err.response || {}).data, err);
      }
    },
    async preRegister() {
      logTrackerEvent(this, 'RegFlow', 'PreRegister', `PreRegister(${this.platform})`, 1);
      this.currentTab = 'loading';
      if (this.signInPayload.suggestedName || this.signInPayload.email) {
        const RANDOM_DIGIT_LENGTH = 5;
        const MAX_SUGGEST_TRY = 5;
        let { suggestedName } = this.signInPayload;
        if (!suggestedName) {
          [suggestedName] = this.signInPayload.email.split('@');
        }
        suggestedName = suggestedName.trim().toLowerCase().replace(/[^a-z0-9-_]/g, '-');
        suggestedName = suggestedName.substring(0, MAX_USER_ID_LENGTH - RANDOM_DIGIT_LENGTH);
        let isIDAvailable = false;
        let tries = 0;
        let tryName = suggestedName;
        if (suggestedName.length < MIN_USER_ID_LENGTH) {
          tryName = `${suggestedName}${getRandomPaddedDigits(RANDOM_DIGIT_LENGTH)}`;
        }
        while (!isIDAvailable && tries < MAX_SUGGEST_TRY) {
          try {
            await apiGetUserMinById(tryName); // eslint-disable-line no-await-in-loop
          } catch (err) {
            if (err.response) {
              if (err.response.status === 404) {
                isIDAvailable = true;
                break;
              }
            }
          }
          tryName = `${suggestedName}${getRandomPaddedDigits(RANDOM_DIGIT_LENGTH)}`;
          tries += 1;
        }
        if (isIDAvailable && tryName && checkUserNameValid(tryName)) {
          Vue.set(this.signInPayload, 'defaultLikeCoinId', tryName);
        }
      }
      if (this.platform === 'authcore') {
        await this.initAuthCoreCosmosWallet();
        const cosmosWallet = await this.fetchAuthCoreCosmosWallet();
        Vue.set(this.signInPayload, 'cosmosWallet', cosmosWallet);
      }

      this.currentTab = 'register';
    },
    async register(registerPayload) {
      this.currentTab = 'loading';

      const payload = {
        locale: this.getCurrentLocale,
        platform: this.platform,
        ...this.signInPayload,
        ...registerPayload,
      };
      this.signInPayload = payload;
      if (this.referrer) payload.referrer = this.referrer;
      if (this.sourceURL) payload.sourceURL = this.sourceURL;

      try {
        await this.newUser(payload);
        logTrackerEvent(
          this,
          'RegFlow',
          'RegistrationComplete',
          `RegistrationComplete(${this.platform})`,
          1,
        );
        this.redirectAfterSignIn();
      } catch (err) {
        let errCode;
        if (err.response && err.response.data) {
          switch (err.response.data) {
            case 'USER_ALREADY_EXIST':
            case 'EMAIL_ALREADY_USED':
            case 'USER_WALLET_INVALID':
            case 'FIREBASE_USER_DUPLICATED':
              errCode = err.response.data;
              break;

            default:
              // eslint-disable-next-line no-console
              console.error(err);
              if (this.$sentry) this.$sentry.captureException(err);
          }
        } else {
          // eslint-disable-next-line no-console
          console.error(err);
          if (this.$sentry) this.$sentry.captureException(err);
          errCode = 'USER_REGISTER_ERROR';
        }
        this.setError(errCode, err);
        logTrackerEvent(
          this,
          'RegFlow',
          'RegistrationFail',
          `RegistrationFail(${this.platform})`,
          1,
        );
      }
    },
    async redirectAfterSignIn() {
      this.currentTab = 'loading';
      await this.refreshUser();

      try {
        window.sessionStorage.removeItem('registerDialogSourceURL');
      } catch (err) {
        // no op
      }
      this.setIsShow(false);
      if (this.isSinglePage) {
        this.currentTab = 'loginSuccessful';
        this.$nextTick(() => {
          if (!tryPostLoginRedirect({ route: this.$route })) {
            // Normal case
            this.redirectToPreAuthPage();
          }
        });
      } else {
        this.$nextTick(() => { this.redirectToPreAuthPage(); });
      }
    },
    redirectToPreAuthPage() {
      const router = this.$router;
      const route = this.$route;
      this.doPostAuthRedirect({ router, route });

      // Remind a suspect legacy user to bind OAuth login
      if (this.hasClickSignWithWalletInError) {
        this.openPopupDialog({
          allowClose: true,
          header: this.$t('AuthDialog.Failure.BindSocialAccounts.header'),
          message: this.$t('AuthDialog.Failure.BindSocialAccounts.message'),
          cancelText: this.$t('General.button.cancel'),
          confirmText: this.$t('AuthDialog.Failure.BindSocialAccounts.confirm'),
          onConfirm: () => {
            this.$router.push({
              name: 'in-settings',
              hash: '#connect',
            });
          },
        });
      }
      // Reset register failure count
      this.hasClickSignWithWalletInError = false;
    },
    logRegisterEvent(...args) {
      /* TODO: implement conditional log on !this.isSignIn */
      return logTrackerEvent(...args);
    },
    logShowAuthDialog(isShow) {
      if (isShow && !this.loggedEvents.showAuthDialog) {
        let value = 1;
        if (window.performance) {
          value = Math.round(performance.now());
        }
        logTimingEvent(this, 'RegFlow', 'AuthDialogMountedTiming', 'AuthDialogMountedTiming', value);
        this.loggedEvents.showAuthDialog = 1;
        this.logRegisterEvent(this, 'RegFlow', 'ShowAuthDialog', 'ShowAuthDialog', 1);
      }
    },
  },
};
</script>


<style lang="scss" scoped>
@import '~assets/variables';
@import "~assets/mixin";

.lc-dialog {
  /deep/ .lc-dialog-header::before {
    background: linear-gradient(246deg, #d2f0f0, #f0e6b4);
  }
}

.auth-dialog {
  perspective: 4000px;
  perspective-origin: 50% 25%;

  /deep/ .lc-dialog-header {
    z-index: 1;
  }

  &--blocking {
    /deep/ .lc-dialog-header::before {
      @include background-image-sliding-animation-x(
        linear-gradient(to right, #d2f0f0, #f0e6b4 50%, #d2f0f0)
      );
    }
  }

  &__header-left {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;

    display: flex;
    align-items: center;

    padding-left: 16px;
  }

  &__header-center {
    padding-top: 16px;
  }

  &__header-right {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;

    display: flex;
    align-items: center;

    padding-right: 16px;
  }

  &__legacy-login-button {
    $login-button-color: $like-green;

    display: inline-block;

    padding: 4px 8px;

    text-decoration: none !important;

    color: $login-button-color !important;
    border: 1px solid darken($login-button-color, 2);
    border-bottom-width: 2px;
    border-radius: 8px;

    font-weight: bold;

    &:hover {
      background-color: transparentize($login-button-color, 0.9);
    }
  }

  &__login-button {
    padding: 3px 26px;

    border: solid 2px $like-green;
    border-radius: 18px;
  }

  &__logo {
    position: absolute;
    top: 16px;

    width: 96px;

    transform: translateX(-50%);

    > img:nth-child(2) {
      transition: opacity 0.25s ease;
    }
  }

  &__support-creator {
    position: absolute;
    top: 120px;
    right: 0;
    left: 0;

    padding: 0 24px;

    text-align: center;

    color: $gray-9b;

    font-size: 16px;
    font-weight: 600;
  }

  &__content {
    overflow: hidden;

    margin-top: 64px;

    transition-timing-function: ease;
    transition-duration: 1s;
    transition-property: margin-top, height;

    &--with-avatar {
      margin-top: 96px;
    }
  }

  &__tab-container {
    position: relative;
  }

  &__tab {
    position: absolute;

    width: 100%;
  }
}
</style>
