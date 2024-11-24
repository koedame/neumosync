<script setup lang="ts">
import { computed } from 'vue';

const props = defineProps<{
  modelValue: number;
  min: number;
  max: number;
  step: number;
}>();

const emit = defineEmits<{
  'update:modelValue': [value: number];
}>();

const sliderStyle = computed(() => {
  const progress = (props.modelValue / props.max) * 100;

  return `background: linear-gradient(to right, rgb(107 114 128) ${progress}%, rgba(0, 0, 0, 0) ${progress}%); box-shadow: inset 0.1em 0.1em calc(0.1em * 2) #A3AAB9, inset calc(0.1em * -1) calc(0.1em * -1) calc(0.1em * 2) #FFFFFF; `;
});

const updateValue = (event: Event) => {
  const target = event.target as HTMLInputElement;
  emit('update:modelValue', Number(target.value));
};

</script>

<template>
  <input :value="props.modelValue" class="range-input-gray" type="range" @input="updateValue" :min="props.min" :max="props.max" :step="props.step" :style="sliderStyle">
</template>
