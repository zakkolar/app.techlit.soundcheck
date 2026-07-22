<template>
    <!--
      Native 44x48 size on purpose: the bars are only whole-pixel-uniform
      when the SVG renders at exactly its own viewBox size. Any CSS-driven
      scaling (e.g. Tailwind h-* classes) forces a fractional scale factor,
      and each bar's edges round to a different pixel width depending on
      where they land sub-pixel - that's what caused the uneven bars.
    -->
    <svg width="44" height="48" viewBox="0 0 44 48" fill="currentColor" shape-rendering="crispEdges" aria-hidden="true">
        <rect x="0" y="18" width="4" height="12" />
        <rect x="8" y="13" width="4" height="22" />
        <rect x="16" y="16" width="4" height="16" />
        <rect x="24" y="5" width="4" height="38" />
        <rect x="32" y="12" width="4" height="24" />
        <rect x="40" y="0" width="4" height="48" />
    </svg>
</template>
