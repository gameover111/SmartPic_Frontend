<script setup lang="ts">
import { computed } from 'vue'
import { cn } from '@/lib/utils'
import { Check } from 'lucide-vue-next'

interface Props {
  class?: string
  id?: string
  modelValue?: boolean
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  modelValue: false,
})

const emit = defineEmits<{
  'update:modelValue': [value: boolean]
}>()

const classes = computed(() =>
  cn(
    'peer h-4 w-4 shrink-0 rounded-sm border border-primary ring-offset-background focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50',
    props.class,
  ),
)

function toggle() {
  if (props.disabled) return
  emit('update:modelValue', !props.modelValue)
}
</script>

<template>
  <button
    type="button"
    role="checkbox"
    :aria-checked="modelValue"
    :class="classes"
    :id="id"
    :disabled="disabled"
    :data-state="modelValue ? 'checked' : 'unchecked'"
    @click="toggle"
  >
    <span v-if="modelValue" class="flex items-center justify-center text-current bg-primary rounded-sm">
      <Check class="h-4 w-4 text-primary-foreground" />
    </span>
  </button>
</template>
