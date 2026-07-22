<script setup>
const props = defineProps({
    src: { type: String, default: '' },
});

const audioEl = ref(null);
const playing = ref(false);
const currentTime = ref(0);
const duration = ref(0);

const formatTime = (t) => {
    if (!isFinite(t) || t < 0) t = 0;
    const m = Math.floor(t / 60);
    const s = Math.floor(t % 60).toString().padStart(2, '0');
    return `${m}:${s}`;
};

const togglePlay = () => {
    if (!audioEl.value) return;
    if (playing.value) {
        audioEl.value.pause();
    } else {
        audioEl.value.play();
    }
};

let resolvingDuration = false;

const onTimeUpdate = () => {
    if (!resolvingDuration) {
        currentTime.value = audioEl.value.currentTime;
    }
    const d = audioEl.value.duration;
    if (isFinite(d)) {
        duration.value = d;
    }
};

// Some MP3s omit a duration header, so Chromium reports duration as
// Infinity until it's forced to seek near the end of the file.
const resolveInfiniteDuration = () => {
    const el = audioEl.value;
    if (!el || resolvingDuration) return;
    resolvingDuration = true;
    const onSeekResolved = () => {
        el.removeEventListener('timeupdate', onSeekResolved);
        el.currentTime = 0;
        resolvingDuration = false;
    };
    el.addEventListener('timeupdate', onSeekResolved);
    el.currentTime = 1e101;
};

const onDurationChange = () => {
    const el = audioEl.value;
    if (!el) return;
    if (isFinite(el.duration)) {
        duration.value = el.duration;
    } else {
        resolveInfiniteDuration();
    }
};

const onLoadedMetadata = () => {
    const el = audioEl.value;
    if (!el) return;
    if (isFinite(el.duration)) {
        duration.value = el.duration;
    } else {
        resolveInfiniteDuration();
    }
};

const onSeek = (e) => {
    const value = Number(e.target.value);
    if (audioEl.value) {
        audioEl.value.currentTime = value;
    }
    currentTime.value = value;
};

onMounted(() => {
    const el = audioEl.value;
    if (!el) return;
    // The browser can start loading (and fire loadedmetadata/durationchange)
    // from the server-rendered <audio> tag before Vue attaches its listeners,
    // so check the already-loaded state directly once mounted too.
    if (isFinite(el.duration)) {
        duration.value = el.duration;
    } else if (el.readyState > 0) {
        resolveInfiniteDuration();
    }
});

watch(() => props.src, () => {
    playing.value = false;
    currentTime.value = 0;
    duration.value = 0;
    nextTick(() => {
        audioEl.value?.load();
    });
});

const progressPercent = computed(() => (duration.value ? (currentTime.value / duration.value) * 100 : 0));
</script>

<template>
    <div class="flex items-center gap-3 border-2 border-ink bg-paper px-3 py-1 shadow-[3px_3px_0_theme(colors.ink)]">
        <button
            type="button"
            class="flex h-7 w-7 shrink-0 items-center justify-center text-sm text-ink transition-colors enabled:hover:text-ink/70 disabled:cursor-not-allowed disabled:text-hairline"
            :disabled="!src"
            :aria-label="playing ? 'Pause' : 'Play'"
            @click="togglePlay"
        >
            <span v-if="!playing" aria-hidden="true">&#9654;</span>
            <span v-else aria-hidden="true">&#10074;&#10074;</span>
        </button>

        <div class="relative h-4 flex-1">
            <div class="absolute left-0 top-1/2 h-1 w-full -translate-y-1/2 bg-ink/15"></div>
            <div
                class="pointer-events-none absolute left-0 top-1/2 h-1 -translate-y-1/2 bg-grape"
                :style="{ width: progressPercent + '%' }"
            ></div>
            <input
                class="player-seek absolute inset-0 h-4 w-full cursor-pointer appearance-none bg-transparent disabled:cursor-not-allowed"
                type="range"
                min="0"
                :max="duration || 0"
                step="0.01"
                :value="currentTime"
                :disabled="!duration"
                aria-label="Seek"
                @input="onSeek"
            />
        </div>

        <span class="shrink-0 font-mono text-[11px] tabular-nums text-ink/60">{{ formatTime(currentTime) }} / {{ formatTime(duration) }}</span>

        <audio
            ref="audioEl"
            class="hidden"
            :src="src"
            preload="auto"
            @play="playing = true"
            @pause="playing = false"
            @ended="playing = false"
            @timeupdate="onTimeUpdate"
            @durationchange="onDurationChange"
            @loadedmetadata="onLoadedMetadata"
        ></audio>
    </div>
</template>

<style scoped>
.player-seek::-webkit-slider-runnable-track {
    @apply h-1 bg-transparent;
}

.player-seek::-moz-range-track {
    @apply h-1 bg-transparent;
    border: none;
}

.player-seek::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 8px;
    height: 16px;
    margin-top: -6px;
    @apply bg-ink transition-colors;
    cursor: pointer;
}

.player-seek:hover::-webkit-slider-thumb {
    @apply bg-ink/70;
}

.player-seek::-moz-range-thumb {
    width: 8px;
    height: 16px;
    @apply bg-ink transition-colors;
    border: none;
    border-radius: 0;
    cursor: pointer;
}

.player-seek:hover::-moz-range-thumb {
    @apply bg-ink/70;
}
</style>
