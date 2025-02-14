<template>

  <!-- Accessibility properties for the overlay -->
  <transition name="modal-fade" appear>
    <div
      id="modal-window"
      ref="modal-overlay"
      class="modal-overlay"
      @keyup.esc.stop="emitCancelEvent"
      @keyup.enter="handleEnter"
    >
      <div
        ref="modal"
        class="modal"
        :tabindex="0"
        role="dialog"
        aria-labelledby="modal-title"
        :style="[ modalSizeStyles, { background: $themeTokens.surface } ]"
      >

        <!-- Modal Title -->
        <h1
          id="modal-title"
          ref="title"
          class="title"
        >
          {{ title }}
          <!-- Accessible error reporting per @radina -->
          <span
            v-if="hasError"
            class="visuallyhidden"
          >
            {{ errorMessage }}
          </span>
        </h1>

        <!-- Stop propagation of enter key to prevent the submit event from being emitted twice -->
        <form
          class="form"
          @submit.prevent="emitSubmitEvent"
          @keyup.enter.stop
        >
          <!-- Wrapper for main content -->
          <div
            ref="content"
            class="content"
            :style="[ contentSectionMaxHeight, scrollShadow ? {
              borderTop: `1px solid ${$themeTokens.fineLine}`,
              borderBottom: `1px solid ${$themeTokens.fineLine}`,
            } : {} ]"
            :class="{ 'scroll-shadow': scrollShadow }"
          >
            <!-- @slot Main content of modal -->
            <slot></slot>
          </div>

          <div
            ref="actions"
            class="actions"
          >
            <!-- @slot Alternative buttons and actions below main content -->
            <slot
              v-if="$slots.actions"
              name="actions"
            >
            </slot>
            <template v-else>
              <KButton
                v-if="cancelText"
                name="cancel"
                :text="cancelText"
                appearance="flat-button"
                :disabled="cancelDisabled || $attrs.disabled"
                @click="emitCancelEvent"
              />
              <KButton
                v-if="submitText"
                name="submit"
                :text="submitText"
                :primary="true"
                :disabled="submitDisabled || $attrs.disabled"
                type="submit"
              />
            </template>
          </div>
        </form>
      </div>
    </div>
  </transition>

</template>


