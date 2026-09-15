<script lang="ts">
import { onDestroy, onMount } from "svelte";
import Icon from "@/components/common/Icon.svelte";
import { siteConfig, ttsConfig } from "@/config";
import I18nKey from "@/i18n/i18nKey";
import { i18n } from "@/i18n/translation";
import { extractReadableText, splitForSpeech } from "@/utils/tts-text";

interface Props {
	title: string;
	cover?: string | null;
}

let { title, cover = null }: Props = $props();

let open = $state(false);
let playing = $state(false);
let loading = $state(false);
let mode = $state<"server" | "speech">("server");
let rate = $state(1);
let voice = $state(ttsConfig.defaultVoice);
let elapsed = $state(0);
let duration = $state(0);
let notice = $state("");

let audio: HTMLAudioElement | null = null;
let text = "";
let speechQueue: string[] = [];
let speechIndex = 0;
let speechEpoch = 0;
let noticeTimer: ReturnType<typeof setTimeout> | null = null;

const hasServer = ttsConfig.enable && ttsConfig.serverUrl.length > 0;

onMount(() => {
	const savedRate = Number(localStorage.getItem("firefly-tts-rate") || "1");
	if (ttsConfig.speeds.includes(savedRate)) {
		rate = savedRate;
	}
	const savedVoice = localStorage.getItem("firefly-tts-voice");
	if (savedVoice && ttsConfig.voices.some((item) => item.name === savedVoice)) {
		voice = savedVoice;
	}
});

onDestroy(() => {
	stopAll();
	if (noticeTimer) clearTimeout(noticeTimer);
});

function showNotice(message: string): void {
	if (!message) return;
	notice = message;
	if (noticeTimer) clearTimeout(noticeTimer);
	noticeTimer = setTimeout(() => {
		notice = "";
	}, 6000);
}

function formatTime(value: number): string {
	if (!Number.isFinite(value) || value <= 0) return "00:00";
	const minutes = Math.floor(value / 60);
	const seconds = Math.floor(value % 60);
	return `${String(minutes).padStart(2, "0")}:${String(seconds).padStart(2, "0")}`;
}

function collectText(): string {
	const root = document.querySelector("#post-container .markdown-content");
	const value = extractReadableText(root);
	if (!value) showNotice(i18n(I18nKey.ttsEmpty));
	return value;
}

function cancelSpeech(): void {
	speechEpoch += 1;
	window.speechSynthesis.cancel();
}

function releaseAudio(): void {
	if (!audio) return;
	audio.onerror = null;
	audio.ontimeupdate = null;
	audio.onended = null;
	audio.pause();
	audio.removeAttribute("src");
	audio.load();
	audio = null;
}

function stopAll(): void {
	releaseAudio();
	cancelSpeech();
	playing = false;
	loading = false;
	elapsed = 0;
	duration = 0;
}

function setupMediaSession(): void {
	if (!("mediaSession" in navigator)) return;
	navigator.mediaSession.metadata = new MediaMetadata({
		title,
		artist: siteConfig.title,
		album: i18n(I18nKey.ttsRead),
		artwork: cover ? [{ src: cover }] : [],
	});
	navigator.mediaSession.setActionHandler("play", () => {
		if (mode === "server" && audio) {
			void audio.play();
			playing = true;
		}
	});
	navigator.mediaSession.setActionHandler("pause", () => {
		audio?.pause();
		playing = false;
	});
}

function speakNext(epoch: number): void {
	if (epoch !== speechEpoch || speechIndex >= speechQueue.length) {
		if (epoch === speechEpoch) playing = false;
		return;
	}
	const utterance = new SpeechSynthesisUtterance(speechQueue[speechIndex]);
	utterance.lang = document.documentElement.lang || "zh-CN";
	utterance.rate = rate;
	utterance.onend = () => {
		speechIndex += 1;
		speakNext(epoch);
	};
	utterance.onerror = () => {
		speechIndex += 1;
		speakNext(epoch);
	};
	playing = true;
	window.speechSynthesis.speak(utterance);
}

function startSpeech(message: string): void {
	if (message) showNotice(message);
	mode = "speech";
	cancelSpeech();
	speechQueue = splitForSpeech(text);
	speechIndex = 0;
	if (speechQueue.length === 0) {
		playing = false;
		return;
	}
	speakNext(speechEpoch);
}

