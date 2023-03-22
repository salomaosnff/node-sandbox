<script setup lang="ts">
import * as monaco from 'monaco-editor'
import { onMounted, ref, watch } from 'vue';

const props = defineProps<{
  content?: string
  filename?: string
}>()

const emit = defineEmits<{
  (event: 'update:content', content: string): void
}>()

const el = ref<HTMLElement>()
let editor: monaco.editor.IStandaloneCodeEditor

onMounted(() => {
  // Add node.js types
  editor = monaco.editor.create(el.value!, {
    detectIndentation: true,
    theme: 'vs-dark',
    automaticLayout: true,
  })

  editor.addAction({
    id: 'save',
    label: 'Save',
    keybindings: [
      monaco.KeyMod.CtrlCmd | monaco.KeyCode.KeyS
    ],
    run: () => emit('update:content', editor.getValue())
  })

  updateFile(props.filename ?? '')
})

function updateFile(filename: string) {
  if (!editor) {
    return
  }

  const model = monaco.editor.createModel(props.content ?? '', undefined, monaco.Uri.parse(filename ?? ''))

  editor.setModel(model)
}

watch(() => props.filename, (file) => file && updateFile(file))
</script>

<template>
  <div class="editor" ref="el">
  </div>
</template>