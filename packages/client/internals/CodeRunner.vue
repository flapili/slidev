<script setup lang="ts">
import { debounce, toArray } from '@antfu/utils'
import { onClickOutside, useVModel } from '@vueuse/core'
import type { CodeRunnerOutput, RawAtValue } from '@slidev/types'
import { computed, onMounted, onUnmounted, ref, shallowRef, watch, watchSyncEffect } from 'vue'
import { useSlideContext } from '../context'
import setupCodeRunners from '../setup/code-runners'
import { useNav } from '../composables/useNav'
import { makeId } from '../logic/utils'
import IconButton from './IconButton.vue'
import DomElement from './DomElement.vue'

const props = defineProps<{
  modelValue: string
  lang: string
  autorun: boolean | 'once'
  height?: string
  showOutputAt?: RawAtValue
  highlightOutput: boolean
  runnerOptions?: Record<string, unknown>
}>()

const emit = defineEmits(['update:modelValue'])

const { isPrintMode } = useNav()

const code = useVModel(props, 'modelValue', emit)

const { $renderContext, $clicksContext } = useSlideContext()
const disabled = computed(() => !['slide', 'presenter'].includes($renderContext.value))

const autorun = isPrintMode.value ? 'once' : props.autorun
const isRunning = ref(autorun)
const outputs = shallowRef<CodeRunnerOutput[]>()
const runCount = ref(0)
const highlightFn = ref<(code: string, lang: string) => string>()

const isStream = ref(false)
const modal = ref(false)
const modalElement = ref<HTMLDivElement>()
const streamResults = ref<{ data: string, type: 'stdout' | 'stderr' }[]>([])
const streamExitCode = ref<number>()

onClickOutside(modalElement, () => {
  modal.value = false
})

const hidden = ref(props.showOutputAt)
if (props.showOutputAt) {
  const id = makeId()
  onMounted(() => {
    const info = $clicksContext.calculate(props.showOutputAt)
    if (info) {
      $clicksContext.register(id, info)
      watchSyncEffect(() => {
        hidden.value = !info.isActive.value
      })
    }
    else {
      hidden.value = false
    }
  })
  onUnmounted(() => $clicksContext.unregister(id))
}

const triggerRun = debounce(200, async () => {
  if (disabled.value)
    return

  if (isRunning.value)
    return

  const { highlight, run } = await setupCodeRunners()
  highlightFn.value = highlight

  outputs.value = []

  const setAsRunning = setTimeout(() => {
    isRunning.value = true
  }, 500)

  isStream.value = false
  const result = await run(code.value, props.lang, props.runnerOptions ?? {})

  if (result instanceof ReadableStream) {
    streamResults.value = []
    streamExitCode.value = undefined
    isStream.value = true
    const reader = result.getReader()
    while (true) {
      const { done, value } = await reader.read()
      if (done)
        break

      if (value.type === 'exit')
        streamExitCode.value = value.code

      else
        streamResults.value.push(value)
    }
  }
  else {
    outputs.value = toArray(result)
  }

  runCount.value += 1
  isRunning.value = false

  clearTimeout(setAsRunning)
})

if (autorun === 'once')
  triggerRun()
else if (autorun)
  watch(code, triggerRun, { immediate: true })
</script>

<template>
  <div
    v-show="!hidden"
    class="relative flex flex-col rounded-b border-t border-main"
    :style="{ height: props.height }"
    data-waitfor=".slidev-runner-output"
  >
    <div v-if="modal" class="z-500 fixed inset-0 flex">
      <div ref="modalElement" class="flex-grow rounded-md m-12 border-white border-2 bg-main p-8 overflow-auto">
        <div v-if="streamExitCode !== undefined">
          Exit code: {{ streamExitCode }}
        </div>
        <div v-else>
          running ...
        </div>
        <pre
          v-for="(chunk, index) in streamResults"
          :key="`run-${runCount}-stream-${index}`"
          :class="{
            'text-red-500': chunk.type === 'stderr',
          }"
        >{{ chunk.data }}</pre>
      </div>
    </div>
  </div>
  <div v-if="disabled" class="text-sm text-center opacity-50">
    Code is disabled in the "{{ $renderContext }}" mode
  </div>

  <div v-else-if="isStream" class="text-sm text-center opacity-50">
    <div>
      <IconButton class="w-8 h-8 max-h-full flex justify-center items-center" title="Open logs" @click="modal = true">
        <carbon:text-long-paragraph />
      </IconButton>
    </div>
  </div>
  <div v-else-if="isRunning" class="text-sm text-center opacity-50">
    Running...
  </div>
  <div v-else-if="!outputs?.length" class="text-sm text-center opacity-50">
    Click the play button to run the code
  </div>
  <div v-else :key="`run-${runCount}`" class="slidev-runner-output">
    <template v-for="output, _idx1 of outputs" :key="_idx1">
      <div v-if="'html' in output" v-html="output.html" />
      <div v-else-if="'error' in output" class="text-red-500">
        {{ output.error }}
      </div>
      <DomElement v-else-if="'element' in output" :element="output.element" />
      <div v-else class="output-line">
        <template
          v-for="item, idx2 in toArray(output)"
          :key="idx2"
        >
          <span
            v-if="item.highlightLang && highlightFn"
            class="highlighted"
            v-html="highlightFn(item.text, item.highlightLang)"
          />
          <span v-else :class="item.class">{{ item.text }}</span>
          <span v-if="idx2 < toArray(output).length - 1" class="separator">,</span>
        </template>
      </div>
    </template>
  </div>
  <div v-if="code.trim()" class="absolute right-1 top-1 max-h-full flex gap-1">
    <IconButton class="w-8 h-8 max-h-full flex justify-center items-center" title="Show logs" @click="triggerRun">
      <carbon:play />
    </IconButton>
  </div>
</template>

<style lang="postcss">
.slidev-runner-output {
  @apply px-5 py-3 flex-grow text-xs leading-[.8rem] font-$slidev-code-font-family select-text;
}

.slidev-runner-output .log-type {
  @apply font-bold op-70;

  &.DBG {
    @apply text-gray-500;
  }

  &.LOG {
    @apply text-blue-500;
  }

  &.WRN {
    @apply text-orange-500;
  }

  &.ERR {
    @apply text-red-500;
  }
}

.slidev-runner-output .output-line {
  @apply flex my-1 w-full;
}

.slidev-runner-output .separator {
  @apply op-40 mr-1;
}

.slidev-runner-output .highlighted > pre {
  @apply inline text-wrap !bg-transparent;
}
</style>
