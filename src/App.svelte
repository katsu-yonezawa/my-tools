<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import { invoke } from "@tauri-apps/api/core";
  import { listen, type UnlistenFn } from "@tauri-apps/api/event";
  import {
    isPermissionGranted,
    requestPermission,
    sendNotification,
  } from "@tauri-apps/plugin-notification";
  import QRCode from "qrcode";
  import aiChatMenuImage from "./assets/menu/ai-chat.png";
  import base64MenuImage from "./assets/menu/base64.png";
  import clockMenuImage from "./assets/menu/clock.png";
  import colorMenuImage from "./assets/menu/color.png";
  import datetimeMenuImage from "./assets/menu/datetime.png";
  import diffMenuImage from "./assets/menu/diff.png";
  import hashMenuImage from "./assets/menu/hash.png";
  import jsonMenuImage from "./assets/menu/json.png";
  import jwtMenuImage from "./assets/menu/jwt.png";
  import memoMenuImage from "./assets/menu/memo.png";
  import qrMenuImage from "./assets/menu/qr.png";
  import regexMenuImage from "./assets/menu/regex.png";
  import textCounterMenuImage from "./assets/menu/text-counter.png";
  import urlMenuImage from "./assets/menu/url.png";
  import uuidMenuImage from "./assets/menu/uuid.png";

  type ToolId =
    | "clock"
    | "uuid"
    | "json"
    | "datetime"
    | "base64"
    | "url"
    | "jwt"
    | "regex"
    | "text-counter"
    | "hash"
    | "diff"
    | "qr"
    | "color"
    | "memo"
    | "ai-chat";
  type ActiveView = "menu" | ToolId;
  type TimerPalette = {
    id: string;
    label: string;
    color: string;
    soft: string;
  };
  type DateTimeResult = {
    unixSeconds: string;
    unixMilliseconds: string;
    iso: string;
    utc: string;
    jst: string;
  };
  type JwtClaimTime = {
    name: string;
    unixSeconds: string;
    utc: string;
    jst: string;
  };
  type Base64Mode = "text" | "image";
  type AiChatRole = "user" | "assistant";
  type AiChatMessage = {
    id: string;
    role: AiChatRole;
    text: string;
    createdAt: string;
  };
  type AiConversation = {
    id: string;
    title: string;
    createdAt: string;
    updatedAt: string;
    messages: AiChatMessage[];
  };
  type AiChatSettings = {
    apiKey: string;
    region: string;
    modelId: string;
    temperature: number;
    maxTokens: number;
    tavilyApiKey: string;
    useWebSearch: boolean;
    tavilySearchDepth: "basic" | "advanced" | "fast" | "ultra-fast";
    tavilyMaxResults: number;
  };
  type BedrockConverseResponse = {
    text: string;
    stop_reason?: string;
    input_tokens?: number;
    output_tokens?: number;
    total_tokens?: number;
    used_web_search?: boolean;
  };
  type AiSearchStatus = "idle" | "checking" | "searching" | "completed" | "error";
  type AiSearchStatusPayload = {
    conversation_id?: string;
    status: AiSearchStatus;
    query?: string;
    result_count?: number;
    error?: string;
  };
  type RegexFlag = "g" | "i" | "m" | "s" | "u" | "y";
  type RegexMatchResult = {
    index: number;
    end: number;
    text: string;
    groups: string[];
  };
  type TextCounterStats = {
    jsLength: number;
    graphemeCount: number;
    utf8Bytes: number;
    lineCount: number;
    wordCount: number;
    trimmedGraphemeCount: number;
    trimmedUtf8Bytes: number;
  };
  type HashAlgorithm = "SHA-1" | "SHA-256" | "SHA-384" | "SHA-512";
  type DiffRowKind = "same" | "added" | "removed" | "changed-old" | "changed-new";
  type DiffRow = {
    kind: DiffRowKind;
    oldLine?: number;
    newLine?: number;
    text: string;
  };
  type RgbColor = {
    red: number;
    green: number;
    blue: number;
  };
  type HslColor = {
    hue: number;
    saturation: number;
    lightness: number;
  };
  type AiStreamDeltaPayload = {
    conversation_id?: string;
    message_id?: string;
    text: string;
  };

  let now = new Date();
  let timer: number | undefined;
  let uuids: string[] = [];
  let copiedUuid = "";
  let activeView: ActiveView = "menu";
  let pomodoroDurationSeconds = 25 * 60;
  let pomodoroRemainingSeconds = pomodoroDurationSeconds;
  let pomodoroEndsAt: number | undefined;
  let isPomodoroRunning = false;
  let hasSentPomodoroCompletionNotification = false;
  let selectedPaletteId = "tomato";
  let isTimerPaletteVisible = false;
  let customPomodoroMinutes: number | undefined = 30;
  let customPomodoroError = "";
  let copiedTextKey = "";
  let jsonInput = "";
  let jsonResult = "";
  let jsonError = "";
  let datetimeInput = "";
  let datetimeResult: DateTimeResult | undefined;
  let datetimeError = "";
  let base64Mode: Base64Mode = "text";
  let base64Input = "";
  let base64Result = "";
  let base64ResultSize = 0;
  let base64Error = "";
  let base64ImageName = "";
  let base64ImageMimeType = "";
  let base64ImageSize = 0;
  let base64ImageSource = "";
  let base64ImagePreviewUrl = "";
  let shouldIncludeBase64DataUrl = true;
  let urlInput = "";
  let urlResult = "";
  let urlError = "";
  let decodePlusAsSpace = true;
  let jwtInput = "";
  let jwtHeaderResult = "";
  let jwtPayloadResult = "";
  let jwtSignatureResult = "";
  let jwtError = "";
  let jwtClaimTimes: JwtClaimTime[] = [];
  let regexPattern = "";
  let regexTestText = "";
  let regexSelectedFlags: RegexFlag[] = ["g"];
  let regexMatches: RegexMatchResult[] = [];
  let regexError = "";
  let textCounterInput = "";
  let hashInput = "";
  let selectedHashAlgorithm: HashAlgorithm = "SHA-256";
  let hashResult = "";
  let hashError = "";
  let hashRequestId = 0;
  let diffOriginal = "";
  let diffChanged = "";
  let qrInput = "";
  let qrDataUrl = "";
  let qrError = "";
  let qrStatus = "";
  let qrRequestId = 0;
  let colorHexInput = "#2f7668";
  let colorRgb: RgbColor = { red: 47, green: 118, blue: 104 };
  let colorHsl: HslColor = { hue: 168, saturation: 43, lightness: 32 };
  let colorError = "";
  let dailyMemo = "";
  let memoDateKey = formatMemoDateKey(new Date());
  let memoStatus = "";
  let isMemoDirty = false;
  let aiChatSettings: AiChatSettings = {
    apiKey: "",
    region: "us-east-1",
    modelId: "us.anthropic.claude-3-5-haiku-20241022-v1:0",
    temperature: 0.7,
    maxTokens: 1200,
    tavilyApiKey: "",
    useWebSearch: false,
    tavilySearchDepth: "basic",
    tavilyMaxResults: 5,
  };
  let aiConversations: AiConversation[] = [];
  let activeAiConversationId = "";
  let aiChatInput = "";
  let aiChatError = "";
  let aiSettingsMessage = "";
  let isAiChatSending = false;
  let lastAiUsageSummary = "";
  let aiSearchStatus: AiSearchStatus = "idle";
  let aiSearchStatusText = "";
  let aiSearchConversationId = "";
  let unlistenAiSearchStatus: UnlistenFn | undefined;
  let unlistenAiStreamDelta: UnlistenFn | undefined;

  const maxDialSeconds = 60 * 60;
  const minPomodoroMinutes = 1;
  const maxPomodoroMinutes = 180;
  const maxBase64ImageBytes = 10 * 1024 * 1024;
  const timerPaletteStorageKey = "my-tools:pomodoro-timer-palette";
  const aiChatSettingsStorageKey = "my-tools:ai-chat-settings";
  const aiChatConversationsStorageKey = "my-tools:ai-chat-conversations";
  const aiChatMaxSavedConversations = 24;
  const dailyMemoStorageKey = "my-tools:daily-memos";
  const hashAlgorithms: HashAlgorithm[] = ["SHA-256", "SHA-1", "SHA-384", "SHA-512"];
  const regexFlagOptions: Array<{ flag: RegexFlag; label: string }> = [
    { flag: "g", label: "global" },
    { flag: "i", label: "ignore case" },
    { flag: "m", label: "multiline" },
    { flag: "s", label: "dotAll" },
    { flag: "u", label: "unicode" },
    { flag: "y", label: "sticky" },
  ];

  const tools: Array<{
    id: ToolId;
    label: string;
    icon: string;
    accent: string;
    image: string;
  }> = [
    { id: "clock", label: "時計", icon: "25", accent: "#d84a3a", image: clockMenuImage },
    { id: "uuid", label: "UUID", icon: "ID", accent: "#3d7fb8", image: uuidMenuImage },
    { id: "json", label: "JSON", icon: "{}", accent: "#4f8f74", image: jsonMenuImage },
    { id: "datetime", label: "日時変換", icon: "TS", accent: "#875aa6", image: datetimeMenuImage },
    { id: "base64", label: "Base64", icon: "64", accent: "#c27a22", image: base64MenuImage },
    { id: "url", label: "URL", icon: "%", accent: "#286f7a", image: urlMenuImage },
    { id: "jwt", label: "JWT", icon: "JWT", accent: "#7b5a2e", image: jwtMenuImage },
    { id: "regex", label: "Regex", icon: ".*", accent: "#6c5db8", image: regexMenuImage },
    { id: "text-counter", label: "文字数", icon: "字", accent: "#2f7668", image: textCounterMenuImage },
    { id: "hash", label: "ハッシュ", icon: "#", accent: "#4f62a8", image: hashMenuImage },
    { id: "diff", label: "Diff", icon: "±", accent: "#64707d", image: diffMenuImage },
    { id: "qr", label: "QR", icon: "QR", accent: "#2d7584", image: qrMenuImage },
    { id: "color", label: "色変換", icon: "●", accent: "#c46a5a", image: colorMenuImage },
    { id: "memo", label: "メモ", icon: "✎", accent: "#b7791f", image: memoMenuImage },
    { id: "ai-chat", label: "AIチャット", icon: "AI", accent: "#2f7668", image: aiChatMenuImage },
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
  const utcDateTimeFormatter = new Intl.DateTimeFormat("ja-JP", {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    timeZone: "UTC",
    timeZoneName: "short",
  });
  const jstDateTimeFormatter = new Intl.DateTimeFormat("ja-JP", {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    timeZone: "Asia/Tokyo",
    timeZoneName: "short",
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
  $: if (base64Mode === "image") {
    base64Result =
      base64ImageSource === ""
        ? ""
        : shouldIncludeBase64DataUrl
          ? `data:${base64ImageMimeType};base64,${base64ImageSource}`
          : base64ImageSource;
  }
  $: base64ResultSize = new TextEncoder().encode(base64Result).length;
  $: activeAiConversation =
    aiConversations.find((conversation) => conversation.id === activeAiConversationId) ??
    aiConversations[0];
  $: hasAiChatSettings =
    aiChatSettings.apiKey.trim() !== "" &&
    aiChatSettings.region.trim() !== "" &&
    aiChatSettings.modelId.trim() !== "";
  $: regexFlags = regexSelectedFlags.join("");
  $: textCounterStats = buildTextCounterStats(textCounterInput);
  $: void generateHash(hashInput, selectedHashAlgorithm);
  $: diffRows = buildDiffRows(diffOriginal, diffChanged);
  $: diffSummary = buildDiffSummary(diffRows);
  $: void generateQrCode(qrInput);
  $: colorPreview = colorError === "" ? colorHexInput : "#ffffff";

  function createUuid() {
    const uuid = crypto.randomUUID();
    uuids = [uuid, ...uuids].slice(0, 8);
    copiedUuid = "";
  }

  function openTool(toolId: ToolId) {
    activeView = toolId;
  }

  function openMenu() {
    activeView = "menu";
  }

  function clearUuids() {
    uuids = [];
    copiedUuid = "";
  }

  async function copyUuid(uuid: string) {
    await copyText(uuid, `uuid:${uuid}`);
    copiedUuid = uuid;
  }

  async function copyText(text: string, key: string) {
    await navigator.clipboard.writeText(text);
    copiedTextKey = key;
  }

  function parseJsonInput() {
    const source = jsonInput.trim();

    if (source === "") {
      jsonError = "";
      jsonResult = "";
      return undefined;
    }

    try {
      jsonError = "";
      return JSON.parse(source) as unknown;
    } catch (error) {
      jsonError =
        error instanceof Error
          ? `JSON として読み取れません: ${error.message}`
          : "JSON として読み取れません。";
      return undefined;
    }
  }

  function formatJson() {
    const parsedJson = parseJsonInput();

    if (jsonInput.trim() === "" || jsonError) {
      return;
    }

    jsonResult = JSON.stringify(parsedJson, null, 2) ?? "";
  }

  function minifyJson() {
    const parsedJson = parseJsonInput();

    if (jsonInput.trim() === "" || jsonError) {
      return;
    }

    jsonResult = JSON.stringify(parsedJson) ?? "";
  }

  function clearJson() {
    jsonInput = "";
    jsonResult = "";
    jsonError = "";
  }

  function convertDateTime() {
    const source = datetimeInput.trim();

    if (source === "") {
      datetimeError = "";
      datetimeResult = undefined;
      return;
    }

    const date = parseDateTimeInput(source);

    if (date === undefined) {
      datetimeError =
        "日時として読み取れません。Unix 秒、Unix ミリ秒、または ISO 形式の日時を入力してください。";
      return;
    }

    datetimeError = "";
    datetimeResult = buildDateTimeResult(date);
  }

  function useCurrentDateTime() {
    const date = new Date();
    datetimeInput = date.toISOString();
    datetimeError = "";
    datetimeResult = buildDateTimeResult(date);
  }

  function clearDateTime() {
    datetimeInput = "";
    datetimeResult = undefined;
    datetimeError = "";
  }

  function parseDateTimeInput(source: string) {
    if (/^[+-]?\d+(\.\d+)?$/.test(source)) {
      const numericValue = Number(source);

      if (!Number.isFinite(numericValue)) {
        return undefined;
      }

      const milliseconds =
        Math.abs(numericValue) < 100_000_000_000
          ? numericValue * 1000
          : numericValue;
      const date = new Date(milliseconds);

      return Number.isNaN(date.getTime()) ? undefined : date;
    }

    const date = new Date(source);

    return Number.isNaN(date.getTime()) ? undefined : date;
  }

  function buildDateTimeResult(date: Date): DateTimeResult {
    const milliseconds = date.getTime();

    return {
      unixSeconds: String(Math.floor(milliseconds / 1000)),
      unixMilliseconds: String(milliseconds),
      iso: date.toISOString(),
      utc: utcDateTimeFormatter.format(date),
      jst: jstDateTimeFormatter.format(date),
    };
  }

  function encodeBase64() {
    if (base64Input === "") {
      base64Error = "";
      base64Result = "";
      return;
    }

    base64Error = "";
    base64Result = bytesToBase64(new TextEncoder().encode(base64Input));
  }

  function decodeBase64() {
    const source = base64Input.replace(/\s+/g, "");

    if (source === "") {
      base64Error = "";
      base64Result = "";
      return;
    }

    try {
      base64Error = "";
      base64Result = new TextDecoder("utf-8", { fatal: true }).decode(
        base64ToBytes(source),
      );
    } catch {
      base64Error = "Base64 として読み取れません。文字列やパディングを確認してください。";
    }
  }

  function clearBase64() {
    base64Input = "";
    base64Result = "";
    base64Error = "";
    clearBase64Image();
  }

  function switchBase64Mode(mode: Base64Mode) {
    base64Mode = mode;
    base64Error = "";
    base64Result = mode === "image" ? buildImageBase64Result() : "";
    copiedTextKey = "";
  }

  async function handleBase64ImageInput(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];

    if (file === undefined) {
      return;
    }

    await convertImageFileToBase64(file);
    input.value = "";
  }

  async function handleBase64ImageDrop(event: DragEvent) {
    event.preventDefault();

    const file = event.dataTransfer?.files[0];

    if (file === undefined) {
      return;
    }

    await convertImageFileToBase64(file);
  }

  async function convertImageFileToBase64(file: File) {
    if (!file.type.startsWith("image/")) {
      base64Error = "画像ファイルを選択してください。";
      return;
    }

    if (file.size > maxBase64ImageBytes) {
      base64Error = `画像サイズは ${formatFileSize(maxBase64ImageBytes)} までにしてください。`;
      return;
    }

    try {
      const bytes = new Uint8Array(await file.arrayBuffer());

      clearBase64ImagePreviewUrl();
      base64ImageName = file.name;
      base64ImageMimeType = file.type;
      base64ImageSize = file.size;
      base64ImageSource = bytesToBase64(bytes);
      base64ImagePreviewUrl = URL.createObjectURL(file);
      base64Result = buildImageBase64Result();
      base64Error = "";
      copiedTextKey = "";
    } catch {
      base64Error = "画像を読み取れませんでした。別のファイルで確認してください。";
    }
  }

  function buildImageBase64Result() {
    if (base64ImageSource === "") {
      return "";
    }

    if (!shouldIncludeBase64DataUrl) {
      return base64ImageSource;
    }

    return `data:${base64ImageMimeType};base64,${base64ImageSource}`;
  }

  function clearBase64Image() {
    clearBase64ImagePreviewUrl();
    base64ImageName = "";
    base64ImageMimeType = "";
    base64ImageSize = 0;
    base64ImageSource = "";
  }

  function clearBase64ImagePreviewUrl() {
    if (base64ImagePreviewUrl !== "") {
      URL.revokeObjectURL(base64ImagePreviewUrl);
      base64ImagePreviewUrl = "";
    }
  }

  function formatFileSize(bytes: number) {
    if (bytes < 1024) {
      return `${bytes} B`;
    }

    const units = ["KB", "MB", "GB"];
    let size = bytes / 1024;
    let unitIndex = 0;

    while (size >= 1024 && unitIndex < units.length - 1) {
      size /= 1024;
      unitIndex += 1;
    }

    return `${size.toFixed(size >= 10 ? 1 : 2)} ${units[unitIndex]}`;
  }

  function encodeUrl() {
    if (urlInput === "") {
      urlError = "";
      urlResult = "";
      return;
    }

    urlError = "";
    urlResult = encodeURIComponent(urlInput);
  }

  function decodeUrl() {
    if (urlInput === "") {
      urlError = "";
      urlResult = "";
      return;
    }

    try {
      urlError = "";
      urlResult = decodeURIComponent(
        decodePlusAsSpace ? urlInput.replace(/\+/g, " ") : urlInput,
      );
    } catch {
      urlError = "URL エンコードとして読み取れません。% の後ろが正しい形式か確認してください。";
    }
  }

  function clearUrl() {
    urlInput = "";
    urlResult = "";
    urlError = "";
  }

  function decodeJwt() {
    const source = jwtInput.trim();

    if (source === "") {
      clearJwtResult();
      return;
    }

    const segments = source.split(".");

    if (segments.length !== 3 || segments[0] === "" || segments[1] === "") {
      jwtError = "JWT は header.payload.signature の 3 セグメント形式で入力してください。";
      return;
    }

    try {
      const header = decodeBase64UrlJson(segments[0], "ヘッダー");
      const payload = decodeBase64UrlJson(segments[1], "ペイロード");

      jwtHeaderResult = JSON.stringify(header, null, 2);
      jwtPayloadResult = JSON.stringify(payload, null, 2);
      jwtSignatureResult = segments[2];
      jwtClaimTimes = buildJwtClaimTimes(payload);
      jwtError = "";
    } catch (error) {
      jwtError = error instanceof Error ? error.message : "JWT を読み取れません。";
    }
  }

  function clearJwt() {
    jwtInput = "";
    clearJwtResult();
  }

  function clearJwtResult() {
    jwtHeaderResult = "";
    jwtPayloadResult = "";
    jwtSignatureResult = "";
    jwtError = "";
    jwtClaimTimes = [];
  }

  function runRegexTester() {
    if (regexPattern === "" || regexTestText === "") {
      regexError = "";
      regexMatches = [];
      return;
    }

    try {
      const expression = new RegExp(regexPattern, regexFlags);
      const matches: RegexMatchResult[] = [];

      if (regexFlags.includes("g") || regexFlags.includes("y")) {
        let match: RegExpExecArray | null;

        while ((match = expression.exec(regexTestText)) !== null) {
          matches.push(buildRegexMatch(match));

          if (match[0] === "") {
            expression.lastIndex += 1;
          }
        }
      } else {
        const match = expression.exec(regexTestText);

        if (match !== null) {
          matches.push(buildRegexMatch(match));
        }
      }

      regexError = "";
      regexMatches = matches;
    } catch (error) {
      regexError =
        error instanceof Error
          ? `正規表現として読み取れません: ${error.message}`
          : "正規表現として読み取れません。";
    }
  }

  function buildRegexMatch(match: RegExpExecArray): RegexMatchResult {
    return {
      index: match.index,
      end: match.index + match[0].length,
      text: match[0],
      groups: match.slice(1).map((group) => group ?? ""),
    };
  }

  function clearRegexTester() {
    regexPattern = "";
    regexTestText = "";
    regexSelectedFlags = ["g"];
    regexMatches = [];
    regexError = "";
  }

  function formatRegexMatchesForCopy() {
    if (regexMatches.length === 0) {
      return "マッチなし";
    }

    return regexMatches
      .map((match, index) => {
        const groups =
          match.groups.length === 0
            ? ""
            : `\n  captures: ${match.groups
                .map((group, groupIndex) => `${groupIndex + 1}=${group}`)
                .join(", ")}`;

        return `${index + 1}. [${match.index}, ${match.end}) ${match.text}${groups}`;
      })
      .join("\n");
  }

  function buildTextCounterStats(source: string): TextCounterStats {
    const trimmed = source.trim();

    return {
      jsLength: source.length,
      graphemeCount: countGraphemes(source),
      utf8Bytes: new TextEncoder().encode(source).length,
      lineCount: source === "" ? 0 : source.split(/\r\n|\r|\n/).length,
      wordCount: source.trim() === "" ? 0 : source.trim().split(/\s+/u).length,
      trimmedGraphemeCount: countGraphemes(trimmed),
      trimmedUtf8Bytes: new TextEncoder().encode(trimmed).length,
    };
  }

  function countGraphemes(source: string) {
    const segmenter = (
      Intl as typeof Intl & {
        Segmenter?: new (
          locale?: string,
          options?: { granularity: "grapheme" },
        ) => { segment: (text: string) => Iterable<unknown> };
      }
    ).Segmenter;

    if (segmenter === undefined) {
      return Array.from(source).length;
    }

    return Array.from(new segmenter("ja-JP", { granularity: "grapheme" }).segment(source))
      .length;
  }

  function clearTextCounter() {
    textCounterInput = "";
  }

  function formatTextCounterStatsForCopy() {
    return [
      `表示上の文字数: ${textCounterStats.graphemeCount}`,
      `JavaScript length: ${textCounterStats.jsLength}`,
      `UTF-8 バイト数: ${textCounterStats.utf8Bytes}`,
      `行数: ${textCounterStats.lineCount}`,
      `単語数: ${textCounterStats.wordCount}`,
      `トリム後文字数: ${textCounterStats.trimmedGraphemeCount}`,
      `トリム後 UTF-8 バイト数: ${textCounterStats.trimmedUtf8Bytes}`,
    ].join("\n");
  }

  async function generateHash(source: string, algorithm: HashAlgorithm) {
    const requestId = ++hashRequestId;

    try {
      const digest = await crypto.subtle.digest(
        algorithm,
        new TextEncoder().encode(source),
      );

      if (requestId !== hashRequestId) {
        return;
      }

      hashResult = bytesToHex(new Uint8Array(digest));
      hashError = "";
    } catch {
      if (requestId === hashRequestId) {
        hashResult = "";
        hashError = "この環境では選択したハッシュを生成できませんでした。";
      }
    }
  }

  function bytesToHex(bytes: Uint8Array) {
    return Array.from(bytes, (byte) => byte.toString(16).padStart(2, "0")).join("");
  }

  function clearHash() {
    hashInput = "";
    hashError = "";
  }

  function buildDiffRows(original: string, changed: string): DiffRow[] {
    const originalLines = original === "" ? [] : original.split(/\r\n|\r|\n/);
    const changedLines = changed === "" ? [] : changed.split(/\r\n|\r|\n/);
    const table = buildLcsTable(originalLines, changedLines);
    const rows: DiffRow[] = [];
    let originalIndex = 0;
    let changedIndex = 0;

    while (originalIndex < originalLines.length || changedIndex < changedLines.length) {
      if (
        originalIndex < originalLines.length &&
        changedIndex < changedLines.length &&
        originalLines[originalIndex] === changedLines[changedIndex]
      ) {
        rows.push({
          kind: "same",
          oldLine: originalIndex + 1,
          newLine: changedIndex + 1,
          text: originalLines[originalIndex],
        });
        originalIndex += 1;
        changedIndex += 1;
        continue;
      }

      if (
        changedIndex < changedLines.length &&
        (originalIndex === originalLines.length ||
          table[originalIndex][changedIndex + 1] >=
            table[originalIndex + 1][changedIndex])
      ) {
        rows.push({
          kind: "added",
          newLine: changedIndex + 1,
          text: changedLines[changedIndex],
        });
        changedIndex += 1;
      } else if (originalIndex < originalLines.length) {
        rows.push({
          kind: "removed",
          oldLine: originalIndex + 1,
          text: originalLines[originalIndex],
        });
        originalIndex += 1;
      }
    }

    return markChangedRows(rows);
  }

  function buildLcsTable(left: string[], right: string[]) {
    const table = Array.from({ length: left.length + 1 }, () =>
      Array.from({ length: right.length + 1 }, () => 0),
    );

    for (let leftIndex = left.length - 1; leftIndex >= 0; leftIndex -= 1) {
      for (let rightIndex = right.length - 1; rightIndex >= 0; rightIndex -= 1) {
        table[leftIndex][rightIndex] =
          left[leftIndex] === right[rightIndex]
            ? table[leftIndex + 1][rightIndex + 1] + 1
            : Math.max(
                table[leftIndex + 1][rightIndex],
                table[leftIndex][rightIndex + 1],
              );
      }
    }

    return table;
  }

  function markChangedRows(rows: DiffRow[]) {
    return rows.map((row, index) => {
      if (row.kind === "removed" && rows[index + 1]?.kind === "added") {
        return { ...row, kind: "changed-old" as const };
      }

      if (row.kind === "added" && rows[index - 1]?.kind === "removed") {
        return { ...row, kind: "changed-new" as const };
      }

      return row;
    });
  }

  function buildDiffSummary(rows: DiffRow[]) {
    const added = rows.filter((row) => row.kind === "added" || row.kind === "changed-new")
      .length;
    const removed = rows.filter(
      (row) => row.kind === "removed" || row.kind === "changed-old",
    ).length;
    const changed = Math.min(
      rows.filter((row) => row.kind === "changed-old").length,
      rows.filter((row) => row.kind === "changed-new").length,
    );

    return { added, removed, changed };
  }

  function clearDiff() {
    diffOriginal = "";
    diffChanged = "";
  }

  function formatDiffForCopy() {
    if (diffRows.length === 0) {
      return "差分なし";
    }

    return diffRows
      .map((row) => {
        const prefix =
          row.kind === "added" || row.kind === "changed-new"
            ? "+"
            : row.kind === "removed" || row.kind === "changed-old"
              ? "-"
              : " ";

        return `${prefix} ${row.text}`;
      })
      .join("\n");
  }

  async function generateQrCode(source: string) {
    const requestId = ++qrRequestId;
    const trimmed = source.trim();

    if (trimmed === "") {
      qrDataUrl = "";
      qrError = "";
      return;
    }

    try {
      const dataUrl = await QRCode.toDataURL(source, {
        errorCorrectionLevel: "M",
        margin: 3,
        width: 320,
        color: {
          dark: "#202832",
          light: "#ffffff",
        },
      });

      if (requestId === qrRequestId) {
        qrDataUrl = dataUrl;
        qrError = "";
        qrStatus = "";
      }
    } catch {
      if (requestId === qrRequestId) {
        qrDataUrl = "";
        qrStatus = "";
        qrError = "QRコードを生成できませんでした。入力を短くして再度お試しください。";
      }
    }
  }

  async function readQrBlob() {
    const response = await fetch(qrDataUrl);
    return response.blob();
  }

  async function copyQrImage() {
    if (qrDataUrl === "") {
      return;
    }

    try {
      const ClipboardItemConstructor = window.ClipboardItem;
      if (navigator.clipboard?.write === undefined || ClipboardItemConstructor === undefined) {
        throw new Error("Image clipboard is not supported.");
      }

      const blob = await readQrBlob();
      const mimeType = blob.type || "image/png";
      await navigator.clipboard.write([
        new ClipboardItemConstructor({
          [mimeType]: blob,
        }),
      ]);
      copiedTextKey = "qr:image";
      qrError = "";
      qrStatus = "QR画像をクリップボードにコピーしました。";
    } catch {
      qrStatus = "";
      qrError = "画像としてコピーできませんでした。保存ボタンをご利用ください。";
    }
  }

  function downloadQrCode() {
    if (qrDataUrl === "") {
      return;
    }

    const link = document.createElement("a");
    link.href = qrDataUrl;
    link.download = "qr-code.png";
    document.body.append(link);
    link.click();
    link.remove();
    qrError = "";
    qrStatus = "QR画像をPNGとして保存しました。";
  }

  function clearQr() {
    qrInput = "";
    qrDataUrl = "";
    qrError = "";
    qrStatus = "";
  }

  function applyHexColor() {
    const parsed = parseHexColor(colorHexInput);

    if (parsed === undefined) {
      colorError = "HEX は #RGB または #RRGGBB の形式で入力してください。";
      return;
    }

    setColorFromRgb(parsed);
  }

  function applyRgbColor() {
    if (!isRgbColor(colorRgb)) {
      colorError = "RGB は 0〜255 の整数で入力してください。";
      return;
    }

    setColorFromRgb(colorRgb);
  }

  function applyHslColor() {
    if (!isHslColor(colorHsl)) {
      colorError = "HSL は H=0〜360、S/L=0〜100 の範囲で入力してください。";
      return;
    }

    setColorFromRgb(hslToRgb(colorHsl));
  }

  function setColorFromPicker(value: string) {
    colorHexInput = value;
    applyHexColor();
  }

  function setColorFromRgb(rgb: RgbColor) {
    const normalizedRgb = {
      red: Math.round(rgb.red),
      green: Math.round(rgb.green),
      blue: Math.round(rgb.blue),
    };

    colorRgb = normalizedRgb;
    colorHsl = rgbToHsl(normalizedRgb);
    colorHexInput = rgbToHex(normalizedRgb);
    colorError = "";
  }

  function parseHexColor(source: string): RgbColor | undefined {
    const normalized = source.trim().replace(/^#/, "");

    if (!/^[0-9a-fA-F]{3}$|^[0-9a-fA-F]{6}$/.test(normalized)) {
      return undefined;
    }

    const hex =
      normalized.length === 3
        ? normalized
            .split("")
            .map((character) => character + character)
            .join("")
        : normalized;

    return {
      red: Number.parseInt(hex.slice(0, 2), 16),
      green: Number.parseInt(hex.slice(2, 4), 16),
      blue: Number.parseInt(hex.slice(4, 6), 16),
    };
  }

  function isRgbColor(rgb: RgbColor) {
    return [rgb.red, rgb.green, rgb.blue].every(
      (value) => Number.isInteger(value) && value >= 0 && value <= 255,
    );
  }

  function isHslColor(hsl: HslColor) {
    return (
      Number.isFinite(hsl.hue) &&
      hsl.hue >= 0 &&
      hsl.hue <= 360 &&
      Number.isFinite(hsl.saturation) &&
      hsl.saturation >= 0 &&
      hsl.saturation <= 100 &&
      Number.isFinite(hsl.lightness) &&
      hsl.lightness >= 0 &&
      hsl.lightness <= 100
    );
  }

  function rgbToHex(rgb: RgbColor) {
    return `#${[rgb.red, rgb.green, rgb.blue]
      .map((value) => value.toString(16).padStart(2, "0"))
      .join("")}`;
  }

  function rgbToHsl(rgb: RgbColor): HslColor {
    const red = rgb.red / 255;
    const green = rgb.green / 255;
    const blue = rgb.blue / 255;
    const max = Math.max(red, green, blue);
    const min = Math.min(red, green, blue);
    const lightness = (max + min) / 2;
    const delta = max - min;

    if (delta === 0) {
      return { hue: 0, saturation: 0, lightness: Math.round(lightness * 100) };
    }

    const saturation =
      delta / (1 - Math.abs(2 * lightness - 1));
    let hue = 0;

    if (max === red) {
      hue = ((green - blue) / delta) % 6;
    } else if (max === green) {
      hue = (blue - red) / delta + 2;
    } else {
      hue = (red - green) / delta + 4;
    }

    return {
      hue: Math.round((hue * 60 + 360) % 360),
      saturation: Math.round(saturation * 100),
      lightness: Math.round(lightness * 100),
    };
  }

  function hslToRgb(hsl: HslColor): RgbColor {
    const saturation = hsl.saturation / 100;
    const lightness = hsl.lightness / 100;
    const chroma = (1 - Math.abs(2 * lightness - 1)) * saturation;
    const huePrime = hsl.hue / 60;
    const x = chroma * (1 - Math.abs((huePrime % 2) - 1));
    const match = lightness - chroma / 2;
    const [red, green, blue] =
      huePrime < 1
        ? [chroma, x, 0]
        : huePrime < 2
          ? [x, chroma, 0]
          : huePrime < 3
            ? [0, chroma, x]
            : huePrime < 4
              ? [0, x, chroma]
              : huePrime < 5
                ? [x, 0, chroma]
                : [chroma, 0, x];

    return {
      red: Math.round((red + match) * 255),
      green: Math.round((green + match) * 255),
      blue: Math.round((blue + match) * 255),
    };
  }

  function clearColor() {
    setColorFromRgb({ red: 47, green: 118, blue: 104 });
  }

  function formatMemoDateKey(date: Date) {
    const year = date.getFullYear();
    const month = String(date.getMonth() + 1).padStart(2, "0");
    const day = String(date.getDate()).padStart(2, "0");

    return `${year}-${month}-${day}`;
  }

  function formatMemoDateLabel(dateKey: string) {
    const date = new Date(`${dateKey}T00:00:00`);

    return Number.isNaN(date.getTime())
      ? dateKey
      : new Intl.DateTimeFormat("ja-JP", {
          year: "numeric",
          month: "long",
          day: "numeric",
          weekday: "long",
        }).format(date);
  }

  function loadDailyMemo() {
    const memos = readDailyMemos();
    dailyMemo = memos[memoDateKey] ?? "";
    memoStatus = dailyMemo === "" ? "未保存" : "保存済み";
    isMemoDirty = false;
  }

  function saveDailyMemo() {
    const memos = readDailyMemos();

    if (dailyMemo === "") {
      delete memos[memoDateKey];
    } else {
      memos[memoDateKey] = dailyMemo;
    }

    try {
      window.localStorage.setItem(dailyMemoStorageKey, JSON.stringify(memos));
      memoStatus = dailyMemo === "" ? "空のメモとして保存しました" : "保存しました";
      isMemoDirty = false;
    } catch {
      memoStatus = "保存できませんでした";
    }
  }

  function readDailyMemos(): Record<string, string> {
    try {
      const saved = window.localStorage.getItem(dailyMemoStorageKey);

      if (saved === null) {
        return {};
      }

      const parsed = JSON.parse(saved);

      if (typeof parsed !== "object" || parsed === null) {
        return {};
      }

      return Object.fromEntries(
        Object.entries(parsed).filter(
          (entry): entry is [string, string] =>
            typeof entry[0] === "string" && typeof entry[1] === "string",
        ),
      );
    } catch {
      return {};
    }
  }

  function handleMemoInput() {
    isMemoDirty = true;
    memoStatus = "未保存の変更があります";
  }

  function clearDailyMemo() {
    dailyMemo = "";
    saveDailyMemo();
  }

  function decodeBase64UrlJson(segment: string, label: string) {
    try {
      const text = decodeBase64UrlToText(segment);

      try {
        return JSON.parse(text) as Record<string, unknown>;
      } catch {
        throw new Error(`${label}を JSON として読み取れません。`);
      }
    } catch (error) {
      if (error instanceof Error && error.message.includes("JSON")) {
        throw error;
      }

      throw new Error(`${label}の Base64URL デコードに失敗しました。`);
    }
  }

  function decodeBase64UrlToText(segment: string) {
    const normalized = segment.replace(/-/g, "+").replace(/_/g, "/");

    return new TextDecoder("utf-8", { fatal: true }).decode(base64ToBytes(normalized));
  }

  function buildJwtClaimTimes(payload: Record<string, unknown>) {
    return ["exp", "iat", "nbf"]
      .filter((name) => typeof payload[name] === "number" && Number.isFinite(payload[name]))
      .map((name) => {
        const date = new Date((payload[name] as number) * 1000);

        return {
          name,
          unixSeconds: String(payload[name]),
          utc: utcDateTimeFormatter.format(date),
          jst: jstDateTimeFormatter.format(date),
        };
      });
  }

  function createAiConversation() {
    const nowIso = new Date().toISOString();
    const conversation: AiConversation = {
      id: crypto.randomUUID(),
      title: "新しいチャット",
      createdAt: nowIso,
      updatedAt: nowIso,
      messages: [],
    };

    aiConversations = [conversation, ...aiConversations].slice(
      0,
      aiChatMaxSavedConversations,
    );
    activeAiConversationId = conversation.id;
    aiChatInput = "";
    aiChatError = "";
    lastAiUsageSummary = "";
    resetAiSearchStatus();
    saveAiConversations();
  }

  function selectAiConversation(conversationId: string) {
    activeAiConversationId = conversationId;
    aiChatError = "";
    lastAiUsageSummary = "";
    resetAiSearchStatus();
  }

  function clearActiveAiConversation() {
    if (activeAiConversation === undefined) {
      createAiConversation();
      return;
    }

    const nowIso = new Date().toISOString();
    aiConversations = aiConversations.map((conversation) =>
      conversation.id === activeAiConversation.id
        ? {
            ...conversation,
            title: "新しいチャット",
            updatedAt: nowIso,
            messages: [],
          }
        : conversation,
    );
    aiChatInput = "";
    aiChatError = "";
    lastAiUsageSummary = "";
    resetAiSearchStatus();
    saveAiConversations();
  }

  function deleteActiveAiConversation() {
    if (activeAiConversation === undefined) {
      return;
    }

    aiConversations = aiConversations.filter(
      (conversation) => conversation.id !== activeAiConversation.id,
    );

    if (aiConversations.length === 0) {
      createAiConversation();
      return;
    }

    activeAiConversationId = aiConversations[0].id;
    aiChatInput = "";
    aiChatError = "";
    lastAiUsageSummary = "";
    resetAiSearchStatus();
    saveAiConversations();
  }

  async function sendAiChatMessage() {
    const userText = aiChatInput.trim();

    if (userText === "") {
      return;
    }

    if (!hasAiChatSettings) {
      aiChatError = "Bedrock APIキー、リージョン、モデルIDを設定してください。";
      return;
    }

    if (activeAiConversation === undefined) {
      createAiConversation();
    }

    const conversationId = activeAiConversationId;
    const nowIso = new Date().toISOString();
    const userMessage: AiChatMessage = {
      id: crypto.randomUUID(),
      role: "user",
      text: userText,
      createdAt: nowIso,
    };
    const assistantMessage: AiChatMessage = {
      id: crypto.randomUUID(),
      role: "assistant",
      text: "",
      createdAt: nowIso,
    };

    aiChatInput = "";
    aiChatError = "";
    lastAiUsageSummary = "";
    startAiSearchStatus(conversationId);
    isAiChatSending = true;
    appendAiMessage(conversationId, userMessage);
    appendAiMessage(conversationId, assistantMessage);

    try {
      const conversation = aiConversations.find(
        (conversation) => conversation.id === conversationId,
      );
      const messages = (conversation?.messages ?? [])
        .filter((message) => message.text.trim() !== "")
        .map((message) => ({
          role: message.role,
          content: [{ text: message.text }],
        }));
      const response = await invoke<BedrockConverseResponse>("bedrock_converse", {
        request: {
          api_key: aiChatSettings.apiKey,
          region: aiChatSettings.region,
          model_id: aiChatSettings.modelId,
          temperature: normalizeTemperature(aiChatSettings.temperature),
          max_tokens: normalizeMaxTokens(aiChatSettings.maxTokens),
          tavily_api_key: aiChatSettings.tavilyApiKey,
          use_web_search: aiChatSettings.useWebSearch,
          tavily_search_depth: aiChatSettings.tavilySearchDepth,
          tavily_max_results: normalizeTavilyMaxResults(aiChatSettings.tavilyMaxResults),
          conversation_id: conversationId,
          stream_message_id: assistantMessage.id,
          messages,
        },
      });

      replaceAiMessageText(
        conversationId,
        assistantMessage.id,
        response.text || "応答本文が空でした。",
      );
      lastAiUsageSummary = formatAiUsageSummary(response);
    } catch (error) {
      removeAiMessage(conversationId, assistantMessage.id);
      aiChatError =
        typeof error === "string"
          ? error
          : error instanceof Error
            ? error.message
            : "Bedrock への送信に失敗しました。設定とモデル利用権限を確認してください。";
    } finally {
      if (aiSearchStatus === "checking") {
        resetAiSearchStatus();
      }

      isAiChatSending = false;
    }
  }

  function startAiSearchStatus(conversationId: string) {
    aiSearchConversationId = conversationId;

    if (aiChatSettings.useWebSearch && aiChatSettings.tavilyApiKey.trim() !== "") {
      aiSearchStatus = "checking";
      aiSearchStatusText = "Web検索が必要か確認しています。";
    } else {
      resetAiSearchStatus();
    }
  }

  function resetAiSearchStatus() {
    aiSearchStatus = "idle";
    aiSearchStatusText = "";
    aiSearchConversationId = "";
  }

  function handleAiSearchStatus(payload: AiSearchStatusPayload) {
    if (
      payload.conversation_id !== undefined &&
      payload.conversation_id !== "" &&
      payload.conversation_id !== activeAiConversationId
    ) {
      return;
    }

    aiSearchConversationId = payload.conversation_id ?? activeAiConversationId;
    aiSearchStatus = payload.status;

    if (payload.status === "searching") {
      aiSearchStatusText = payload.query
        ? `Tavilyで検索しています: ${payload.query}`
        : "Tavilyで検索しています。";
      return;
    }

    if (payload.status === "completed") {
      aiSearchStatusText =
        payload.result_count === undefined
          ? "検索結果を取得しました。"
          : `検索結果を取得しました（${payload.result_count}件）。`;
      return;
    }

    if (payload.status === "error") {
      aiSearchStatusText = payload.error
        ? `検索に失敗しました: ${payload.error}`
        : "検索に失敗しました。";
      return;
    }

    aiSearchStatusText = "";
  }

  function appendAiMessage(conversationId: string, message: AiChatMessage) {
    const nowIso = new Date().toISOString();

    aiConversations = aiConversations.map((conversation) => {
      if (conversation.id !== conversationId) {
        return conversation;
      }

      const messages = [...conversation.messages, message];
      const firstUserMessage = messages.find((message) => message.role === "user");

      return {
        ...conversation,
        title:
          firstUserMessage === undefined
            ? conversation.title
            : buildAiConversationTitle(firstUserMessage.text),
        updatedAt: nowIso,
        messages,
      };
    });
    saveAiConversations();
  }

  function updateAiMessageText(
    conversationId: string,
    messageId: string,
    update: (text: string) => string,
  ) {
    aiConversations = aiConversations.map((conversation) => {
      if (conversation.id !== conversationId) {
        return conversation;
      }

      return {
        ...conversation,
        updatedAt: new Date().toISOString(),
        messages: conversation.messages.map((message) =>
          message.id === messageId ? { ...message, text: update(message.text) } : message,
        ),
      };
    });
    saveAiConversations();
  }

  function replaceAiMessageText(conversationId: string, messageId: string, text: string) {
    updateAiMessageText(conversationId, messageId, () => text);
  }

  function removeAiMessage(conversationId: string, messageId: string) {
    aiConversations = aiConversations.map((conversation) => {
      if (conversation.id !== conversationId) {
        return conversation;
      }

      return {
        ...conversation,
        messages: conversation.messages.filter((message) => message.id !== messageId),
      };
    });
    saveAiConversations();
  }

  function handleAiStreamDelta(payload: AiStreamDeltaPayload) {
    if (
      payload.conversation_id === undefined ||
      payload.message_id === undefined ||
      payload.conversation_id !== activeAiConversationId
    ) {
      return;
    }

    updateAiMessageText(payload.conversation_id, payload.message_id, (text) => {
      return `${text}${payload.text}`;
    });
  }

  function buildAiConversationTitle(text: string) {
    const normalized = text.replace(/\s+/g, " ").trim();

    if (normalized === "") {
      return "新しいチャット";
    }

    return normalized.length > 28 ? `${normalized.slice(0, 28)}…` : normalized;
  }

  function saveAiChatSettings() {
    try {
      window.localStorage.setItem(
        aiChatSettingsStorageKey,
        JSON.stringify(aiChatSettings),
      );
      aiSettingsMessage = "設定を保存しました。";
    } catch {
      aiSettingsMessage = "設定を保存できませんでした。";
    }
  }

  function loadAiChatSettings() {
    try {
      const savedSettings = window.localStorage.getItem(aiChatSettingsStorageKey);

      if (savedSettings === null) {
        return;
      }

      const parsed = JSON.parse(savedSettings) as Partial<AiChatSettings>;

      aiChatSettings = {
        apiKey: typeof parsed.apiKey === "string" ? parsed.apiKey : "",
        region: typeof parsed.region === "string" ? parsed.region : "us-east-1",
        modelId:
          typeof parsed.modelId === "string"
            ? parsed.modelId
            : "us.anthropic.claude-3-5-haiku-20241022-v1:0",
        temperature:
          typeof parsed.temperature === "number" ? parsed.temperature : 0.7,
        maxTokens: typeof parsed.maxTokens === "number" ? parsed.maxTokens : 1200,
        tavilyApiKey:
          typeof parsed.tavilyApiKey === "string" ? parsed.tavilyApiKey : "",
        useWebSearch:
          typeof parsed.useWebSearch === "boolean" ? parsed.useWebSearch : false,
        tavilySearchDepth: isTavilySearchDepth(parsed.tavilySearchDepth)
          ? parsed.tavilySearchDepth
          : "basic",
        tavilyMaxResults:
          typeof parsed.tavilyMaxResults === "number" ? parsed.tavilyMaxResults : 5,
      };
    } catch {
      aiSettingsMessage = "保存済みのAI設定を読み取れませんでした。";
    }
  }

  function saveAiConversations() {
    try {
      window.localStorage.setItem(
        aiChatConversationsStorageKey,
        JSON.stringify(aiConversations.slice(0, aiChatMaxSavedConversations)),
      );
    } catch {
      aiChatError = "会話履歴を保存できませんでした。";
    }
  }

  function loadAiConversations() {
    try {
      const savedConversations = window.localStorage.getItem(
        aiChatConversationsStorageKey,
      );

      if (savedConversations !== null) {
        const parsed = JSON.parse(savedConversations);

        if (Array.isArray(parsed)) {
          aiConversations = parsed
            .filter(isAiConversation)
            .slice(0, aiChatMaxSavedConversations);
        }
      }
    } catch {
      aiChatError = "保存済みの会話履歴を読み取れませんでした。";
    }

    if (aiConversations.length === 0) {
      createAiConversation();
    } else {
      activeAiConversationId = aiConversations[0].id;
    }
  }

  function isAiConversation(value: unknown): value is AiConversation {
    if (typeof value !== "object" || value === null) {
      return false;
    }

    const conversation = value as Partial<AiConversation>;

    return (
      typeof conversation.id === "string" &&
      typeof conversation.title === "string" &&
      typeof conversation.createdAt === "string" &&
      typeof conversation.updatedAt === "string" &&
      Array.isArray(conversation.messages) &&
      conversation.messages.every(isAiChatMessage)
    );
  }

  function isAiChatMessage(value: unknown): value is AiChatMessage {
    if (typeof value !== "object" || value === null) {
      return false;
    }

    const message = value as Partial<AiChatMessage>;

    return (
      typeof message.id === "string" &&
      (message.role === "user" || message.role === "assistant") &&
      typeof message.text === "string" &&
      typeof message.createdAt === "string"
    );
  }

  function normalizeTemperature(value: number) {
    return Math.min(1, Math.max(0, Number.isFinite(value) ? value : 0.7));
  }

  function normalizeMaxTokens(value: number) {
    return Math.min(8192, Math.max(1, Math.trunc(Number.isFinite(value) ? value : 1200)));
  }

  function normalizeTavilyMaxResults(value: number) {
    return Math.min(10, Math.max(1, Math.trunc(Number.isFinite(value) ? value : 5)));
  }

  function isTavilySearchDepth(value: unknown): value is AiChatSettings["tavilySearchDepth"] {
    return (
      value === "basic" ||
      value === "advanced" ||
      value === "fast" ||
      value === "ultra-fast"
    );
  }

  function formatAiUsageSummary(response: BedrockConverseResponse) {
    const parts = [
      response.used_web_search ? "Web検索あり" : "",
      response.input_tokens === undefined ? "" : `入力 ${response.input_tokens}`,
      response.output_tokens === undefined ? "" : `出力 ${response.output_tokens}`,
      response.total_tokens === undefined ? "" : `合計 ${response.total_tokens}`,
    ].filter(Boolean);

    return parts.length === 0 ? "" : `${parts.join(" / ")} tokens`;
  }

  function formatChatTime(value: string) {
    const date = new Date(value);

    return Number.isNaN(date.getTime())
      ? ""
      : new Intl.DateTimeFormat("ja-JP", {
          month: "2-digit",
          day: "2-digit",
          hour: "2-digit",
          minute: "2-digit",
        }).format(date);
  }

  function bytesToBase64(bytes: Uint8Array) {
    const chunkSize = 0x8000;
    const chunks: string[] = [];

    for (let index = 0; index < bytes.length; index += chunkSize) {
      chunks.push(String.fromCharCode(...bytes.slice(index, index + chunkSize)));
    }

    return btoa(chunks.join(""));
  }

  function base64ToBytes(source: string) {
    const normalized = source.trim();

    if (!/^[A-Za-z0-9+/]*={0,2}$/.test(normalized) || normalized.length % 4 === 1) {
      throw new Error("Invalid Base64");
    }

    const padded = normalized.padEnd(Math.ceil(normalized.length / 4) * 4, "=");
    const binary = atob(padded);

    return Uint8Array.from(binary, (character) => character.charCodeAt(0));
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
    loadDailyMemo();
    loadAiChatSettings();
    loadAiConversations();

    void listen<AiSearchStatusPayload>("ai-chat-search-status", (event) => {
      handleAiSearchStatus(event.payload);
    }).then((unlisten) => {
      unlistenAiSearchStatus = unlisten;
    });
    void listen<AiStreamDeltaPayload>("ai-chat-stream-delta", (event) => {
      handleAiStreamDelta(event.payload);
    }).then((unlisten) => {
      unlistenAiStreamDelta = unlisten;
    });

    timer = window.setInterval(() => {
      now = new Date();
      const nextMemoDateKey = formatMemoDateKey(now);

      if (nextMemoDateKey !== memoDateKey) {
        memoDateKey = nextMemoDateKey;
        loadDailyMemo();
      }

      updatePomodoroRemaining();
    }, 250);

    void invoke<number>("current_timestamp").catch(() => undefined);
    createUuid();
  });

  onDestroy(() => {
    if (timer !== undefined) {
      window.clearInterval(timer);
    }

    clearBase64ImagePreviewUrl();
    unlistenAiSearchStatus?.();
    unlistenAiStreamDelta?.();
  });
</script>

<main class={activeView === "menu" ? "menu-shell" : "app-shell"}>
  {#if activeView === "menu"}
    <section class="menu-screen" aria-labelledby="menu-title">
      <div class="menu-header">
        <p class="section-label">TOOLS</p>
        <h1 id="menu-title">My Tools</h1>
      </div>

      <div class="menu-grid">
        {#each tools as tool}
          <button
            type="button"
            aria-label={tool.label}
            style={`--tool-accent: ${tool.accent};`}
            on:click={() => openTool(tool.id)}
          >
            <span class="menu-image-wrap" aria-hidden="true">
              <img src={tool.image} alt="" />
              <span class="dock-icon">{tool.icon}</span>
            </span>
            <span class="dock-label">{tool.label}</span>
          </button>
        {/each}
      </div>
    </section>
  {:else}
    <div class="tool-workspace">
      <div class="tool-topbar">
        <button type="button" class="back-button" on:click={openMenu}>
          <span aria-hidden="true">←</span>
          <span>メニュー</span>
        </button>
      </div>

      {#if activeView === "clock"}
        <section
          class="tool-panel clock-panel"
          aria-label="ポモドーロタイマー"
          style={`--timer-color: ${selectedPalette.color}; --timer-soft: ${selectedPalette.soft};`}
        >
        <div class="clock-info">
          <div>
            <p class="section-label">Pomodoro Clock</p>
            <div class="clock-title-row">
              <h1>ポモドーロ</h1>
              <button
                type="button"
                class="palette-toggle"
                aria-controls="timer-palette-panel"
                aria-expanded={isTimerPaletteVisible}
                on:click={() => (isTimerPaletteVisible = !isTimerPaletteVisible)}
              >
                <span
                  class="palette-toggle-swatch"
                  aria-hidden="true"
                  style={`--swatch-color: ${selectedPalette.color};`}
                ></span>
                <span>色</span>
              </button>
            </div>
            {#if isTimerPaletteVisible}
              <div id="timer-palette-panel" class="palette-popover" aria-label="タイマーの色">
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
            {/if}
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
      {:else if activeView === "uuid"}
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
      {:else if activeView === "json"}
        <section class="tool-panel converter-panel" aria-labelledby="json-title">
          <div class="panel-header">
            <div>
              <p class="section-label">JSON</p>
              <h1 id="json-title">JSON 整形・圧縮</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearJson}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                disabled={jsonResult === ""}
                on:click={() => copyText(jsonResult, "json:result")}
              >
                {copiedTextKey === "json:result" ? "コピー済み" : "コピー"}
              </button>
            </div>
          </div>

          <div class="converter-layout">
            <label class="text-area-field">
              <span>入力</span>
              <textarea
                bind:value={jsonInput}
                spellcheck="false"
                placeholder="JSON を貼り付けてください"
              ></textarea>
            </label>

            <label class="text-area-field">
              <span>結果</span>
              <textarea readonly value={jsonResult} spellcheck="false"></textarea>
            </label>
          </div>

          {#if jsonError}
            <p class="tool-message error-message" role="alert">{jsonError}</p>
          {/if}

          <div class="tool-actions">
            <button type="button" class="primary-button" on:click={formatJson}>整形</button>
            <button type="button" class="secondary-button" on:click={minifyJson}>圧縮</button>
          </div>
        </section>
      {:else if activeView === "datetime"}
        <section class="tool-panel converter-panel" aria-labelledby="datetime-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Date / Timestamp</p>
              <h1 id="datetime-title">日時・タイムスタンプ変換</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearDateTime}>クリア</button>
              <button type="button" class="primary-button" on:click={useCurrentDateTime}>現在時刻</button>
            </div>
          </div>

          <label class="single-input-field">
            <span>入力</span>
            <input
              bind:value={datetimeInput}
              placeholder="1700000000 / 1700000000000 / 2023-11-14T22:13:20Z"
              on:keydown={(event) => event.key === "Enter" && convertDateTime()}
            />
          </label>

          {#if datetimeError}
            <p class="tool-message error-message" role="alert">{datetimeError}</p>
          {/if}

          <div class="tool-actions">
            <button type="button" class="primary-button" on:click={convertDateTime}>変換</button>
          </div>

          {#if datetimeResult}
            <dl class="result-list">
              <div>
                <dt>Unix 秒</dt>
                <dd>
                  <code>{datetimeResult.unixSeconds}</code>
                  <button
                    type="button"
                    class="copy-button"
                    on:click={() => copyText(datetimeResult.unixSeconds, "datetime:seconds")}
                  >
                    {copiedTextKey === "datetime:seconds" ? "コピー済み" : "コピー"}
                  </button>
                </dd>
              </div>
              <div>
                <dt>Unix ミリ秒</dt>
                <dd>
                  <code>{datetimeResult.unixMilliseconds}</code>
                  <button
                    type="button"
                    class="copy-button"
                    on:click={() => copyText(datetimeResult.unixMilliseconds, "datetime:milliseconds")}
                  >
                    {copiedTextKey === "datetime:milliseconds" ? "コピー済み" : "コピー"}
                  </button>
                </dd>
              </div>
              <div>
                <dt>ISO</dt>
                <dd>
                  <code>{datetimeResult.iso}</code>
                  <button
                    type="button"
                    class="copy-button"
                    on:click={() => copyText(datetimeResult.iso, "datetime:iso")}
                  >
                    {copiedTextKey === "datetime:iso" ? "コピー済み" : "コピー"}
                  </button>
                </dd>
              </div>
              <div>
                <dt>UTC</dt>
                <dd><code>{datetimeResult.utc}</code></dd>
              </div>
              <div>
                <dt>JST</dt>
                <dd><code>{datetimeResult.jst}</code></dd>
              </div>
            </dl>
          {/if}
        </section>
      {:else if activeView === "base64"}
        <section class="tool-panel converter-panel" aria-labelledby="base64-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Base64</p>
              <h1 id="base64-title">Base64 encode/decode</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearBase64}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                disabled={base64Result === ""}
                on:click={() => copyText(base64Result, "base64:result")}
              >
                {copiedTextKey === "base64:result" ? "コピー済み" : "コピー"}
              </button>
            </div>
          </div>

          <div class="mode-switch" aria-label="Base64 の入力種別">
            <button
              type="button"
              class:active={base64Mode === "text"}
              aria-pressed={base64Mode === "text"}
              on:click={() => switchBase64Mode("text")}
            >
              テキスト
            </button>
            <button
              type="button"
              class:active={base64Mode === "image"}
              aria-pressed={base64Mode === "image"}
              on:click={() => switchBase64Mode("image")}
            >
              画像
            </button>
          </div>

          {#if base64Error}
            <p class="tool-message error-message" role="alert">{base64Error}</p>
          {/if}

          {#if base64Mode === "text"}
            <div class="converter-layout">
              <label class="text-area-field">
                <span>入力</span>
                <textarea bind:value={base64Input} spellcheck="false" placeholder="hello"></textarea>
              </label>

              <label class="text-area-field">
                <span>結果</span>
                <textarea readonly value={base64Result} spellcheck="false"></textarea>
              </label>
            </div>

            <div class="tool-actions">
              <button type="button" class="primary-button" on:click={encodeBase64}>エンコード</button>
              <button type="button" class="secondary-button" on:click={decodeBase64}>デコード</button>
            </div>
          {:else}
            <div class="image-base64-layout">
              <label
                class="image-dropzone"
                on:drop={handleBase64ImageDrop}
                on:dragover={(event) => event.preventDefault()}
              >
                <input type="file" accept="image/*" on:change={handleBase64ImageInput} />
                <span class="image-dropzone-title">画像を選択</span>
                <span>PNG、JPEG、WebP などをドラッグ&ドロップできます。</span>
                <span class="image-dropzone-limit">上限 {formatFileSize(maxBase64ImageBytes)}</span>
              </label>

              <section class="image-preview-panel" aria-label="選択中の画像">
                {#if base64ImagePreviewUrl}
                  <img src={base64ImagePreviewUrl} alt={base64ImageName} />
                  <dl class="image-meta-list">
                    <div>
                      <dt>ファイル名</dt>
                      <dd>{base64ImageName}</dd>
                    </div>
                    <div>
                      <dt>形式</dt>
                      <dd>{base64ImageMimeType}</dd>
                    </div>
                    <div>
                      <dt>元サイズ</dt>
                      <dd>{formatFileSize(base64ImageSize)}</dd>
                    </div>
                    <div>
                      <dt>変換後</dt>
                      <dd>{formatFileSize(base64ResultSize)}</dd>
                    </div>
                  </dl>
                {:else}
                  <div class="image-preview-empty">画像が選択されていません</div>
                {/if}
              </section>
            </div>

            <label class="checkbox-field">
              <input type="checkbox" bind:checked={shouldIncludeBase64DataUrl} />
              <span>data:image/...;base64, を含める</span>
            </label>

            <label class="text-area-field">
              <span>結果</span>
              <textarea readonly value={base64Result} spellcheck="false"></textarea>
            </label>
          {/if}
        </section>
      {:else if activeView === "url"}
        <section class="tool-panel converter-panel" aria-labelledby="url-title">
          <div class="panel-header">
            <div>
              <p class="section-label">URL</p>
              <h1 id="url-title">URL encode/decode</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearUrl}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                disabled={urlResult === ""}
                on:click={() => copyText(urlResult, "url:result")}
              >
                {copiedTextKey === "url:result" ? "コピー済み" : "コピー"}
              </button>
            </div>
          </div>

          <div class="converter-layout">
            <label class="text-area-field">
              <span>入力</span>
              <textarea bind:value={urlInput} spellcheck="false" placeholder="callback=https://example.com/完了"></textarea>
            </label>

            <label class="text-area-field">
              <span>結果</span>
              <textarea readonly value={urlResult} spellcheck="false"></textarea>
            </label>
          </div>

          <label class="checkbox-field">
            <input type="checkbox" bind:checked={decodePlusAsSpace} />
            <span>デコード時に + をスペースとして扱う</span>
          </label>

          {#if urlError}
            <p class="tool-message error-message" role="alert">{urlError}</p>
          {/if}

          <div class="tool-actions">
            <button type="button" class="primary-button" on:click={encodeUrl}>エンコード</button>
            <button type="button" class="secondary-button" on:click={decodeUrl}>デコード</button>
          </div>
        </section>
      {:else if activeView === "jwt"}
        <section class="tool-panel jwt-panel" aria-labelledby="jwt-title">
          <div class="panel-header">
            <div>
              <p class="section-label">JWT</p>
              <h1 id="jwt-title">JWT デコーダー</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearJwt}>クリア</button>
              <button type="button" class="primary-button" on:click={decodeJwt}>デコード</button>
            </div>
          </div>

          <p class="notice-message">署名の検証は行わず、ヘッダーとペイロードの内容確認のみを行います。</p>

          <label class="text-area-field">
            <span>JWT</span>
            <textarea bind:value={jwtInput} spellcheck="false" placeholder="header.payload.signature"></textarea>
          </label>

          {#if jwtError}
            <p class="tool-message error-message" role="alert">{jwtError}</p>
          {/if}

          <div class="jwt-result-grid">
            <section class="jwt-result-section">
              <div class="result-section-header">
                <h2>ヘッダー</h2>
                <button
                  type="button"
                  class="copy-button"
                  disabled={jwtHeaderResult === ""}
                  on:click={() => copyText(jwtHeaderResult, "jwt:header")}
                >
                  {copiedTextKey === "jwt:header" ? "コピー済み" : "コピー"}
                </button>
              </div>
              <pre>{jwtHeaderResult}</pre>
            </section>

            <section class="jwt-result-section">
              <div class="result-section-header">
                <h2>ペイロード</h2>
                <button
                  type="button"
                  class="copy-button"
                  disabled={jwtPayloadResult === ""}
                  on:click={() => copyText(jwtPayloadResult, "jwt:payload")}
                >
                  {copiedTextKey === "jwt:payload" ? "コピー済み" : "コピー"}
                </button>
              </div>
              <pre>{jwtPayloadResult}</pre>
            </section>
          </div>

          {#if jwtClaimTimes.length > 0}
            <dl class="result-list compact">
              {#each jwtClaimTimes as claim}
                <div>
                  <dt>{claim.name}</dt>
                  <dd>
                    <code>{claim.unixSeconds}</code>
                    <span>{claim.utc}</span>
                    <span>{claim.jst}</span>
                  </dd>
                </div>
              {/each}
            </dl>
          {/if}

          <section class="jwt-result-section signature-section">
            <div class="result-section-header">
              <h2>署名</h2>
              <button
                type="button"
                class="copy-button"
                disabled={jwtSignatureResult === ""}
                on:click={() => copyText(jwtSignatureResult, "jwt:signature")}
              >
                {copiedTextKey === "jwt:signature" ? "コピー済み" : "コピー"}
              </button>
            </div>
            <pre>{jwtSignatureResult}</pre>
          </section>
        </section>
      {:else if activeView === "regex"}
        <section class="tool-panel regex-panel" aria-labelledby="regex-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Regex</p>
              <h1 id="regex-title">正規表現テスター</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearRegexTester}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                disabled={regexMatches.length === 0}
                on:click={() => copyText(formatRegexMatchesForCopy(), "regex:matches")}
              >
                {copiedTextKey === "regex:matches" ? "コピー済み" : "コピー"}
              </button>
              <button type="button" class="primary-button" on:click={runRegexTester}>テスト</button>
            </div>
          </div>

          <div class="regex-input-grid">
            <label class="single-input-field">
              <span>正規表現</span>
              <input
                bind:value={regexPattern}
                spellcheck="false"
                placeholder="例: ([\\w.-]+)@([\\w.-]+)"
                on:keydown={(event) => event.key === "Enter" && runRegexTester()}
              />
            </label>

            <fieldset class="flag-fieldset">
              <legend>フラグ</legend>
              <div>
                {#each regexFlagOptions as option}
                  <label>
                    <input type="checkbox" bind:group={regexSelectedFlags} value={option.flag} />
                    <span>{option.flag}</span>
                  </label>
                {/each}
              </div>
            </fieldset>
          </div>

          <label class="text-area-field compact-area">
            <span>テスト文字列</span>
            <textarea bind:value={regexTestText} spellcheck="false" placeholder="ログや検証したい文字列を貼り付けてください"></textarea>
          </label>

          {#if regexError}
            <p class="tool-message error-message" role="alert">{regexError}</p>
          {/if}

          <section class="match-result-list" aria-live="polite">
            <div class="result-section-header">
              <h2>マッチ結果</h2>
              <span>{regexMatches.length}件</span>
            </div>
            {#if regexPattern === "" || regexTestText === ""}
              <p class="empty-result">正規表現とテスト文字列を入力してください。</p>
            {:else if regexMatches.length === 0 && regexError === ""}
              <p class="empty-result">マッチはありません。</p>
            {:else}
              {#each regexMatches as match, index}
                <article class="match-result-item">
                  <div>
                    <strong>#{index + 1}</strong>
                    <span>{match.index}〜{match.end}</span>
                  </div>
                  <code>{match.text}</code>
                  {#if match.groups.length > 0}
                    <dl>
                      {#each match.groups as group, groupIndex}
                        <div>
                          <dt>Group {groupIndex + 1}</dt>
                          <dd>{group === "" ? "（空）" : group}</dd>
                        </div>
                      {/each}
                    </dl>
                  {/if}
                </article>
              {/each}
            {/if}
          </section>
        </section>
      {:else if activeView === "text-counter"}
        <section class="tool-panel counter-panel" aria-labelledby="counter-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Text Counter</p>
              <h1 id="counter-title">文字数・バイト数カウンター</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearTextCounter}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                on:click={() => copyText(textCounterInput, "counter:input")}
              >
                {copiedTextKey === "counter:input" ? "コピー済み" : "入力をコピー"}
              </button>
              <button
                type="button"
                class="primary-button"
                on:click={() => copyText(formatTextCounterStatsForCopy(), "counter:stats")}
              >
                {copiedTextKey === "counter:stats" ? "コピー済み" : "集計をコピー"}
              </button>
            </div>
          </div>

          <label class="text-area-field counter-input">
            <span>入力</span>
            <textarea bind:value={textCounterInput} spellcheck="false" placeholder="文字数やバイト数を確認したいテキスト"></textarea>
          </label>

          <dl class="metric-grid">
            <div>
              <dt>表示上の文字数</dt>
              <dd>{textCounterStats.graphemeCount}</dd>
            </div>
            <div>
              <dt>UTF-8 バイト数</dt>
              <dd>{textCounterStats.utf8Bytes}</dd>
            </div>
            <div>
              <dt>行数</dt>
              <dd>{textCounterStats.lineCount}</dd>
            </div>
            <div>
              <dt>単語数</dt>
              <dd>{textCounterStats.wordCount}</dd>
            </div>
            <div>
              <dt>JavaScript length</dt>
              <dd>{textCounterStats.jsLength}</dd>
            </div>
            <div>
              <dt>トリム後</dt>
              <dd>{textCounterStats.trimmedGraphemeCount}文字 / {textCounterStats.trimmedUtf8Bytes}B</dd>
            </div>
          </dl>
        </section>
      {:else if activeView === "hash"}
        <section class="tool-panel hash-panel" aria-labelledby="hash-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Hash</p>
              <h1 id="hash-title">ハッシュ生成</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearHash}>クリア</button>
              <button
                type="button"
                class="primary-button"
                disabled={hashResult === ""}
                on:click={() => copyText(hashResult, "hash:result")}
              >
                {copiedTextKey === "hash:result" ? "コピー済み" : "コピー"}
              </button>
            </div>
          </div>

          <label class="text-area-field compact-area">
            <span>入力</span>
            <textarea bind:value={hashInput} spellcheck="false" placeholder="hello"></textarea>
          </label>

          <label class="single-input-field select-field">
            <span>アルゴリズム</span>
            <select bind:value={selectedHashAlgorithm}>
              {#each hashAlgorithms as algorithm}
                <option value={algorithm}>{algorithm}</option>
              {/each}
            </select>
          </label>

          {#if hashError}
            <p class="tool-message error-message" role="alert">{hashError}</p>
          {/if}

          <section class="hash-result-section">
            <div class="result-section-header">
              <h2>結果</h2>
              <span>空文字列もそのまま計算します</span>
            </div>
            <code>{hashResult}</code>
          </section>
        </section>
      {:else if activeView === "diff"}
        <section class="tool-panel diff-panel" aria-labelledby="diff-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Diff</p>
              <h1 id="diff-title">Diff ビューア</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearDiff}>クリア</button>
              <button
                type="button"
                class="primary-button"
                on:click={() => copyText(formatDiffForCopy(), "diff:result")}
              >
                {copiedTextKey === "diff:result" ? "コピー済み" : "コピー"}
              </button>
            </div>
          </div>

          <div class="diff-input-grid">
            <label class="text-area-field diff-input-pane">
              <span>比較元</span>
              <textarea bind:value={diffOriginal} spellcheck="false" placeholder="before"></textarea>
              <div class="diff-inline-preview" aria-label="比較元の行別差分">
                {#if diffOriginal === "" && diffChanged === ""}
                  <p>差分プレビュー</p>
                {:else}
                  {#each diffRows.filter((row) => row.oldLine !== undefined) as row}
                    <div class={`diff-preview-row ${row.kind}`}>
                      <span>{row.oldLine}</span>
                      <code>{row.text === "" ? " " : row.text}</code>
                    </div>
                  {/each}
                {/if}
              </div>
            </label>
            <label class="text-area-field diff-input-pane">
              <span>比較先</span>
              <textarea bind:value={diffChanged} spellcheck="false" placeholder="after"></textarea>
              <div class="diff-inline-preview target-preview" aria-label="比較先の行別差分">
                {#if diffOriginal === "" && diffChanged === ""}
                  <p>差分プレビュー</p>
                {:else}
                  {#each diffRows.filter((row) => row.newLine !== undefined) as row}
                    <div class={`diff-preview-row ${row.kind}`}>
                      <span>{row.newLine}</span>
                      <code>{row.text === "" ? " " : row.text}</code>
                    </div>
                  {/each}
                {/if}
              </div>
            </label>
          </div>

          <div class="diff-summary">
            <span>追加 {diffSummary.added}</span>
            <span>削除 {diffSummary.removed}</span>
            <span>変更 {diffSummary.changed}</span>
          </div>

          <section class="diff-result" aria-live="polite">
            {#if diffOriginal === "" && diffChanged === ""}
              <p class="empty-result">比較するテキストを入力してください。</p>
            {:else if diffRows.every((row) => row.kind === "same")}
              <p class="empty-result">差分はありません。</p>
            {:else}
              {#each diffRows as row}
                <div class={`diff-row ${row.kind}`}>
                  <span>{row.oldLine ?? ""}</span>
                  <span>{row.newLine ?? ""}</span>
                  <code>{row.text === "" ? " " : row.text}</code>
                </div>
              {/each}
            {/if}
          </section>
        </section>
      {:else if activeView === "qr"}
        <section class="tool-panel qr-panel" aria-labelledby="qr-title">
          <div class="panel-header">
            <div>
              <p class="section-label">QR Code</p>
              <h1 id="qr-title">QRコード生成</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearQr}>クリア</button>
              <button type="button" class="secondary-button" disabled={qrDataUrl === ""} on:click={downloadQrCode}>PNG保存</button>
              <button type="button" class="primary-button" disabled={qrDataUrl === ""} on:click={copyQrImage}>
                {copiedTextKey === "qr:image" ? "コピー済み" : "画像をコピー"}
              </button>
            </div>
          </div>

          <div class="qr-layout">
            <label class="text-area-field compact-area">
              <span>入力</span>
              <textarea bind:value={qrInput} spellcheck="false" placeholder="https://example.com"></textarea>
            </label>

            <section class="qr-preview" aria-label="QRコードプレビュー">
              {#if qrDataUrl}
                <img src={qrDataUrl} alt="生成したQRコード" />
              {:else}
                <div>入力するとQRコードを表示します</div>
              {/if}
            </section>
          </div>

          {#if qrError}
            <p class="tool-message error-message" role="alert">{qrError}</p>
          {/if}
          {#if qrStatus}
            <p class="notice-message" role="status">{qrStatus}</p>
          {/if}
        </section>
      {:else if activeView === "color"}
        <section class="tool-panel color-panel" aria-labelledby="color-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Color</p>
              <h1 id="color-title">カラーピッカー・色変換</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearColor}>クリア</button>
            </div>
          </div>

          <div class="color-layout">
            <div class="color-preview-card">
              <input
                type="color"
                value={colorPreview}
                aria-label="色を選択"
                on:input={(event) => setColorFromPicker((event.currentTarget as HTMLInputElement).value)}
              />
              <div style={`--preview-color: ${colorPreview};`}></div>
            </div>

            <div class="color-form-grid">
              <form class="color-format-section" on:submit|preventDefault={applyHexColor}>
                <label class="single-input-field">
                  <span>HEX</span>
                  <input bind:value={colorHexInput} spellcheck="false" placeholder="#2f7668" />
                </label>
                <button type="submit" class="secondary-button">反映</button>
                <button type="button" class="copy-button" on:click={() => copyText(colorHexInput, "color:hex")}>
                  {copiedTextKey === "color:hex" ? "コピー済み" : "コピー"}
                </button>
              </form>

              <form class="color-format-section" on:submit|preventDefault={applyRgbColor}>
                <div class="number-field-row">
                  <label class="single-input-field">
                    <span>R</span>
                    <input type="number" min="0" max="255" step="1" bind:value={colorRgb.red} />
                  </label>
                  <label class="single-input-field">
                    <span>G</span>
                    <input type="number" min="0" max="255" step="1" bind:value={colorRgb.green} />
                  </label>
                  <label class="single-input-field">
                    <span>B</span>
                    <input type="number" min="0" max="255" step="1" bind:value={colorRgb.blue} />
                  </label>
                </div>
                <button type="submit" class="secondary-button">反映</button>
                <button
                  type="button"
                  class="copy-button"
                  on:click={() => copyText(`rgb(${colorRgb.red}, ${colorRgb.green}, ${colorRgb.blue})`, "color:rgb")}
                >
                  {copiedTextKey === "color:rgb" ? "コピー済み" : "コピー"}
                </button>
              </form>

              <form class="color-format-section" on:submit|preventDefault={applyHslColor}>
                <div class="number-field-row">
                  <label class="single-input-field">
                    <span>H</span>
                    <input type="number" min="0" max="360" step="1" bind:value={colorHsl.hue} />
                  </label>
                  <label class="single-input-field">
                    <span>S</span>
                    <input type="number" min="0" max="100" step="1" bind:value={colorHsl.saturation} />
                  </label>
                  <label class="single-input-field">
                    <span>L</span>
                    <input type="number" min="0" max="100" step="1" bind:value={colorHsl.lightness} />
                  </label>
                </div>
                <button type="submit" class="secondary-button">反映</button>
                <button
                  type="button"
                  class="copy-button"
                  on:click={() => copyText(`hsl(${colorHsl.hue} ${colorHsl.saturation}% ${colorHsl.lightness}%)`, "color:hsl")}
                >
                  {copiedTextKey === "color:hsl" ? "コピー済み" : "コピー"}
                </button>
              </form>
            </div>
          </div>

          {#if colorError}
            <p class="tool-message error-message" role="alert">{colorError}</p>
          {/if}
        </section>
      {:else if activeView === "memo"}
        <section class="tool-panel memo-panel" aria-labelledby="memo-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Daily Memo</p>
              <h1 id="memo-title">簡易メモ・今日の作業ログ</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={clearDailyMemo}>クリア</button>
              <button
                type="button"
                class="secondary-button"
                disabled={dailyMemo === ""}
                on:click={() => copyText(dailyMemo, "memo:today")}
              >
                {copiedTextKey === "memo:today" ? "コピー済み" : "コピー"}
              </button>
              <button type="button" class="primary-button" on:click={saveDailyMemo}>保存</button>
            </div>
          </div>

          <div class="memo-status-row">
            <time datetime={memoDateKey}>{formatMemoDateLabel(memoDateKey)}</time>
            <span class:dirty={isMemoDirty}>{memoStatus}</span>
          </div>

          <label class="text-area-field memo-input">
            <span>今日のメモ</span>
            <textarea
              bind:value={dailyMemo}
              spellcheck="false"
              placeholder="今日の作業、気づき、次にすること"
              on:input={handleMemoInput}
            ></textarea>
          </label>
        </section>
      {:else if activeView === "ai-chat"}
        <section class="tool-panel ai-chat-panel" aria-labelledby="ai-chat-title">
          <div class="panel-header">
            <div>
              <p class="section-label">Bedrock</p>
              <h1 id="ai-chat-title">AIチャット</h1>
            </div>
            <div class="panel-actions">
              <button type="button" class="secondary-button" on:click={createAiConversation}>
                新規
              </button>
              <button
                type="button"
                class="secondary-button"
                disabled={activeAiConversation === undefined}
                on:click={clearActiveAiConversation}
              >
                クリア
              </button>
            </div>
          </div>

          <div class="ai-chat-layout">
            <aside class="ai-chat-sidebar" aria-label="会話履歴">
              <div class="ai-chat-sidebar-header">
                <h2>履歴</h2>
                <button
                  type="button"
                  class="copy-button"
                  disabled={aiConversations.length <= 1}
                  on:click={deleteActiveAiConversation}
                >
                  削除
                </button>
              </div>

              <div class="ai-conversation-list">
                {#each aiConversations as conversation}
                  <button
                    type="button"
                    class:active={conversation.id === activeAiConversationId}
                    on:click={() => selectAiConversation(conversation.id)}
                  >
                    <span>{conversation.title}</span>
                    <time datetime={conversation.updatedAt}>
                      {formatChatTime(conversation.updatedAt)}
                    </time>
                  </button>
                {/each}
              </div>

              <form class="ai-settings-form" on:submit|preventDefault={saveAiChatSettings}>
                <h2>設定</h2>
                <label class="single-input-field">
                  <span>Bedrock APIキー</span>
                  <input
                    type="password"
                    bind:value={aiChatSettings.apiKey}
                    placeholder="Bedrock API key"
                    autocomplete="off"
                  />
                </label>
                <label class="single-input-field">
                  <span>リージョン</span>
                  <input bind:value={aiChatSettings.region} placeholder="us-east-1" />
                </label>
                <label class="single-input-field">
                  <span>モデルID</span>
                  <input
                    bind:value={aiChatSettings.modelId}
                    placeholder="us.anthropic.claude-3-5-haiku-20241022-v1:0"
                  />
                </label>
                <div class="ai-settings-grid">
                  <label class="single-input-field">
                    <span>Temperature</span>
                    <input
                      type="number"
                      min="0"
                      max="1"
                      step="0.1"
                      bind:value={aiChatSettings.temperature}
                    />
                  </label>
                  <label class="single-input-field">
                    <span>最大トークン</span>
                    <input
                      type="number"
                      min="1"
                      max="8192"
                      step="1"
                      bind:value={aiChatSettings.maxTokens}
                    />
                  </label>
                </div>
                <label class="single-input-field">
                  <span>Tavily APIキー</span>
                  <input
                    type="password"
                    bind:value={aiChatSettings.tavilyApiKey}
                    placeholder="Tavily API key"
                    autocomplete="off"
                  />
                </label>
                <label class="checkbox-field">
                  <input
                    type="checkbox"
                    bind:checked={aiChatSettings.useWebSearch}
                    disabled={aiChatSettings.tavilyApiKey.trim() === ""}
                  />
                  <span>必要に応じてWeb検索を使う</span>
                </label>
                <div class="ai-settings-grid">
                  <label class="single-input-field">
                    <span>検索深度</span>
                    <select bind:value={aiChatSettings.tavilySearchDepth}>
                      <option value="basic">basic</option>
                      <option value="advanced">advanced</option>
                      <option value="fast">fast</option>
                      <option value="ultra-fast">ultra-fast</option>
                    </select>
                  </label>
                  <label class="single-input-field">
                    <span>検索件数</span>
                    <input
                      type="number"
                      min="1"
                      max="10"
                      step="1"
                      bind:value={aiChatSettings.tavilyMaxResults}
                    />
                  </label>
                </div>
                <button type="submit" class="primary-button">保存</button>
                {#if aiSettingsMessage}
                  <p class="input-message">{aiSettingsMessage}</p>
                {/if}
              </form>
            </aside>

            <div class="ai-chat-main">
              <div class="ai-message-list" aria-live="polite">
                {#if activeAiConversation === undefined || activeAiConversation.messages.length === 0}
                  <div class="ai-empty-state">
                    <strong>Bedrock とテキストで会話できます</strong>
                    <span>Bedrock設定を保存すると使えます。Tavily APIキーを追加すると、必要なときだけWeb検索も使えます。</span>
                  </div>
                {:else}
                  {#each activeAiConversation.messages as message}
                    <article class={message.role === "user" ? "ai-message user-message" : "ai-message"}>
                      <div class="ai-message-meta">
                        <span>{message.role === "user" ? "You" : "Bedrock"}</span>
                        <time datetime={message.createdAt}>{formatChatTime(message.createdAt)}</time>
                      </div>
                      <p>
                        {#if message.text === "" && message.role === "assistant" && isAiChatSending}
                          応答を準備しています...
                        {:else}
                          {message.text}
                        {/if}
                      </p>
                    </article>
                  {/each}
                {/if}

                {#if isAiChatSending && activeAiConversation?.messages.at(-1)?.role !== "assistant"}
                  <article class="ai-message">
                    <div class="ai-message-meta">
                      <span>Bedrock</span>
                    </div>
                    <p>応答を生成しています...</p>
                  </article>
                {/if}
              </div>

              {#if aiSearchStatus !== "idle" && aiSearchStatusText && aiSearchConversationId === activeAiConversationId}
                <p class={`ai-search-status ${aiSearchStatus}`}>
                  {#if aiSearchStatus === "checking" || aiSearchStatus === "searching"}
                    <span aria-hidden="true"></span>
                  {/if}
                  {aiSearchStatusText}
                </p>
              {/if}

              {#if aiChatError}
                <p class="tool-message error-message" role="alert">{aiChatError}</p>
              {/if}
              {#if lastAiUsageSummary}
                <p class="notice-message">{lastAiUsageSummary}</p>
              {/if}

              <form class="ai-chat-composer" on:submit|preventDefault={sendAiChatMessage}>
                <textarea
                  bind:value={aiChatInput}
                  placeholder="メッセージを入力"
                  rows="3"
                  disabled={isAiChatSending}
                ></textarea>
                <button
                  type="submit"
                  class="primary-button"
                  disabled={isAiChatSending || aiChatInput.trim() === ""}
                >
                  送信
                </button>
              </form>
            </div>
          </div>
        </section>
      {/if}
    </div>
  {/if}
</main>