async function startServer(): Promise<void> {
	releaseAudio();
	cancelSpeech();
	mode = "server";
	loading = true;
	notice = "";
	const controller = new AbortController();
	const timeout = setTimeout(
		() => controller.abort(),
		ttsConfig.requestTimeoutMs,
	);
	try {
		const response = await fetch(`${ttsConfig.serverUrl}/tts`, {
			method: "POST",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({ text, voice }),
			signal: controller.signal,
		});
		if (!response.ok) {
			throw new Error(`HTTP ${response.status}`);
		}
		const data = (await response.json()) as { id: string };
		const player = new Audio(`${ttsConfig.serverUrl}/audio/${data.id}`);
		player.playbackRate = rate;
		player.ontimeupdate = () => {
			elapsed = player.currentTime;
			duration = Number.isFinite(player.duration) ? player.duration : 0;
		};
		player.onended = () => {
			playing = false;
			elapsed = 0;
		};
		player.onerror = () => {
			startSpeech(i18n(I18nKey.ttsFallback));
		};
		audio = player;
		setupMediaSession();
		try {
			await player.play();
			playing = true;
		} catch {
			playing = false;
		}
	} catch {
		startSpeech(i18n(I18nKey.ttsFallback));
	} finally {
		clearTimeout(timeout);
		loading = false;
	}
}

async function start(): Promise<void> {
	stopAll();
	open = true;
	text = collectText();
	if (!text) {
		open = false;
		return;
	}
	if (hasServer) {
		await startServer();
	} else {
		startSpeech(i18n(I18nKey.ttsNoServer));
	}
}

function toggle(): void {
	if (!open) {
		void start();
		return;
	}
	if (loading) return;
	if (mode === "server" && audio) {
		if (playing) {
			audio.pause();
			playing = false;
		} else {
			void audio.play();
			playing = true;
		}
		return;
	}
	if (mode === "speech") {
		if (playing) {
			window.speechSynthesis.pause();
			playing = false;
		} else {
			window.speechSynthesis.resume();
			playing = true;
		}
	}
}

function changeRate(event: Event): void {
	const value = Number((event.currentTarget as HTMLSelectElement).value);
	rate = value;
	localStorage.setItem("firefly-tts-rate", String(value));
	if (mode === "server" && audio) {
		audio.playbackRate = value;
		return;
	}
	if (mode === "speech") {
		cancelSpeech();
		speakNext(speechEpoch);
	}
}

function changeVoice(event: Event): void {
	voice = (event.currentTarget as HTMLSelectElement).value;
	localStorage.setItem("firefly-tts-voice", voice);
	if (mode === "server" && open) {
		void startServer();
	}
}

function seek(event: Event): void {
	const value = Number((event.currentTarget as HTMLInputElement).value);
	if (mode === "server" && audio) {
		audio.currentTime = value;
		elapsed = value;
	}
}

function close(): void {
	stopAll();
	open = false;
	notice = "";
}
</script>

<span class="tts-player__trigger">
	<button
		type="button"
		class="tts-player__button"
		disabled={loading}
		onclick={toggle}
		aria-label={i18n(I18nKey.ttsRead)}
	>
		<Icon icon="material-symbols:headphones" />
		<span class="tts-player__button-text">{i18n(I18nKey.ttsRead)}</span>
	</button>
</span>

{#if open}
	<div class="tts-player" role="region" aria-label={i18n(I18nKey.ttsRead)}>
		<button
			type="button"
			class="tts-player__control"
			onclick={toggle}
			aria-label={playing ? i18n(I18nKey.ttsPause) : i18n(I18nKey.ttsResume)}
		>
			<Icon icon={playing ? "material-symbols:pause" : "material-symbols:play-arrow"} />
		</button>

		<div class="tts-player__timeline">
			<input
				class="tts-player__seek"
				type="range"
				min="0"
				max={duration > 0 ? duration : 0}
				step="0.5"
				value={elapsed}
				oninput={seek}
				aria-label={i18n(I18nKey.ttsRead)}
				disabled={mode === "speech" || duration === 0}
			/>
			<span class="tts-player__time">
				{formatTime(elapsed)} / {duration > 0 ? formatTime(duration) : "--:--"}
			</span>
		</div>

		<label class="tts-player__field">
			<span>{i18n(I18nKey.ttsSpeed)}</span>
			<select class="tts-player__select" value={rate} onchange={changeRate}>
				{#each ttsConfig.speeds as speed (speed)}
					<option value={speed}>{speed === 1 ? "1x" : `${speed}x`}</option>
				{/each}
			</select>
		</label>

		{#if mode === "server"}
			<label class="tts-player__field">
				<span>{i18n(I18nKey.ttsVoice)}</span>
				<select class="tts-player__select" value={voice} onchange={changeVoice}>
					{#each ttsConfig.voices as item (item.name)}
						<option value={item.name}>{item.label}</option>
					{/each}
				</select>
			</label>
		{/if}

		<button
			type="button"
			class="tts-player__control"
			onclick={close}
			aria-label={i18n(I18nKey.ttsClose)}
		>
			<Icon icon="material-symbols:close" />
		</button>

		{#if loading}
			<span class="tts-player__notice">{i18n(I18nKey.ttsPreparing)}</span>
		{:else if notice}
			<span class="tts-player__notice">{notice}</span>
		{/if}
	</div>
{/if}
