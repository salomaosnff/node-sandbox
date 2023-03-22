<script setup lang="ts">
import type { WebContainerProcess } from '@webcontainer/api';
import { onMounted, ref, type PropType } from 'vue';
import { Terminal } from 'xterm'
import { FitAddon } from 'xterm-addon-fit';


const props = defineProps({
  process: {
    type: Object as PropType<WebContainerProcess>,
    required: true
  }
})

const emit = defineEmits<{
  (event: 'exit', code: number): void
}>()

const el = ref<HTMLDivElement>()
let terminal: Terminal

async function start() {
  props.process.exit.then((code) => emit('exit', code));

  props.process.output.pipeTo(
    new WritableStream({
      write(data) {
        terminal.write(data);
      },
    })
  );

  new ReadableStream({
    start(controller) {
      terminal.onData((data) => controller.enqueue(data));
    },
  }).pipeTo(props.process.input);
}

onMounted(async () => {
  terminal = new Terminal({
    theme: {
      background: '#181818'
    },
    convertEol: true,
  })

  const fitAddon = new FitAddon();

  terminal.loadAddon(fitAddon);

  terminal.open(el.value!)
  fitAddon.fit();

  terminal.onResize((size) => props.process.resize(size));  
  
  props.process.resize({
    cols: terminal.cols,
    rows: terminal.rows,
  })
  
  start()
})
</script>

<template>
  <div ref="el"></div>
</template>