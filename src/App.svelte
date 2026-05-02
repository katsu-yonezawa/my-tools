<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import { invoke } from "@tauri-apps/api/core";

  let now = new Date();
  let timer: number | undefined;

  const formatter = new Intl.DateTimeFormat("ja-JP", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });

  onMount(() => {
    timer = window.setInterval(() => {
      now = new Date();
    }, 1000);

    void invoke<number>("current_timestamp").catch(() => undefined);
  });

  onDestroy(() => {
    if (timer !== undefined) {
      window.clearInterval(timer);
    }
  });
</script>

<main class="clock-screen">
  <time class="clock" datetime={now.toISOString()}>{formatter.format(now)}</time>
</main>
