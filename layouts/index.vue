<template>
  <div class="lc-layout">

    <tool-bars />

    <div class="lc-page-wrapper with-sliding-menu">

      <sliding-menu />

      <div class="lc-page-content">
        <nuxt />
      </div>

      <footer class="lc-page-footer lc-page-footer--no-margin">
        <div class="lc-container-0">
          <my-footer />
        </div>
      </footer>

    </div>

    <no-ssr>
      <portal-target name="light-box" />
    </no-ssr>

  </div>
</template>


<script>
import localeMixin from '~/mixins/locale';

import MyFooter from '~/components/footer/Footer';
import SlidingMenu from '~/components/SlidingMenu/index';
import ToolBars from '~/components/toolbars/ToolBars';

export default {
  components: {
    MyFooter,
    SlidingMenu,
    ToolBars,
  },
  mixins: [localeMixin],
};
</script>

<style lang="scss" scoped>

.social-links-container {
  position: fixed;
  z-index: 3;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;

  pointer-events: none;

  .lc-container-0,
  .lc-container-1 {
    height: 100%;
  }

  .platform-icon-bar-wrapper {
    position: relative;

    height: 100%;

    > div {
      position: absolute;
      top: 0;
      bottom: 0;
      left: 0;

      > div {
        display: flex;
        align-items: center;

        height: 100%;
        margin-left: 12px;

        @media (max-width: 768px) {
          margin-left: 4px;
        }
      }
    }

    .icon-bar {
      pointer-events: auto;
    }
  }
}
</style>
