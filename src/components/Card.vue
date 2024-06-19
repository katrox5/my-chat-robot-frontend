<script setup lang="ts">
  import { marked } from 'marked'
  import { ref, onMounted, watchEffect } from 'vue'
  import { useMessage, NCard, NTabs, NTabPane, NInput, NSkeleton, NFloatButton, NIcon } from 'naive-ui'
  import { Reload } from '@vicons/ionicons5'
  import hljs from 'highlight.js'
  import 'highlight.js/styles/a11y-light.css'

  const emit = defineEmits(['response'])
  const props = defineProps({ content: String })

  const output = ref('')

  const loading = ref(true)
  const curTab = ref('提问')
  const swtchTab = (value: string) => curTab.value = value

  function regen() {
    loading.value = true
    useMessage().loading(
      '重新生成回答...', 
      { closable: true }
    )
    output.value = ''
    request()
  }

  function request() {
    fetch('http://localhost:5000/prompt', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        prompt: props.content
      }),
    }).then(async (resp) => {
      if (resp.body == null) return
      const reader = resp.body.getReader()
      const decoder = new TextDecoder()

      loading.value = false
      while (true) {
        const { done, value } = await reader.read()
        if (done) {
          const blocks = document.querySelectorAll('pre code')
          blocks.forEach((block) => hljs.highlightBlock(block as HTMLElement))
          break
        }
        const chunkText = decoder.decode(value)
        output.value += chunkText
      }
    })
  }

  watchEffect(() => {
    if (loading.value) {
      curTab.value = '回答'
    }
  })

  onMounted(() => request())
</script>

<template>
  <NCard content-style="padding-top: 0;" hoverable>
    <NTabs type="line" size="large" @update:value="swtchTab" :value="curTab" :tabs-padding=20>
      <NTabPane name="提问">
        <NInput type="textarea" readonly show-count :value="props.content" :autosize=true :resizable=false />
        <NFloatButton v-if="!loading" @click="regen" top=10 right=10 width=30 height=30 position="absolute">
          <NIcon><Reload /></NIcon>
        </NFloatButton>
      </NTabPane>
      <NTabPane name="回答">
        <NSkeleton v-if="loading" text :repeat=1 />
        <div v-else v-html="marked(output)" />
      </NTabPane>
    </NTabs>
  </NCard>
</template>