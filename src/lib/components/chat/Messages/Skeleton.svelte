<script lang="ts">
  export let size = 'md'; // You can use this to control size if desired
</script>

<style>
  @keyframes bounce {
    0%, 100% {
      transform: translateY(0);
      opacity: 0.4;
    }
    50% {
      transform: translateY(-8px);
      opacity: 1;
    }
  }

  .dot {
    width: 8px;
    height: 8px;
    background-color: #6b7280; /* Tailwind gray-500 */
    border-radius: 50%;
    display: inline-block;
    margin: 0 4px;
    animation: bounce 1.4s ease-in-out infinite;
  }

  .dot:nth-child(1) {
    animation-delay: 0s;
  }

  .dot:nth-child(2) {
    animation-delay: 0.2s;
  }

  .dot:nth-child(3) {
    animation-delay: 0.4s;
  }
</style>

<!-- Container with content aligned to left -->
<div class="flex items-center h-16 w-full pl-2">
  <div class="flex space-x-2">
    <div class="dot" />
    <div class="dot" />
    <div class="dot" />
  </div>
</div>