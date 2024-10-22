<template>
  <div>
    <button @click="startAnalyzing">Start Analyzing</button>
    <button @click="stopAnalyzing">Stop Analyzing</button>
    <div>{{ dominantFrequency.toFixed(2) }} Hz</div>
    <div>{{ frequencyHistory }}</div>
    <VisXYContainer :data="data" v-if="data.length > 0" class="w-full" :duration="0">
      <VisArea :x="x" :y="y" />
    </VisXYContainer>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { VisXYContainer, VisArea } from '@unovis/vue'
const data = ref([]);
const dominantFrequency = ref(0);
const frequencyHistory = ref([]); // Historique des fréquences
const historySize = 5; // Taille de l'historique pour le lissage

const x = (d) => d.y;
const y = (d) => d.x;

let isAnalyzing = false;
let analyser, audioContext, dataArray, bufferLength;
let lastTime = 0;
const fps = 5

const startAnalyzing = () => {
  if (!isAnalyzing) {
    isAnalyzing = true;
    requestAnimationFrame(getFrequency);
  }
};

const stopAnalyzing = () => {
  isAnalyzing = false;
};

const getFrequency = (timestamp) => {
  if (!isAnalyzing) return;

  const elapsed = timestamp - lastTime;
  if (elapsed < 1000 / fps) {
    requestAnimationFrame(getFrequency);
    return;
  }

  lastTime = timestamp;

  analyser.getByteFrequencyData(dataArray);

  let maxIndex = 0;
  for (let i = 1; i < bufferLength; i++) {
    if (dataArray[i] > dataArray[maxIndex]) {
      maxIndex = i;
    }
  }

  console.log(maxIndex);


  if (maxIndex === 0 || maxIndex === bufferLength - 1) {
    return;
  }

  const alpha = dataArray[maxIndex - 1] || 0;
  const beta = dataArray[maxIndex];
  const gamma = dataArray[maxIndex + 1] || 0;

  const peakOffset = (gamma - alpha) / (2 * (2 * beta - alpha - gamma));
  const interpolatedIndex = maxIndex + peakOffset;

  const nyquist = audioContext.sampleRate / 2;
  const frequency = (interpolatedIndex / bufferLength) * nyquist;

  if (isNaN(frequency)) {
    return;
  }

  frequencyHistory.value.push(frequency);
  if (frequencyHistory.value.length > historySize) {
    frequencyHistory.value.shift();
  }

  const smoothedFrequency = frequencyHistory.value.reduce((acc, val) => acc + val, 0) / frequencyHistory.value.length;

  dominantFrequency.value = smoothedFrequency;

  data.value = Array.from(dataArray).slice(0, 400).map((data, index) => ({ x: data, y: index }));

  requestAnimationFrame(getFrequency);
};

onMounted(() => {
  if (navigator.mediaDevices.getUserMedia) {
    navigator.mediaDevices.getUserMedia({ audio: true })
      .then((stream) => {
        audioContext = new AudioContext();
        analyser = audioContext.createAnalyser();
        const source = audioContext.createMediaStreamSource(stream);

        source.connect(analyser);

        analyser.fftSize = 4096;
        bufferLength = analyser.frequencyBinCount;
        dataArray = new Uint8Array(bufferLength);
      })
      .catch((err) => {
        console.error("Erreur d'accès au microphone", err);
      });
  }
});

</script>
