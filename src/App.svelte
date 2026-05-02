<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import { invoke } from "@tauri-apps/api/core";
  import {
    isPermissionGranted,
    requestPermission,
    sendNotification,
  } from "@tauri-apps/plugin-notification";

  type ToolId = "clock" | "uuid";
  type TimerPalette = {
    id: string;
    label: string;
    color: string;
    soft: string;
  };

  let now = new Date();
  let timer: number | undefined;
  let uuids: string[] = [];
  let copiedUuid = "";
  let activeTool: ToolId = "clock";
  let pomodoroDurationSeconds = 25 * 60;
  let pomodoroRemainingSeconds = pomodoroDurationSeconds;
  let pomodoroEndsAt: number | undefined;
  let isPomodoroRunning = false;
  let hasSentPomodoroCompletionNotification = false;
  let selectedPaletteId = "tomato";
  let customPomodoroMinutes: number | undefined = 30;
  let customPomodoroError = "";

  const maxDialSeconds = 60 * 60;
  const minPomodoroMinutes = 1;
  const maxPomodoroMinutes = 180;
  const timerPaletteStorageKey = "my-tools:pomodoro-timer-palette";

  const tools: Array<{ id: ToolId; label: string; icon: string }> = [
    { id: "clock", label: "時計", icon: "12" },
    { id: "uuid", label: "UUID", icon: "#" },
  ];

  const pomodoroPresetGroups = [
    {
      label: "集中",
      icon: "focus",
      presets: [
        { label: "通常", minutes: 25 },
        { label: "深い集中", minutes: 50 },
      ],
    },
    {
      label: "休憩",
      icon: "break",
      presets: [
        { label: "短い休憩", minutes: 5 },
        { label: "長い休憩", minutes: 15 },
      ],
    },
  ];

  const timerPalettes: TimerPalette[] = [
    { id: "tomato", label: "Tomato", color: "#d84a3a", soft: "#f7ddd9" },
    { id: "sage", label: "Sage", color: "#4f8f74", soft: "#dfeee8" },
    { id: "sky", label: "Sky", color: "#3d7fb8", soft: "#dceaf5" },
    { id: "amber", label: "Amber", color: "#c27a22", soft: "#f3e5cf" },
    { id: "plum", label: "Plum", color: "#875aa6", soft: "#eadff0" },
  ];

  const timerMarks = Array.from({ length: 60 }, (_, index) => ({
    isHour: index % 5 === 0,
    rotation: index * 6,
  }));

  const timerNumbers = Array.from({ length: 12 }, (_, index) => {
    const position = polarToCartesian(120, 74, -90 + index * 30);

    return {
      label: index === 0 ? "60" : String(index * 5),
      x: position.x,
      y: position.y,
    };
  });

  const timeFormatter = new Intl.DateTimeFormat("ja-JP", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  });

  const dateFormatter = new Intl.DateTimeFormat("ja-JP", {
    year: "numeric",
    month: "long",
    day: "numeric",
    weekday: "long",
  });

  const timeZoneName =
    Intl.DateTimeFormat().resolvedOptions().timeZone ?? "Local time";

  $: selectedPalette =
    timerPalettes.find((palette) => palette.id === selectedPaletteId) ??
    timerPalettes[0];
  $: pomodoroDialProgress = pomodoroRemainingSeconds / maxDialSeconds;
  $: pomodoroSectorPath = describeSector(pomodoroDialProgress);
  $: isCustomPomodoroDuration = !pomodoroPresetGroups.some((group) =>
    group.presets.some((preset) => preset.minutes * 60 === pomodoroDurationSeconds),
  );

  function createUuid() {
    const uuid = crypto.randomUUID();
    uuids = [uuid, ...uuids].slice(0, 8);
    copiedUuid = "";
  }

  function clearUuids() {
    uuids = [];
    copiedUuid = "";
  }

  async function copyUuid(uuid: string) {
    await navigator.clipboard.writeText(uuid);
    copiedUuid = uuid;
  }

  function setPomodoroDuration(minutes: number) {
    const normalizedMinutes = Math.trunc(minutes);

    pomodoroDurationSeconds = normalizedMinutes * 60;
    pomodoroRemainingSeconds = pomodoroDurationSeconds;
    pomodoroEndsAt = undefined;
    isPomodoroRunning = false;
    hasSentPomodoroCompletionNotification = false;
    customPomodoroMinutes = normalizedMinutes;
    customPomodoroError = "";
  }

  function applyCustomPomodoroDuration() {
    const minutes = Number(customPomodoroMinutes);

    if (customPomodoroMinutes === undefined || !Number.isFinite(minutes)) {
      customPomodoroError = "分数を入力してください。";
      return;
    }

    if (!Number.isInteger(minutes)) {
      customPomodoroError = "分単位の整数で入力してください。";
      return;
    }

    if (minutes < minPomodoroMinutes || minutes > maxPomodoroMinutes) {
      customPomodoroError = `${minPomodoroMinutes}〜${maxPomodoroMinutes}分で入力してください。`;
      return;
    }

    setPomodoroDuration(minutes);
  }

  function selectTimerPalette(paletteId: string) {
    selectedPaletteId = paletteId;
    saveSelectedTimerPalette(paletteId);
  }

  function loadSavedTimerPalette() {
    try {
      const savedPaletteId = window.localStorage.getItem(timerPaletteStorageKey);

      if (
        savedPaletteId !== null &&
        timerPalettes.some((palette) => palette.id === savedPaletteId)
      ) {
        selectedPaletteId = savedPaletteId;
      }
    } catch {
      // Local storage can be unavailable in restricted webviews.
    }
  }

  function saveSelectedTimerPalette(paletteId: string) {
    try {
      window.localStorage.setItem(timerPaletteStorageKey, paletteId);
    } catch {
      // Color persistence is a convenience; the timer can still run without it.
    }
  }

  function startPomodoro() {
    if (pomodoroRemainingSeconds <= 0) {
      pomodoroRemainingSeconds = pomodoroDurationSeconds;
    }

    pomodoroEndsAt = Date.now() + pomodoroRemainingSeconds * 1000;
    isPomodoroRunning = true;
    hasSentPomodoroCompletionNotification = false;

    void ensureNotificationPermission();
  }

  function pausePomodoro() {
    updatePomodoroRemaining();
    pomodoroEndsAt = undefined;
    isPomodoroRunning = false;
  }

  function resetPomodoro() {
    pomodoroRemainingSeconds = pomodoroDurationSeconds;
    pomodoroEndsAt = undefined;
    isPomodoroRunning = false;
    hasSentPomodoroCompletionNotification = false;
  }

  function updatePomodoroRemaining() {
    if (!isPomodoroRunning || pomodoroEndsAt === undefined) {
      return;
    }

    pomodoroRemainingSeconds = Math.max(
      0,
      Math.ceil((pomodoroEndsAt - Date.now()) / 1000),
    );

    if (pomodoroRemainingSeconds === 0) {
      pomodoroEndsAt = undefined;
      isPomodoroRunning = false;

      if (!hasSentPomodoroCompletionNotification) {
        hasSentPomodoroCompletionNotification = true;
        void sendPomodoroCompletionNotification();
      }
    }
  }

  async function ensureNotificationPermission() {
    try {
      if (await isPermissionGranted()) {
        return true;
      }

      return (await requestPermission()) === "granted";
    } catch {
      return false;
    }
  }

  async function sendPomodoroCompletionNotification() {
    if (!(await ensureNotificationPermission())) {
      return;
    }

    try {
      sendNotification({
        title: "ポモドーロ終了",
        body: "時間になりました。少し休憩しましょう。",
      });
    } catch {
      // Notification support depends on the Tauri runtime and OS permissions.
    }
  }

  function formatTimer(seconds: number) {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = seconds % 60;

    return `${minutes.toString().padStart(2, "0")}:${remainingSeconds
      .toString()
      .padStart(2, "0")}`;
  }

  function describeSector(progress: number) {
    const radius = 92;
    const center = 120;
    const ratio = Math.min(1, Math.max(0, progress));

    if (ratio <= 0) {
      return "";
    }

    if (ratio >= 0.999) {
      return [
        `M ${center} ${center}`,
        `m ${-radius} 0`,
        `a ${radius} ${radius} 0 1 0 ${radius * 2} 0`,
        `a ${radius} ${radius} 0 1 0 ${-radius * 2} 0`,
      ].join(" ");
    }

    const start = polarToCartesian(center, radius, -90);
    const end = polarToCartesian(center, radius, -90 + ratio * 360);
    const largeArcFlag = ratio > 0.5 ? 1 : 0;

    return [
      `M ${center} ${center}`,
      `L ${start.x} ${start.y}`,
      `A ${radius} ${radius} 0 ${largeArcFlag} 1 ${end.x} ${end.y}`,
      "Z",
    ].join(" ");
  }

  function polarToCartesian(center: number, radius: number, angleDegrees: number) {
    const angleRadians = (angleDegrees * Math.PI) / 180;

    return {
      x: center + radius * Math.cos(angleRadians),
      y: center + radius * Math.sin(angleRadians),
    };
  }

  onMount(() => {
    loadSavedTimerPalette();

    timer = window.setInterval(() => {
      now = new Date();
      updatePomodoroRemaining();
    }, 250);

    void invoke<number>("current_timestamp").catch(() => undefined);
    createUuid();
  });

  onDestroy(() => {
    if (timer !== undefined) {
      window.clearInterval(timer);
    }
  });
