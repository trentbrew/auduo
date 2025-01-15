<script lang="ts">
  import BrownNoiseGenerator from "./lib/BrownNoiseGenerator.svelte";
</script>

<main>
  <div class="container">
    <BrownNoiseGenerator />
  </div>
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 2rem;
  }

  h1 {
    color: #ff3e00;
    text-transform: uppercase;
    margin-bottom: 2rem;
  }

  .container {
    width: 100%;
    max-width: 500px;
  }
</style>
