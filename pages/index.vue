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

        currentInput.value = inputDevices.value[0].id;

        currentOutput.value = outputDevices.value[0].id;

        updateAudioIO();
        enable();

    })
}

const enable = (initial = false) => {

    const constraints = {
        audio: true
    };

    if(currentInput.value) {
        constraints.audio = {
            exact: currentInput.value
        }
    }

        navigator.mediaDevices.getUserMedia({
            audio: true
        })
            .then(function(stream) {
                if(initial) {
                    navigator.mediaDevices.addEventListener('devicechange', updateDeviceList);
                    updateDeviceList();
                    access.value = ACCESS_MODES.ENABLED;
                }
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

const TOTAL_SEGMENTS = 20;
const meterSegments = computed(() => Array.from({ length: TOTAL_SEGMENTS }));
const clampedLevel = computed(() => Math.min(100, Math.max(0, level.value)));
const filledSegments = computed(() => Math.round((clampedLevel.value / 100) * TOTAL_SEGMENTS));
const levelDisplay = computed(() => String(Math.round(clampedLevel.value)).padStart(3, '0'));

const segmentTone = (index) => {
    if (index < 13) return 'border-grape bg-grape';
    if (index < 17) return 'border-amber bg-amber';
    return 'border-signal bg-signal';
};

</script>
<template>
    <div class="relative">
        <span class="reg top-5 left-5" aria-hidden="true">+</span>
        <span class="reg top-5 right-5" aria-hidden="true">+</span>
        <span class="reg bottom-5 left-5" aria-hidden="true">+</span>
        <span class="reg bottom-5 right-5" aria-hidden="true">+</span>

        <div class="mx-auto max-w-2xl px-6 py-14 sm:py-20">
            <header class="mb-10 border-b-2 border-hairline pb-5">
                <div class="flex flex-wrap items-baseline justify-between gap-x-4 gap-y-2">
                    <div class="flex items-center gap-3">
                      <WaveformMark class="shrink-0 text-grape" />
                        <h1 class="font-display text-5xl font-extrabold uppercase tracking-tight sm:text-6xl">
                            Sound Check
                        </h1>

                    </div>
                    <p class="font-mono text-[11px] uppercase tracking-widest text-ink/60">
                        No ads &middot; No data collection
                    </p>
                </div>
            </header>

            <main class="space-y-8">
                <section class="panel">
                    <h2 class="panel-label">Output &mdash; Speakers</h2>
                    <div class="panel-body">
                        <p>If your speakers work, you should hear classical music.</p>
                        <RetroAudioPlayer class="mt-3" src="/audio/test.mp3" />
                    </div>
                </section>

                <section class="panel">
                    <h2 class="panel-label">Input &mdash; Microphone</h2>
                    <div class="panel-body">
                        <div v-if="access === ACCESS_MODES.ENABLED" class="space-y-5">
                            <div>
                                <div class="mb-1.5 flex items-center justify-between font-mono text-xs uppercase tracking-widest text-ink/60">
                                    <span>Level</span>
                                    <span>{{ levelDisplay }}</span>
                                </div>
                                <div
                                    class="flex gap-[3px]"
                                    role="meter"
                                    aria-label="Microphone level"
                                    :aria-valuenow="clampedLevel"
                                    aria-valuemin="0"
                                    aria-valuemax="100"
                                >
                                    <span
                                        v-for="(seg, i) in meterSegments"
                                        :key="i"
                                        class="h-5 flex-1 border"
                                        :class="i < filledSegments ? segmentTone(i) : 'border-ink/15 bg-ink/5'"
                                    ></span>
                                </div>
                            </div>

                            <div class="flex flex-wrap items-center gap-3">
                                <button v-if="!recording" class="btn bg-paper text-ink hover:bg-grape hover:text-paper" @click="record">&#9679; Record</button>
                                <button v-if="recording" class="btn bg-paper text-ink hover:bg-amber" @click="stop">
                                    <span
                                        class="mr-1.5 inline-block h-2 w-2 rounded-full bg-ink align-middle motion-safe:animate-pulse"
                                        aria-hidden="true"
                                    ></span>Stop
                                </button>
                                <button v-if="!recording && audioSrc" class="btn bg-paper text-ink hover:bg-[#c7c7c7]" @click="deleteAudio">Delete</button>
                                <RetroAudioPlayer
                                    v-show="!recording && audioSrc"
                                    class="min-w-[180px] grow"
                                    :src="audioSrc"
                                />
                            </div>
                        </div>

                        <div v-else-if="access === ACCESS_MODES.ASK">
                            <button class="btn bg-paper text-ink hover:bg-grape hover:text-paper" @click="enable(true)">Enable microphone to test</button>
                        </div>

                        <div v-else class="border-2 border-amber bg-amber/10 px-4 py-3 text-sm">
                            Microphone access is blocked.
                        </div>
                    </div>
                </section>

                <div class="divide-y divide-hairline">
                    <Faq question="Is this app privacy-friendly?">
                        <p>Yes! No data is collected and nothing you record leaves your browser.</p>
                    </Faq>
                    <Faq question="Is this open-source?">
                        <p>Yes! The source code is <a href="https://github.com/zakkolar/tools.techlit.soundcheck" target="_blank">here</a>.</p>
                    </Faq>
                    <Faq question="Who made this?">
                        <p>My name is Zak Kolar and I'm an educator. See more information <a href="https://techlit.tools/about/">here</a>.</p>
                    </Faq>
                </div>
            </main>
        </div>
    </div>
</template>

<style scoped>
.panel {
    @apply border-2 border-ink;
}

.panel-label {
    @apply bg-ink px-3 py-1.5 font-mono text-xs uppercase tracking-[0.2em] text-paper;
}

.panel-body {
    @apply p-5 sm:p-6;
}

.btn {
    @apply border-2 border-ink px-4 py-2.5 font-mono text-xs uppercase tracking-widest transition-all shadow-[3px_3px_0_theme(colors.ink)];
}

.btn:active {
    @apply translate-x-[3px] translate-y-[3px] shadow-none;
}

.reg {
    @apply pointer-events-none fixed hidden select-none font-mono text-lg text-hairline sm:block;
}
</style>