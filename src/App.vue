<script lang="ts" setup>
import { getVersion } from '@tauri-apps/api/app'
import InputRange from './components/InputRange.vue';
import { ref, onMounted, onUnmounted, computed } from 'vue';

const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
let timerWorker: Worker;
const isPlaying = ref(false);
const currentTwelveletNote = ref(0);
const tempo = ref(120);
const masterVolume = ref(0.5);
const accentVolume = ref(1);
const quarterVolume = ref(0.75);
const eighthVolume = ref(0);
const sixteenthVolume = ref(0);
const tripletVolume = ref(0);
const lookahead = ref(25.0); // milliseconds
const scheduleAheadTime = ref(0.1); // seconds
const nextNoteTime = ref(0.0);
const noteLength = ref(0.05); // seconds
const notesInQueue: { note: number, time: number }[] = [];

const beats = ref<number>(4);
const maxBeats = computed(() => beats.value * 12);

const nextTwelvelet = () => {
  const secondsPerBeat = 60.0 / tempo.value;
  nextNoteTime.value += 0.08333 * secondsPerBeat;
  currentTwelveletNote.value++;
  if (currentTwelveletNote.value === maxBeats.value) {
    currentTwelveletNote.value = 0;
  }
};

const calcVolume = (beatVolume: number) => {
  return beatVolume * masterVolume.value;
};

const scheduleNote = (beatNumber: number, time: number) => {
  notesInQueue.push({ note: beatNumber, time });

  const osc = audioContext.createOscillator();
  const gainNode = audioContext.createGain();
  osc.connect(gainNode);
  gainNode.connect(audioContext.destination);

  if (beatNumber % maxBeats.value === 0) {
    osc.frequency.value = 880.0;
    gainNode.gain.value = calcVolume(accentVolume.value);
  } else if (beatNumber % 12 === 0) {
    osc.frequency.value = 440.0;
    gainNode.gain.value = calcVolume(quarterVolume.value);
  } else if (beatNumber % 6 === 0) {
    osc.frequency.value = 440.0;
    gainNode.gain.value = calcVolume(eighthVolume.value);
  } else if (beatNumber % 4 === 0) {
    osc.frequency.value = 300.0;
    gainNode.gain.value = calcVolume(tripletVolume.value);
  } else if (beatNumber % 3 === 0) {
    osc.frequency.value = 220.0;
    gainNode.gain.value = calcVolume(sixteenthVolume.value);
  } else {
    gainNode.gain.value = 0;
  }

  osc.start(time);
  osc.stop(time + noteLength.value);
};

const scheduler = () => {
  while (nextNoteTime.value < audioContext.currentTime + scheduleAheadTime.value) {
    scheduleNote(currentTwelveletNote.value, nextNoteTime.value);
    nextTwelvelet();
  }
};

const togglePlay = () => {
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) {
    currentTwelveletNote.value = 0;
    nextNoteTime.value = audioContext.currentTime;
    timerWorker.postMessage('start');
  } else {
    timerWorker.postMessage('stop');
  }
};

const version = ref<string>('');

onMounted(async () => {
  version.value = await getVersion();
  timerWorker = new Worker(new URL('./worker/main.ts', import.meta.url));
  timerWorker.onmessage = (e) => {
    if (e.data === 'tick') {
      scheduler();
    }
  };
  timerWorker.postMessage({ interval: lookahead.value });
});

onUnmounted(() => {
  timerWorker.terminate();
});
</script>

