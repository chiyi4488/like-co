<template>
  <footer>
    <div class="lc-container-1">
      <div class="lc-container-2">
        <div class="lc-content-wrapper">

          <span class="chain-address lc-mobile-hide">
            <a
              :href="BIGDIPPER_HOST"
              target="_blank"
              rel="noopener"
            >
              {{ $t('Footer.label.chain') }}
            </a>
          </span>

          <div class="right footer-links">
            <a href="https://docs.like.co/">{{ $t('Footer.button.developer') }}</a>
            <nuxt-link :to="{ name: 'in-policies' }">{{ $t('PoliciesPage.title') }}</nuxt-link>
            <a href="https://help.like.co/">{{ $t('Footer.button.support') }}</a>
          </div>

        </div>
      </div>
    </div>
  </footer>
</template>
<script>
import { setTrackerUserId } from '@/util/EventLogger';

import { mapGetters } from 'vuex';
import {
  BIGDIPPER_HOST,
  ETHERSCAN_HOST,
} from '@/constant';

export default {
  data() {
    return {
      BIGDIPPER_HOST,
    };
  },
  computed: {
    ...mapGetters([
      'getUserInfo',
      'getCurrentLocale',
      'getInfoMsg',
      'getInfoIsError',
      'getAuthCoreOAuthFactors',
      'getUserLikeCoinAmountInBigNumber',
    ]),
    getAddress() {
      return `${ETHERSCAN_HOST}/address/${this.contractAddress}`;
    },
  },
  watch: {
    async getUserInfo(e) {
      const {
        user,
        displayName,
        email,
        wallet,
        cosmosWallet,
        isAuthCore,
      } = e;
      if (window.$crisp) {
        // if (user) opt.user_id = user;
        const { $crisp } = window;
        if (displayName) $crisp.push(['set', 'user:nickname', [displayName]]);
        if (email) $crisp.push(['set', 'user:email', email]);
        const opt = [['LikeCoin', true]];
        if (wallet) {
          opt.push(['wallet', wallet]);
        }
        if (cosmosWallet) {
          opt.push(['cosmos_wallet', cosmosWallet]);
        }
        if (isAuthCore) {
          opt.push(['binded_authcore', true]);
        }
        window.$crisp.push(['set', 'session:data', [opt]]);
      }
      if (user) {
        if (this.$sentry) {
          const opt = {
            id: user,
            username: displayName,
          };
          this.$sentry.configureScope((scope) => {
            scope.setUser(opt);
          });
        }
        await setTrackerUserId(user);
      }
    },
    getCurrentLocale(language) {
      if (window.$crisp) {
        window.$crisp.push(['set', 'session:data', [[['language', language]]]]);
      }
    },
    getInfoMsg(message) {
      if (this.getInfoIsError && window.$crisp) {
        window.$crisp.push(['set', 'session:data', [[['lastError', message]]]]);
        window.$crisp.push(['set', 'session:event', [[['likecoin-store_error', message]]]]);
      }
    },
    getAuthCoreOAuthFactors(factors) {
      if (window.$crisp && factors) {
        const opt = [];
        const services = factors.map(f => f.service);
        services.forEach((service) => {
          // eslint-disable-next-line no-param-reassign
          if (service) opt.push([`binded_${service.toLowerCase()}`, true]);
        });
        window.$crisp.push(['set', 'session:data', [opt]]);
      }
    },
    getUserLikeCoinAmountInBigNumber(amount) {
      if (window.$crisp && amount) {
        window.$crisp.push(['set', 'session:data', [[['LIKE', amount.toFixed(4)]]]]);
      }
    },
  },
  async mounted() {
    const {
      user,
      displayName,
      email,
      wallet,
      cosmosWallet,
      isAuthCore,
    } = this.getUserInfo;
    if (window.$crisp) {
      // if (user) opt.user_id = user;
      const { $crisp } = window;
      const language = this.getCurrentLocale;
      if (displayName) $crisp.push(['set', 'user:nickname', [displayName]]);
      if (email) $crisp.push(['set', 'user:email', email]);
      const opt = [['LikeCoin', true], ['language', language]];
      if (wallet) {
        opt.push(['wallet', wallet]);
      }
      if (cosmosWallet) {
        opt.push(['cosmos_wallet', cosmosWallet]);
      }
      if (isAuthCore) {
        opt.push(['binded_authcore', true]);
      }
      const factors = this.getAuthCoreOAuthFactors || [];
      const services = factors.map(f => f.service);
      services.forEach((service) => {
        // eslint-disable-next-line no-param-reassign
        if (service) opt.push([`binded_${service.toLowerCase()}`, true]);
      });
      window.$crisp.push(['set', 'session:data', [opt]]);
    }
    if (user) {
      if (this.$sentry) {
        const opt = {
          id: user,
          username: displayName,
        };
        this.$sentry.configureScope((scope) => {
          scope.setUser(opt);
        });
      }
      await setTrackerUserId(user);
    }
  },
};
</script>

<style lang="scss" scoped>
@import "~assets/variables";

footer {
  position: relative;

  width: 100%;
  padding: 12px 0;

  border-top: 0;
  background-color: $like-white;

  // Gradient separator
  &::before {
    position: absolute;
    right: 0;
    bottom: 100%;
    left: 0;

    height: 2px;

    content: " ";

    background-image: linear-gradient(269deg, $like-light-blue, $like-gradient-1);
  }

  .lc-content-wrapper {
    display: flex;
    justify-content: space-between;
  }

  .contract-address {
    text-align: left;

    font-size: 10px;

    a {
      color: $like-green;
    }
  }

  .footer-links {
    display: flex;
    flex-wrap: wrap;

    margin-left: 24px;

    // Prevent overlapping with Intercom button
    @media (max-width: 1647px) {
      margin-right: 64px;
    }
    @media (max-width: 1024px) {
      margin-right: 108px;
    }
    @media (max-width: 768px) {
      flex-direction: column;
    }

    > a {
      display: block;

      text-align: left;

      color: #6B6B6B;

      font-size: 12px;
      font-weight: 400;

      &:not(:last-child) {
        margin-right: 24px;
      }
    }
  }
}
</style>