</script>

<main class="app-shell">
  <nav class="tool-dock" aria-label="ツールメニュー">
    {#each tools as tool}
      <button
        type="button"
        class:active={activeTool === tool.id}
        aria-label={tool.label}
        aria-current={activeTool === tool.id ? "page" : undefined}
        on:click={() => (activeTool = tool.id)}
      >
        <span class="dock-icon" aria-hidden="true">{tool.icon}</span>
        <span class="dock-label">{tool.label}</span>
      </button>
    {/each}
  </nav>

  <div class="tool-workspace">
    {#if activeTool === "clock"}
      <section
        class="tool-panel clock-panel"
        aria-label="ポモドーロタイマー"
        style={`--timer-color: ${selectedPalette.color}; --timer-soft: ${selectedPalette.soft};`}
      >
        <div class="clock-info">
          <div>
            <p class="section-label">Pomodoro Clock</p>
            <h1>ポモドーロ</h1>
          </div>
          <div class="current-time">
            <time datetime={now.toISOString()}>{timeFormatter.format(now)}</time>
            <span>{dateFormatter.format(now)} / {timeZoneName}</span>
          </div>
        </div>

        <div class="pomodoro-layout">
          <div class="analog-timer" aria-label={`残り時間 ${formatTimer(pomodoroRemainingSeconds)}`}>
            <svg viewBox="0 0 240 240" role="img" aria-hidden="true">
              <circle class="timer-face" cx="120" cy="120" r="108" />
              {#if pomodoroSectorPath}
                <path class="timer-sector" d={pomodoroSectorPath} />
              {/if}
              <circle class="timer-inner" cx="120" cy="120" r="42" />
              {#each timerMarks as mark}
                <line
                  class:hour-mark={mark.isHour}
                  class="timer-mark"
                  x1="120"
                  y1="18"
                  x2="120"
                  y2={mark.isHour ? 32 : 26}
                  style={`transform: rotate(${mark.rotation}deg);`}
                />
              {/each}
              {#each timerNumbers as number}
                <text class="timer-number" x={number.x} y={number.y}>{number.label}</text>
              {/each}
            </svg>
            <div class="timer-readout">
              <strong>{formatTimer(pomodoroRemainingSeconds)}</strong>
              <span>{isPomodoroRunning ? "進行中" : "待機中"}</span>
            </div>
          </div>

          <div class="pomodoro-controls">
            <div>
              <p class="control-label">時間</p>
              <div class="preset-zones">
                {#each pomodoroPresetGroups as group}
                  <section class="preset-zone" aria-label={group.label}>
                    <div class="preset-zone-header">
                      <span class="zone-icon" aria-hidden="true">
                        {#if group.icon === "focus"}
                          <svg viewBox="0 0 24 24">
                            <circle cx="12" cy="12" r="7" />
                            <circle cx="12" cy="12" r="2.5" />
                            <path d="M12 2v3M12 19v3M2 12h3M19 12h3" />
                          </svg>
                        {:else}
                          <svg viewBox="0 0 24 24">
                            <path d="M7 9h9v4a5 5 0 0 1-5 5H9a2 2 0 0 1-2-2V9Z" />
                            <path d="M16 10h2a2 2 0 0 1 0 4h-2M8 5h7M5 21h14" />
                          </svg>
                        {/if}
                      </span>
                      <span>{group.label}</span>
                    </div>

                    <div class="segmented-control">
                      {#each group.presets as preset}
                        <button
                          type="button"
                          class:active={pomodoroDurationSeconds === preset.minutes * 60}
                          on:click={() => setPomodoroDuration(preset.minutes)}
                        >
                          <span>{preset.label}</span>
                          <strong>{preset.minutes}分</strong>
                        </button>
                      {/each}
                    </div>
                  </section>
                {/each}
              </div>

              <form class="custom-duration" on:submit|preventDefault={applyCustomPomodoroDuration}>
                <label for="custom-pomodoro-minutes">カスタム</label>
                <div class="custom-duration-row">
                  <div class="custom-duration-input">
                    <input
                      id="custom-pomodoro-minutes"
                      type="number"
                      min={minPomodoroMinutes}
                      max={maxPomodoroMinutes}
                      step="1"
                      bind:value={customPomodoroMinutes}
                      class:error={customPomodoroError !== ""}
                      aria-describedby={customPomodoroError
                        ? "custom-pomodoro-help custom-pomodoro-error"
                        : "custom-pomodoro-help"}
                      on:input={() => (customPomodoroError = "")}
                    />
                    <span>分</span>
                  </div>
                  <button type="submit" class:active={isCustomPomodoroDuration}>設定</button>
                </div>
                <p id="custom-pomodoro-help" class="input-message">
                  {minPomodoroMinutes}〜{maxPomodoroMinutes}分まで設定できます
                </p>
                {#if customPomodoroError}
                  <p id="custom-pomodoro-error" class="input-message error-message" role="alert">
                    {customPomodoroError}
                  </p>
                {/if}
              </form>
            </div>

            <div>
              <p class="control-label">色</p>
              <div class="palette-list">
                {#each timerPalettes as palette}
                  <button
                    type="button"
                    class:active={selectedPaletteId === palette.id}
                    aria-label={palette.label}
                    title={palette.label}
                    style={`--swatch-color: ${palette.color};`}
                    on:click={() => selectTimerPalette(palette.id)}
                  ></button>
                {/each}
              </div>
            </div>

            <div class="timer-actions">
              {#if isPomodoroRunning}
                <button type="button" class="primary-button" on:click={pausePomodoro}>一時停止</button>
              {:else}
                <button type="button" class="primary-button" on:click={startPomodoro}>開始</button>
              {/if}
              <button type="button" class="secondary-button" on:click={resetPomodoro}>リセット</button>
            </div>
          </div>
        </div>
      </section>
    {:else if activeTool === "uuid"}
      <section class="tool-panel uuid-panel" aria-labelledby="uuid-title">
        <div class="panel-header">
          <div>
            <p class="section-label">UUID</p>
            <h1 id="uuid-title">UUID 作成</h1>
          </div>
          <div class="panel-actions">
            <button type="button" class="secondary-button" disabled={uuids.length === 0} on:click={clearUuids}>
              クリア
            </button>
            <button type="button" class="primary-button" on:click={createUuid}>作成</button>
          </div>
        </div>

        <div class="uuid-list" aria-live="polite">
          {#each uuids as uuid}
            <div class="uuid-row">
              <code>{uuid}</code>
              <button type="button" class="copy-button" on:click={() => copyUuid(uuid)}>
                {copiedUuid === uuid ? "コピー済み" : "コピー"}
              </button>
            </div>
          {/each}
        </div>
      </section>
    {/if}
  </div>
</main>
