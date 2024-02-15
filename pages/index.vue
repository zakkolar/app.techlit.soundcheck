<script setup>

const level = ref(0);

const ACCESS_MODES = Object.freeze({
    ASK: 'ASK',
    ENABLED: 'ENABLED',
    BLOCKED: 'BLOCKED'
})

useHead({
    title: 'Sound Check'
});

const access = ref(ACCESS_MODES.ASK);
const recording = ref(false);

const audioSrc = ref("");

let mediaRecorder = null;
let chunks = [];

const inputDevices = ref([]);
const currentInput = ref(null);
const outputDevices = ref([]);
const currentOutput = ref(null);

const updateAudioIO = () => {
    [...document.getElementsByTagName('audio')].forEach(item => item.setSinkId(currentOutput.value))
}


const updateDeviceList = () => {
    navigator.mediaDevices.enumerateDevices().then(devices => {
        inputDevices.value.length = 0;
        outputDevices.value.length = 0;
        devices.forEach(device => {

            const info = {
                name: device.label,
                id: device.deviceId
            }

            if(device.kind === 'audioinput') {
                inputDevices.value.push(info);
            }
            else if (device.kind === 'audiooutput') {
                outputDevices.value.push(info);
            }

        })

        if(!currentInput.value || !inputDevices.value.find(item => item.id === currentInput.value)) {
            currentInput.value = inputDevices.value[0].id;
        }

        if(!currentOutput.value) {
            currentOutput.value = outputDevices.value[0].id;
        }

        updateAudioIO();

    })
}

const enable = () => {
    if(access.value === ACCESS_MODES.ASK) {
        navigator.mediaDevices.getUserMedia({
            audio: true
        })
            .then(function(stream) {
                navigator.mediaDevices.addEventListener('devicechange', updateDeviceList);
                updateDeviceList();
                access.value = ACCESS_MODES.ENABLED;
                const audioContext = new AudioContext();
                const analyser = audioContext.createAnalyser();
                const microphone = audioContext.createMediaStreamSource(stream);
                mediaRecorder = new MediaRecorder(stream);
                mediaRecorder.ondataavailable = e => {
                    chunks.push(e.data);
                }

                mediaRecorder.onstop = e => {
                    audioSrc.value = window.URL.createObjectURL(new Blob(chunks, { type: "audio/ogg; codecs=opus" }));
                    chunks = [];
                }

                const scriptProcessor = audioContext.createScriptProcessor(2048, 1, 1);

                analyser.smoothingTimeConstant = 0.8;
                analyser.fftSize = 1024;

                microphone.connect(analyser);
                analyser.connect(scriptProcessor);
                scriptProcessor.connect(audioContext.destination);
                scriptProcessor.onaudioprocess = function() {
                    const array = new Uint8Array(analyser.frequencyBinCount);
                    analyser.getByteFrequencyData(array);
                    const arraySum = array.reduce((a, value) => a + value, 0);
                    const average = arraySum / array.length;

                    level.value = Math.round(average);

                };
            })
            .catch(function(err) {
                console.error(err);
                access.value = ACCESS_MODES.BLOCKED;
            });
    }

}

const record = () => {
    mediaRecorder.start();
    recording.value = true;
}

const stop = () => {
    mediaRecorder.stop();
    recording.value = false;
}

const deleteAudio = () => {
    audioSrc.value = "";
}


</script>
<template>
    <div class="prose mx-auto mt-5">
        <h1>Sound check</h1>
        <p>No ads, no data collection, no nonsense.</p>




        <div v-if="access === ACCESS_MODES.ENABLED">
            <h2>Speakers</h2>

            <select v-model="currentOutput" @change="updateAudioIO">
                <option v-for="output of outputDevices" :value="output.id">{{output.name}}</option>
            </select>

            <p>If you hear classical music, your speakers work!</p>
            <audio id="headphones" controls src="/audio/test.mp3"></audio>

            <h2>Microphone</h2>

<!--            <select v-model="currentInput" @change="updateAudioIO">-->
<!--                <option v-for="input of inputDevices" :value="input.id">{{input.name}}</option>-->
<!--            </select>-->

            <div>
                Microphone level: <progress max="100" :value="level"></progress>
            </div>
            <div>
                <button v-if="!recording" @click="record">Record</button>
                <button v-if="recording" @click="stop">Stop</button>
                <span v-show="!recording && audioSrc">
                    <button v-if="audioSrc" @click="deleteAudio">Delete</button>
                <audio class="inline-block" v-show="audioSrc" controls :src="audioSrc"></audio>
                </span>

            </div>

        </div>
        <div v-else-if="access === ACCESS_MODES.ASK">
            <button @click="enable">Enable microphone</button>
        </div>
        <div v-else>
            Microphone access is blocked.
        </div>


    </div>



</template>

<style scoped>
button {
    @apply border p-2 hover:bg-gray-50 transition-colors mr-2;
}

select {
    @apply p-2;
}
</style>