<script>

  import log from 'loglevel';
  import debounce from 'lodash/debounce';
  import KResponsiveWindowMixin from './KResponsiveWindowMixin';

  const SIZE_SM = 'small';
  const SIZE_MD = 'medium';
  const SIZE_LG = 'large';
  const SIZE_STRINGS = [SIZE_SM, SIZE_MD, SIZE_LG];

  // check for Nuxt.js SSR
  const nuxtServerSideRendering = process && process.server;

  /**
   * Used to focus attention on a singular action/task
   */
  export default {
    name: 'KModal',
    mixins: [KResponsiveWindowMixin],
    props: {
      /**
       * The title of the modal
       */
      title: {
        type: String,
        required: true,
      },
      /**
       * If provided, text of the submit button
       */
      submitText: {
        type: String,
        default: null,
      },
      /**
       * If provided, text of the cancel button
       */
      cancelText: {
        type: String,
        default: null,
      },
      /**
       * Disable the submit button
       */
      submitDisabled: {
        type: Boolean,
        default: false,
      },
      /**
       * Disable the cancel button
       */
      cancelDisabled: {
        type: Boolean,
        default: false,
      },
      /**
       * Width of modal. For consistency use the strings `'small'`, `'medium'`, or `'large'`.
       * For more precise control use an integer number of pixels.
       */
      size: {
        type: [String, Number],
        default: 'medium',
        validator(val) {
          if (typeof val === 'string') {
            if (!SIZE_STRINGS.includes(val)) {
              log.error(`'${val}' is not one of: ${SIZE_STRINGS}`);
              return false;
            }
            return true;
          }
          return val > 0;
        },
      },
      /**
       * Toggles error message indicator in title
       */
      hasError: {
        type: Boolean,
        default: false,
      },
      errorMessage: {
        type: String,
        default: null,
        required: false,
      },
    },
    data() {
      return {
        lastFocus: null,
        maxContentHeight: '1000',
        contentHeight: 0,
        scrollShadow: false,
        delayedEnough: false,
      };
    },
    computed: {
      modalSizeStyles() {
        return {
          'max-width': `${this.maxModalWidth - 32}px`,
          'max-height': `${this.windowHeight - 32}px`,
          width: this.modalWidth,
        };
      },
      modalWidth() {
        if (this.size === SIZE_SM) return '300px';
        if (this.size === SIZE_MD) return '450px';
        if (this.size === SIZE_LG) return '100%';
        return `${this.size}px`;
      },
      maxModalWidth() {
        if (this.windowWidth < 1000) {
          return this.windowWidth;
        }
        return 1000;
      },
      contentSectionMaxHeight() {
        return {
          'max-height': `${this.maxContentHeight}px`,
          height: `${this.contentHeight}px`,
        };
      },
    },
    created() {
      if (this.$props.cancelText && !this.$listeners.cancel) {
        log.error(
          'A "cancelText" has been set, but there is no "cancel" listener. The "cancel" button may not work correctly.'
        );
      }
      if (this.$props.submitText && !this.$listeners.submit) {
        log.error(
          'A "submitText" has been set, but there is no "submit" listener. The "submit" button may not work correctly.'
        );
      }
    },
    beforeMount() {
      this.lastFocus = document.activeElement;
    },
    mounted() {
      if (nuxtServerSideRendering) {
        return;
      }
      // Remove scrollbars from the <html> tag, so user's can't scroll while modal is open
      window.document.documentElement.style['overflow'] = 'hidden';
      this.$nextTick(() => {
        if (this.$refs.modal && !this.$refs.modal.contains(document.activeElement)) {
          this.focusModal();
        }
      });
      window.addEventListener('focus', this.focusElementTest, true);
      window.setTimeout(() => (this.delayedEnough = true), 500);
    },
    updated() {
      this.updateContentSectionStyle();
    },
    destroyed() {
      if (nuxtServerSideRendering) {
        return;
      }
      // Restore scrollbars to <html> tag
      window.document.documentElement.style['overflow'] = '';
      window.removeEventListener('focus', this.focusElementTest, true);
      // Wait for events to finish propagating before changing the focus.
      // Otherwise the `lastFocus` item receives events such as 'enter'.
      window.setTimeout(() => this.lastFocus.focus());
    },
    methods: {
      /**
       * Calculate the max-height of the content section of the modal
       * If there is not enough vertical space, create a vertically scrollable area and a
       * scroll shadow
       */
      updateContentSectionStyle: debounce(function() {
        if (this.$refs.title && this.$refs.actions) {
          if (Math.abs(this.$refs.content.scrollHeight - this.contentHeight) >= 8) {
            this.contentHeight = this.$refs.content.scrollHeight;
          }
          const maxContentHeightCheck =
            this.windowHeight -
            this.$refs.title.clientHeight -
            this.$refs.actions.clientHeight -
            32;
          // to prevent max height from toggling between pixels
          // we set a threshold of how many pixels the height should change before we update
          if (Math.abs(maxContentHeightCheck - this.maxContentHeight) >= 8) {
            this.maxContentHeight = maxContentHeightCheck;
            this.scrollShadow = this.maxContentHeight < this.$refs.content.scrollHeight;
          }

          // make sure that overflow-y won't be updated to 'auto' if this
          // function is running for the first time
          // (otherwise Firefox would add a vertical scrollbar right away)
          if (this.$refs.content.clientHeight !== 0) {
            // add a vertical scrollbar if content doesn't fit
            if (this.$refs.content.scrollHeight > this.$refs.content.clientHeight) {
              this.$refs.content.style.overflowY = 'auto';
            }
          }
        }
      }, 50),
      emitCancelEvent() {
        if (!this.cancelDisabled) {
          /**
           * Emitted when the cancel button is clicked or the esc key is pressed
           */
          this.$emit('cancel');
        }
      },
      emitSubmitEvent() {
        if (!this.submitDisabled) {
          /**
           * Emitted when the submit button or the enter key is pressed
           */
          this.$emit('submit');
        }
      },
      handleEnter() {
        if (this.delayedEnough) {
          this.emitSubmitEvent();
        }
      },
      focusModal() {
        this.$refs.modal.focus();
      },
      focusElementTest(event) {
        const { target } = event;
        const noopOnFocus =
          target === window || // switching apps
          !this.$refs.modal || // if $refs.modal isn't available
          target === this.$refs.modal || // addresses #3824
          this.$refs.modal.contains(target.activeElement);
        if (noopOnFocus) {
          return;
        }
        // Fixes possible infinite recursion when disconnection snackbars appear
        // along with KModal (#6301)
        const $coreSnackbar = document.getElementById('coresnackbar');
        if ($coreSnackbar && $coreSnackbar.contains(target)) {
          return;
        }
        // focus has escaped the modal - put it back!
        if (!this.$refs.modal.contains(target)) {
          this.focusModal();
        }
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import './styles/definitions';

  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 24;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.7);
    background-attachment: fixed;
    transition: opacity $core-time ease;
  }

  // TODO: margins for stacked buttons.
  .modal {
    @extend %dropshadow-16dp;

    position: absolute;
    top: 50%;
    left: 50%;
    margin: 0 auto;
    overflow-y: auto;
    border-radius: $radius;
    transform: translate(-50%, -50%);

    &:focus {
      outline: none;
    }
  }

  .form {
    @extend %momentum-scroll;
  }

  .modal-fade-enter-active,
  .modal-fade-leave-active {
    transition: all $core-time ease;
  }

  .modal-fade-enter,
  .modal-fade-leave-active {
    opacity: 0;
  }

  .title {
    padding: 24px;
    margin: 0;
    font-size: 24px;
  }

  .content {
    padding: 0 24px;
    overflow-x: hidden;
  }

  .scroll-shadow {
    background: linear-gradient(white 30%, hsla(0, 0%, 100%, 0)),
      linear-gradient(hsla(0, 0%, 100%, 0) 10px, white 70%) bottom,
      radial-gradient(at top, rgba(0, 0, 0, 0.2), transparent 70%),
      radial-gradient(at bottom, rgba(0, 0, 0, 0.2), transparent 70%) bottom;
    background-repeat: no-repeat;
    background-attachment: local, local, scroll, scroll;
    background-size: 100% 20px, 100% 20px, 100% 10px, 100% 10px;
  }

  .actions {
    padding: 24px;
    text-align: right;
    button {
      margin: 0;
    }
  }

  .actions button:last-of-type {
    margin-left: 16px;
  }

</style>
