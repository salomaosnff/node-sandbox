<script setup lang="ts">
import * as monaco from 'monaco-editor'
import { ref, onMounted, watchEffect, computed } from 'vue';
import Terminal from '../components/Terminal.vue';
import Editor from '../components/Editor.vue';
import { WebContainer, type WebContainerProcess, type FileSystemTree } from '@webcontainer/api'
import { useStorage } from '@vueuse/core'

let container: WebContainer
const el = ref<HTMLElement>()
const iframeUrl = ref('')
const status = ref('Iniciando...')

const files = useStorage('files', {
  'package.json': JSON.stringify({
    type: 'module',
    name: 'hello-world',
    version: '1.0.0',
    main: 'index.js',
    scripts: {
      start: 'nodemon src/index.js',
    },
    devDependencies: {
      'nodemon': '^2.0.7',
    },
  }),
  'src': {
    'index.js': 'console.log("Hello World")',
  }
}, localStorage, {
  deep: true
})

function fileTree(files: any): FileSystemTree {
  const tree: FileSystemTree = {}

  for (const [key, value] of Object.entries(files)) {
    if (typeof value === 'string') {
      tree[key] = {
        file: {
          contents: value,
        }
      }
    } else {
      tree[key] = {
        directory: fileTree(value),
      }
    }
  }

  return tree
}

async function boot() {
  container = await WebContainer.boot()

  container.on('server-ready', (port, url) => {
    iframeUrl.value = url
  })

  await container.mount(fileTree(files.value))
}

let ready = ref(false)

onMounted(async () => {
  await boot()

  ready.value = true;

  await run('yarn')
  await run('yarn', ['start'])
})

watchEffect(() => {
  if (!el.value || !ready.value) {
    return
  }

  const editor = monaco.editor.create(el.value!, {
    value: 'console.log("Hello World")',
    language: 'javascript',
    theme: 'vs-dark',
  })

  editor.addAction({
    id: 'save',
    label: 'Save',
    keybindings: [
      monaco.KeyMod.CtrlCmd | monaco.KeyCode.KeyS,
    ],
    run: async () => {
      await container.fs.writeFile('/index.js', editor.getValue())
    }
  })
})

const activeTerminal = ref('run')

interface TerminalProcess {
  id: string
  process: WebContainerProcess
  active(): void
}

const terminals = ref<Array<TerminalProcess>>([])

async function run(command: string = 'jsh', args: string[] = []) {
  const process = await container.spawn(command, args);

  const item: TerminalProcess = {
    id: Math.random().toString(36).substring(2, 9),
    process,
    active: () => {
      activeTerminal.value = item.id
    }
  }

  terminals.value.push(item)
  activeTerminal.value = item.id

  return process.exit;
}

const filename = ref('src/index.js')

const fileContent = computed({
  get: (): string => {
    const queue = filename.value.split('/');

    let current: any = files.value;

    while (queue.length) {
      const item = queue.shift();

      if (item) {
        current = current[item];
      }
    }

    if (typeof current === 'string') {
      return current;
    }

    return '';
  },
  set: (value: string) => {
    const queue = filename.value.split('/');

    let current: any = files.value;

    while (queue.length) {
      const item = queue.shift();

      if (item) {
        if (queue.length) {
          current = current[item];
        } else {
          current[item] = value;
        }
      }
    }

    container.fs.writeFile(filename.value, value)
  }
})
</script>

<template>
  <div class="app" v-if="ready">
    <div class="panels">
      <Editor :filename="filename" v-model:content="fileContent" />
      <div class="terminals-tabs">
        <button @click="run()">Novo terminal</button>

        <button v-for="item, index in terminals" :key="item.id" @click="item.active">
          Terminal {{ index + 1 }}
        </button>
      </div>
      <div class="terminals">
        <template v-for="item, index in terminals" :key="item.id">
          <Terminal v-show="item.id === activeTerminal" :process="item.process" @exit="terminals.splice(index, 1)" />
        </template>
      </div>
    </div>
    <div class="webview">
      <input v-model="iframeUrl" type="url">
      <iframe frameborder="no" :src="iframeUrl" />
    </div>
  </div>
  <div v-else>
    <p>{{ status }}</p>
  </div>
</template>

<style>
.app {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: row;
  overflow: hidden;
}

.panels {
  display: flex;
  flex-direction: column;
  flex: 1;
  overflow: hidden;
}

.webview {
  flex: 0 auto;
  width: 800px;
  display: flex;
  flex-direction: column;
  padding: 1px;
  gap: 0.5rem;
}

.webview iframe {
  flex: 1;
}

.webview input {
  background: none;
  color: white;
  border: 1px solid #333;
  padding: 0.5rem;
  outline: none;
}

.editor {
  flex: 1
}

.terminals {
  display: flex;
  flex-direction: row;
  gap: 1rem;
  overflow: hidden;
  height: 320px;
  flex: 0 auto;
}

.terminals-tabs {
  display: flex;
  padding: 1rem;
  gap: 1rem;
}

.terminals>* {
  flex: 1;
  padding: 1rem;
}
</style>