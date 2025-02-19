<template>
  <textarea
    v-bind="{ value: modelValue, placeholder, maxlength: max }"
    :id="textareaId"
    ref="textarea"
    class="ui-textarea w-full ui-input rounded-lg px-4 py-2 transition bg-input"
    :class="{ 'overflow-hidden resize-none': autoresize }"
    :style="{ height }"
    @input="emitValue"
  ></textarea>
</template>
<script>
import { ref, onMounted } from 'vue';
import { useComponentId } from '@/composable/componentId';

export default {
  props: {
    modelValue: {
      type: String,
      default: '',
    },
    label: {
      type: String,
      default: '',
    },
    placeholder: {
      type: String,
      default: '',
    },
    autoresize: {
      type: Boolean,
      default: false,
    },
    height: {
      type: [Number, String],
      default: '',
    },
    max: {
      type: [Number, String],
      default: null,
    },
    block: Boolean,
  },
  emits: ['update:modelValue', 'change'],
  setup(props, { emit }) {
    const textareaId = useComponentId('textarea');
    const textarea = ref(null);

    function calcHeight() {
      if (!props.autoresize) return;

      textarea.value.style.height = 'auto';
      textarea.value.style.height = `${textarea.value.scrollHeight}px`;
    }
    function emitValue(event) {
      let { value } = event.target;
      const maxLength = Math.abs(props.max) || Infinity;

      if (value.length > maxLength) {
        value = value.slice(0, maxLength);
      }

      emit('update:modelValue', value);
      emit('change', value);
      calcHeight();
    }

    onMounted(calcHeight);

    return {
      textarea,
      emitValue,
      textareaId,
    };
  },
};
</script>
