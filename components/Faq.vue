<script setup>
defineProps({
    question: { type: String, required: true },
});
</script>

<template>
    <details class="py-3 [&_a]:underline">
        <summary
            class="cursor-pointer font-mono text-sm font-bold uppercase tracking-[0.2em] text-ink transition-colors hover:text-grape"
        >
            {{ question }}
        </summary>
        <div class="mt-2 pl-[18px] text-sm leading-relaxed text-ink/70">
            <slot />
        </div>
    </details>
</template>
