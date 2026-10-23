<script lang='ts'>
  import { cn } from '$lib/utils'

  export let color = 'currentColor'
  export let size = 24
  export let strokeWidth = 2
  let className = ''

  export { className as class }
</script>

<svg
  xmlns='http://www.w3.org/2000/svg'
  width={size}
  height={size}
  viewBox='0 0 24 24'
  fill='none'
  stroke={color}
  stroke-width={strokeWidth}
  stroke-linecap='round'
  stroke-linejoin='round'
  class={cn(className, 'overflow-visible')}
  {...$$restProps}
>
  <path d='M8 5H4a2 2 0 0 0-2 2v10a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V7a2 2 0 0 0-2-2h-4' />
  <g>
    <path d='M12 3v12' class='target-animated-icon' />
    <path d='m8 11 4 4 4-4' class='target-animated-icon' />
  </g>
</svg>

<style>
  .target-animated-icon {
    transform: translateY(2px);
    transition: transform 0.3s cubic-bezier(0.68, -0.6, 0.32, 1.6);
  }
</style>