<template>
  <div class="bg-gray-200 px-20 py-16 h-[100vh] overflow-hidden">
    <div class="flex items-center justify-center space-x-20">
      <div class="w-50 space-y-4">
        <div class="flex items-center justify-between space-x-4">
          <div class="flex items-center justify-between space-x-2">
            <svg class="w-8 w-8" fill="none" viewBox="0 0 170 170" width="170" xmlns="http://www.w3.org/2000/svg"><path d="m78.328 91.1333v-66.1333z" fill="#374151"/><g stroke="#374151"><path d="m78.328 91.1333v-66.1333" stroke-linejoin="bevel" stroke-width="2.48"/><path d="m78.539 91.6293-41.8562 51.9147h-8.6458l41.8563-51.9147z" fill="#374151" stroke-width=".992001"/><path d="m107 64h35" stroke-width="8"/><path d="m107 98h35" stroke-width="8"/></g></svg>
            <input :readonly="isPlaying" v-model.number="tempo" type="number" class="outline-none nm-inset-gray-200-sm text-gray-600 px-2 py-1 rounded-full text-center w-20" min="30" max="300">
          </div>
          <div class="flex items-center justify-between">
            <svg class="w-8 w-8" fill="none" viewBox="0 0 170 170" xmlns="http://www.w3.org/2000/svg"><g fill="#374151"><path d="m83.2448 73.2992v7.3408h-8.2336c-.6944 0-1.3888.5952-1.3888 1.3889v1.8848c0 .7936.6944 1.3887 1.3888 1.3887h30.0578c.793 0 1.389-.5951 1.389-1.3887v-1.8848c0-.7937-.596-1.3889-1.389-1.3889h-8.829v-7.3408h8.829c.793 0 1.389-.6944 1.389-1.488v-1.8848c0-.6944-.596-1.3888-1.389-1.3888h-8.829v-15.7728c0-.5952-.2976-1.0912-.7936-1.2896l-1.1904-.5952c-.1984-.0992-.496-.1984-.6944-.1984-.2976.0992-.6944.1984-.8928.3968l-9.0272 8.2336c-.2976.2976-.3968.6944-.3968 1.0912v8.1344h-11.408s11.0112-11.904 22.7168-27.5776c.3968-.496.496-.992.496-1.3888 0-.496-.1984-.8928-.2976-.992l-2.1824-2.1824c-.1984-.1984-.5952-.3968-.992-.3968-.5952 0-13.9872 0-14.6816 0-.7936 0-1.2896.5952-1.3888 1.1904 0 0-.496 7.3408-3.0752 15.376s-5.5552 12.9952-8.1344 16.6656c0 0-.2976.496-.2976 1.0912 0 .1984 0 .3968.0992.5952.3968.6944.992 1.7856.992 1.7856s.1984.5952 1.0912.5952z"/><path d="m83.2448 122.899v7.341h-8.2336c-.6944 0-1.3888.595-1.3888 1.389v1.885c0 .793.6944 1.389 1.3888 1.389h30.0578c.793 0 1.389-.596 1.389-1.389v-1.885c0-.794-.596-1.389-1.389-1.389h-8.829v-7.341h8.829c.793 0 1.389-.694 1.389-1.488v-1.884c0-.695-.596-1.389-1.389-1.389h-8.829v-15.773c0-.595-.2976-1.091-.7936-1.29l-1.1904-.595c-.1984-.099-.496-.198-.6944-.198-.2976.099-.6944.198-.8928.397l-9.0272 8.233c-.2976.298-.3968.695-.3968 1.091v8.135h-11.408s11.0112-11.904 22.7168-27.5779c.3968-.496.496-.992.496-1.3888 0-.496-.1984-.8928-.2976-.992l-2.1824-2.1824c-.1984-.1984-.5952-.3968-.992-.3968-.5952 0-13.9872 0-14.6816 0-.7936 0-1.2896.5952-1.3888 1.1904 0 0-.496 7.3408-3.0752 15.3765-2.5792 8.035-5.5552 12.995-8.1344 16.665 0 0-.2976.496-.2976 1.091 0 .199 0 .397.0992.596.3968.694.992 1.785.992 1.785s.1984.595 1.0912.595z"/></g></svg>
            <input :readonly="isPlaying" class="outline-none nm-inset-gray-200-sm text-gray-600 px-2 py-1 rounded-full text-center w-20" v-model.number="beats" type="number" min="1" max="300">
          </div>
        </div>

        <div class="bg-gray-300 h-[1px] w-full" />

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center text-gray-700">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="w-4 h-4">
              <path d="M13.5 4.06c0-1.336-1.616-2.005-2.56-1.06l-4.5 4.5H4.508c-1.141 0-2.318.664-2.66 1.905A9.76 9.76 0 0 0 1.5 12c0 .898.121 1.768.35 2.595.341 1.24 1.518 1.905 2.659 1.905h1.93l4.5 4.5c.945.945 2.561.276 2.561-1.06V4.06ZM18.584 5.106a.75.75 0 0 1 1.06 0c3.808 3.807 3.808 9.98 0 13.788a.75.75 0 0 1-1.06-1.06 8.25 8.25 0 0 0 0-11.668.75.75 0 0 1 0-1.06Z" />
              <path d="M15.932 7.757a.75.75 0 0 1 1.061 0 6 6 0 0 1 0 8.486.75.75 0 0 1-1.06-1.061 4.5 4.5 0 0 0 0-6.364.75.75 0 0 1 0-1.06Z" />
            </svg>
          </span>
          <InputRange v-model.number="masterVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center text-gray-700">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="w-4 h-4">
              <path fill-rule="evenodd" d="M5.636 4.575a.75.75 0 0 1 0 1.061 9 9 0 0 0 0 12.728.75.75 0 1 1-1.06 1.06c-4.101-4.1-4.101-10.748 0-14.849a.75.75 0 0 1 1.06 0Zm12.728 0a.75.75 0 0 1 1.06 0c4.101 4.1 4.101 10.75 0 14.85a.75.75 0 1 1-1.06-1.061 9 9 0 0 0 0-12.728.75.75 0 0 1 0-1.06ZM7.757 6.697a.75.75 0 0 1 0 1.06 6 6 0 0 0 0 8.486.75.75 0 0 1-1.06 1.06 7.5 7.5 0 0 1 0-10.606.75.75 0 0 1 1.06 0Zm8.486 0a.75.75 0 0 1 1.06 0 7.5 7.5 0 0 1 0 10.606.75.75 0 0 1-1.06-1.06 6 6 0 0 0 0-8.486.75.75 0 0 1 0-1.06ZM9.879 8.818a.75.75 0 0 1 0 1.06 3 3 0 0 0 0 4.243.75.75 0 1 1-1.061 1.061 4.5 4.5 0 0 1 0-6.364.75.75 0 0 1 1.06 0Zm4.242 0a.75.75 0 0 1 1.061 0 4.5 4.5 0 0 1 0 6.364.75.75 0 0 1-1.06-1.06 3 3 0 0 0 0-4.243.75.75 0 0 1 0-1.061ZM10.875 12a1.125 1.125 0 1 1 2.25 0 1.125 1.125 0 0 1-2.25 0Z" clip-rule="evenodd" />
            </svg>
          </span>
          <InputRange v-model.number="accentVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center">
            <svg fill="none" viewBox="0 0 320 170" class="w-10" xmlns="http://www.w3.org/2000/svg"><path d="m185.328 91.1333v-66.1333" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><path d="m176.656 91.1332-42.656 52.9068h9.92l42.656-52.9068z" fill="#374151"/></svg>
          </span>
          <InputRange v-model.number="quarterVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center">
            <svg fill="none" viewBox="0 0 320 170" class="w-10" xmlns="http://www.w3.org/2000/svg"><path d="m200.052 92.4031v-65.9221" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><g fill="#374151"><path d="m198.812 50.2894c0 .3175.099.9523.794 1.164 4.265 1.4813 12.796 7.1953 19.244 18.9406 1.786 3.2802 4.663 7.8302 4.663 16.6127 0 7.6185-1.786 15.4483-4.464 23.0673-.199.635-.298 1.164-.298 1.587 0 .741.397 1.058 1.091 1.058.596 0 1.092-.423 1.488-1.164 4.464-8.465 6.151-18.3053 6.151-27.9344-.496-13.6499-7.837-23.7022-7.837-23.7022.298 0-9.126-13.015-11.805-17.4592-3.67-6.0313-5.357-12.0627-5.555-12.486-.099-.3174-.992-4.1267-.992-4.1267-.099-.4232-.595-.8465-1.19-.8465-.695 0-1.29.6349-1.29 1.3756z"/><path d="m191.379 92.4031-42.656 52.9069h9.92l42.656-52.9069z"/><path d="m119.486 70.923c-.099 0-.297-.1059-.396-.1059-.596 0-1.092.3175-1.29.8465-2.282 4.8675-7.043 11.4279-12.4 12.486 1.091-1.3755 1.786-3.0686 1.786-5.079 0-4.1267-3.175-7.5128-7.044-7.5128-3.9676 0-7.142 3.3861-7.142 7.5128 0 5.7139 5.2576 8.3592 9.92 8.3592 5.258 0 8.73-2.6453 12.102-6.5604l-11.308 37.6696 2.678.847 13.888-46.6642c.099-.1058.099-.3175.099-.5291 0-.5291-.297-.9523-.893-1.2697z"/></g></svg>
          </span>
          <InputRange v-model.number="eighthVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center">
            <svg fill="none" viewBox="0 0 320 170" class="w-10" xmlns="http://www.w3.org/2000/svg"><path d="m127.761 91.4031v-66.0277" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><path d="m264.351 91.4031v-66.0277" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><g fill="#374151"><path d="m126.521 65.5847c0 .3174.793.7407 1.488 1.0581 2.579.9523 10.317 6.243 16.269 15.6604 3.372 5.3965 4.166 9.2058 4.166 12.8034 0 .6349 0 1.164-.099 1.7988 0 2.5396-.695 7.9356-2.183 12.1686-.099.529-.396 1.164-.396 1.693 0 .423.198.847.694 1.164.198.106.397.106.496.106.694 0 1.091-.847 1.488-1.482 1.786-2.962 4.365-8.676 4.365-14.6019 0-3.0686-.199-5.6081-.496-7.8302 1.091-3.9151 1.885-7.8302 1.885-13.015 0-5.5023-1.389-10.4755-4.762-15.2371-4.662-6.5605-9.722-11.957-13.491-17.8825-3.77-5.8198-6.547-16.7185-6.547-16.7185-.199-.7407-.298-1.2698-1.389-1.2698-1.191 0-1.488.5291-1.488 1.2698zm20.733 9.7348c-2.679-3.7035-5.655-7.0895-10.516-12.2743-3.868-4.1268-5.654-7.7244-6.249-10.4756-.199-.529-.298-1.2697-.397-2.1162 5.059.1058 11.507 7.3011 15.078 12.2743 3.869 5.3965 4.762 9.7349 4.762 13.6499 0 1.0582-.099 2.1163-.198 3.1745-.794-1.6931-1.687-3.0686-2.48-4.2326z"/><path d="m263.111 65.5847c0 .3174.793.7407 1.488 1.0581 2.579.9523 10.317 6.243 16.269 15.6604 3.372 5.3965 4.166 9.2058 4.166 12.8034 0 .6349 0 1.164-.099 1.7988 0 2.5396-.695 7.9356-2.183 12.1686-.099.529-.396 1.164-.396 1.693 0 .423.198.847.694 1.164.198.106.397.106.496.106.694 0 1.091-.847 1.488-1.482 1.786-2.962 4.365-8.676 4.365-14.6019 0-3.0686-.199-5.6081-.496-7.8302 1.091-3.9151 1.885-7.8302 1.885-13.015 0-5.5023-1.389-10.4755-4.762-15.2371-4.662-6.5605-9.722-11.957-13.491-17.8825-3.77-5.8198-6.547-16.7185-6.547-16.7185-.199-.7407-.298-1.2698-1.389-1.2698-1.191 0-1.488.5291-1.488 1.2698zm20.733 9.7348c-2.679-3.7035-5.655-7.0895-10.516-12.2743-3.868-4.1268-5.654-7.7244-6.249-10.4756-.199-.529-.298-1.2697-.397-2.1162 5.059.1058 11.507 7.3011 15.078 12.2743 3.869 5.3965 4.762 9.7349 4.762 13.6499 0 1.0582-.099 2.1163-.198 3.1745-.794-1.6931-1.687-3.0686-2.48-4.2326z"/><path d="m119.087 91.4031-42.6562 52.9069h9.92l42.6562-52.9069z"/><path d="m255.677 91.4031-42.656 52.9069h9.92l42.656-52.9069z"/><path d="m200.526 71.7218c.1-.2116.1-.3175.1-.5291 0-.5291-.298-1.0581-.893-1.2697-.199-.1059-.397-.1059-.496-.1059-.595 0-.992.3175-1.29.8465-2.381 5.0791-6.349 10.793-11.805 12.2744 1.191-1.4814 1.687-3.1744 1.687-5.079 0-4.1268-3.472-7.3012-7.341-7.3012-3.869.2117-6.845 3.7035-6.845 7.8302.199 5.9256 5.555 7.936 10.218 7.936 4.96-.2116 8.829-2.8569 11.805-6.9837l-4.464 16.8244c-1.389 5.0793-7.044 11.8513-12.103 13.2263 1.091-1.481 1.687-3.28 1.687-5.079 0-4.232-3.373-7.3008-7.242-7.3008-3.968 0-6.944 3.5978-6.944 7.8298 0 3.175 2.083 5.82 4.662 6.772 1.885.847 3.572 1.059 5.556 1.059 4.96-.106 8.828-2.857 11.804-6.984l-10.118 38.622 2.678.74z"/><path d="m63.9264 71.7218c.0992-.2116.0992-.3175.0992-.5291 0-.5291-.2976-1.0581-.8928-1.2697-.1984-.1059-.3968-.1059-.496-.1059-.5952 0-.992.3175-1.2896.8465-2.3808 5.0791-6.3488 10.793-11.8048 12.2744 1.1904-1.4814 1.6864-3.1744 1.6864-5.079 0-4.1268-3.472-7.3012-7.3408-7.3012-3.8688.2117-6.8448 3.7035-6.8448 7.8302.1984 5.9256 5.5552 7.936 10.2176 7.936 4.96-.2116 8.8288-2.8569 11.8048-6.9837l-4.464 16.8244c-1.3888 5.0793-7.0432 11.8513-12.1024 13.2263 1.0912-1.481 1.6864-3.28 1.6864-5.079 0-4.232-3.3728-7.3008-7.2416-7.3008-3.968 0-6.944 3.5978-6.944 7.8298 0 3.175 2.0832 5.82 4.6624 6.772 1.8848.847 3.5712 1.059 5.5552 1.059 4.96-.106 8.8288-2.857 11.8048-6.984l-10.1184 38.622 2.6784.74z"/></g></svg>
          </span>
          <InputRange v-model.number="sixteenthVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>

        <div class="flex items-center justify-between space-x-4">
          <span class="block w-10 flex items-center justify-center">
            <svg fill="none" viewBox="0 0 320 170" class="w-10" xmlns="http://www.w3.org/2000/svg"><path d="m233.11 116.459v-59.5203" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><path d="m168.14 116.459v-59.5203" stroke="#374151" stroke-linejoin="bevel" stroke-width="2.48"/><g fill="#374151"><path d="m159.466 116.459-42.656 52.906h9.92l42.656-52.906z"/><path d="m224.436 116.459-42.656 52.906h9.92l42.656-52.906z"/><path d="m101.486 94.9786c-.099 0-.297-.1058-.396-.1058-.596 0-1.0916.3174-1.29.8465-2.2816 4.8677-7.0432 11.4277-12.4 12.4857 1.0912-1.375 1.7856-3.068 1.7856-5.079 0-4.1265-3.1744-7.5125-7.0432-7.5125-3.968 0-7.1424 3.386-7.1424 7.5125 0 5.714 5.2576 8.36 9.92 8.36 5.2576 0 8.7296-2.646 12.1024-6.561l-11.3088 37.67 2.6784.846 13.888-46.6636c.099-.1058.099-.3174.099-.529 0-.5291-.297-.9523-.893-1.2698z"/><path clip-rule="evenodd" d="m166.9 50.3254h67.45v13.2267h-67.45z" fill-rule="evenodd"/><path d="m164.83 16.32c2.12-.72 2.97-1.104 4.05-1.872 2.12-1.488 3.42-4.128 3.42-6.81603 0-4.56-3.42-7.63197-8.46-7.63197-4.9 0-9.04 3.264-9.04 7.2 0 1.968 1.12 3.312 2.83 3.312 1.49 0 2.57-1.15202 2.57-2.73602 0-.816-.23-1.39203-.95-2.16003-.45-.432-.63-.76797-.63-1.10397 0-1.152 2.25-2.35202 4.37-2.35202 3.01 0 4.81 1.77602 4.81 4.75202 0 4.27202-2.83 8.30392-5.85 8.30392-.13 0-.54-.0479-1.03-.0959-1.31-.096-1.31-.096-1.58-.096-.99 0-1.53.48-1.53 1.344 0 .768.5 1.2479 1.31 1.2479.22 0 .54-.0479.9-.0959.63-.144 1.3-.192 1.71-.192 2.11 0 3.55 2.016 3.55 4.992 0 5.184-2.38 10.176-7.83 10.176-2.52 0-4.68-1.3441-4.68-2.8801 0-.528.18-.7199.9-1.0559 1.31-.624 1.85-1.488 1.85-2.784 0-1.68-1.26-2.976-2.88-2.976-1.94 0-3.2 1.632-3.2 4.128 0 4.656 3.33 7.584 8.73 7.584 6.8 0 11.97-4.848 11.97-11.28 0-3.744-2.16-5.856-5.31-6.912z"/></g><path d="m76.24 37.0986v-19.8399h60.81" stroke="#374151" stroke-miterlimit="2" stroke-width="2.48"/><path d="m184.71 17.2587h60.8v19.8399" stroke="#374151" stroke-miterlimit="2" stroke-width="2.48"/></svg>
          </span>
          <InputRange v-model.number="tripletVolume" :min="0" :max="1" :step="0.01" class="w-56" />
        </div>
      </div>

      <button @click="togglePlay" :class="`flex items-center justify-center rounded-full w-60 h-60 text-lg ${isPlaying ? 'hover:nm-concave-gray-200-lg nm-concave-gray-200-lg text-gray-600' : 'nm-flat-gray-200-xl hover:nm-concave-gray-200-lg text-gray-400'}`">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor" class="w-10 h-10">
          <path fill-rule="evenodd" d="M4.5 5.653c0-1.427 1.529-2.33 2.779-1.643l11.54 6.347c1.295.712 1.295 2.573 0 3.286L7.28 19.99c-1.25.687-2.779-.217-2.779-1.643V5.653Z" clip-rule="evenodd" />
        </svg>
      </button>
    </div>

    <div class="text-gray-400 flex items-center justify-center mt-10 text-xs">
      <span>NeumoSync v{{ version }} / Developed by <a href="https://twitter.com/koedamedev" target="_blank" class="hover:text-gray-700">肥溜め@koedamedev</a></span>
    </div>
  </div>
</template>
