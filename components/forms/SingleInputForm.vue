<template>
  <form
    :id="getFormId"
    class="lc-form"
    @keydown.esc="onCancel"
    @submit.prevent="onSubmit"
  >

    <md-field :class="['md-likecoin', { 'md-invalid': !!errorText }]">
      <label>{{ label }}</label>
      <md-input
        v-model="inputText"
        :type="type"
        :pattern="pattern"
        :autocomplete="autocomplete"
        required
      />
    </md-field>
    <p
      v-if="errorText"
      class="md-error lc-font-size-12"
    >
      {{ errorText }}
    </p>

    <div class="lc-button-group">
      <md-button
        :form="getFormId"
        class="md-likecoin"
        type="submit"
      >
        {{ $t('General.button.confirm') }}
      </md-button>
      <br>
      <md-button
        :form="getFormId"
        class="md-likecoin lc-cancel"
        @click="onCancel"
      >
        {{ $t('General.button.cancel') }}
      </md-button>
    </div>

  </form>
</template>

<script>
export default {
  name: 'input-form',
  props: {
    formId: {
      type: String,
      default: undefined,
    },
    type: {
      type: String,
      default: '',
    },
    autocomplete: {
      type: String,
      default: undefined,
    },
    pattern: {
      type: String,
      default: undefined,
    },
    text: {
      type: String,
      default: '',
    },
    label: {
      type: String,
      default: '',
    },
    errorText: {
      type: String,
      default: '',
    },
  },
  data() {
    return {
      inputText: '',
    };
  },
  computed: {
    getFormId() {
      return this.formId || 'single-input-form';
    },
  },
  watch: {
    text(value) {
      this.inputText = value;
    },
  },
  created() {
    this.inputText = this.text;
  },
  methods: {
    onCancel() {
      this.$emit('cancel');
    },
    onSubmit() {
      this.$emit('submit', this.inputText);
    },
    getText() {
      return this.inputText;
    },
  },
};
</script>


<style lang="scss" scoped>
@import "~assets/variables";

.md-error {
  margin: -24px 0 24px;

  color: $like-red;
}
</style>
