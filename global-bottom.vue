<template>
  <footer
    v-if="$slidev?.nav?.current !== 1"
    class="absolute bottom-3 right-4 text-xs font-mono text-gray-400 opacity-80 z-50 pointer-events-none select-none"
  >
    {{ $slidev?.nav?.current }} / {{ $slidev?.nav?.total }}
  </footer>
</template>
