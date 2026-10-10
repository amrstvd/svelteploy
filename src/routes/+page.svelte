<script lang="ts">
	import { onMount } from 'svelte';

	const LOG = [
		'→ pulling main @ 9b41829',
		'→ docker build · node:22-alpine',
		'→ vite build · 41 modules',
		'→ adapter-node · output ready',
		'→ health check /  200 OK',
		'✓ live'
	];

	let canvas: HTMLCanvasElement;
	let lines = $state<string[]>([]);
	let deploys = $state(1);
	let since = $state(Date.now());
	let now = $state(Date.now());

	let ripples: { x: number; y: number; t: number }[] = [];
	let runId = 0;

	const uptime = $derived.by(() => {
		const s = Math.floor((now - since) / 1000);
		const pad = (n: number) => String(n).padStart(2, '0');
		return `${pad(Math.floor(s / 3600))}:${pad(Math.floor(s / 60) % 60)}:${pad(s % 60)}`;
	});

	async function playLog() {
		const id = ++runId;
		lines = [];
		for (const line of LOG) {
			await new Promise((r) => setTimeout(r, 280 + Math.random() * 320));
			if (id !== runId) return;
			lines = [...lines, line];
		}
	}

	function redeploy() {
		deploys += 1;
		since = Date.now();
		playLog();
	}

	onMount(() => {
		const ctx = canvas.getContext('2d')!;
		const reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;
		const GAP = 26;
		let w = 0;
		let h = 0;
		let mx = -9999;
		let my = -9999;
		let raf = 0;

		function resize() {
			const dpr = devicePixelRatio || 1;
			w = innerWidth;
			h = innerHeight;
			canvas.width = w * dpr;
			canvas.height = h * dpr;
			ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
		}

		function frame(t: number) {
			ctx.clearRect(0, 0, w, h);
			ripples = ripples.filter((r) => t - r.t < 2200);

			for (let y = GAP / 2; y < h; y += GAP) {
				for (let x = GAP / 2; x < w; x += GAP) {
					let dx = 0;
					let dy = 0;
					let heat = 0;

					// cursor pushes dots away
					const cx = x - mx;
					const cy = y - my;
					const cd = Math.hypot(cx, cy);
					if (cd < 160) {
						const f = (1 - cd / 160) ** 2;
						dx += (cx / (cd || 1)) * f * 18;
						dy += (cy / (cd || 1)) * f * 18;
						heat = Math.max(heat, f);
					}

					// redeploy shockwaves
					for (const r of ripples) {
						const age = (t - r.t) / 1000;
						const front = age * 900;
						const rd = Math.hypot(x - r.x, y - r.y);
						const band = Math.abs(rd - front);
						if (band < 60) {
							const f = (1 - band / 60) * Math.max(0, 1 - age / 2.2);
							dx += ((x - r.x) / (rd || 1)) * f * 10;
							dy += ((y - r.y) / (rd || 1)) * f * 10;
							heat = Math.max(heat, f);
						}
					}

					const idle = reduce ? 0 : Math.sin(t / 1400 + x * 0.012 + y * 0.009) * 0.5 + 0.5;
					const size = 1 + heat * 2.2 + idle * 0.4;
					ctx.fillStyle =
						heat > 0.05
							? `rgba(255, 62, 0, ${0.35 + heat * 0.65})`
							: `rgba(237, 230, 218, ${0.08 + idle * 0.1})`;
					ctx.fillRect(x + dx - size / 2, y + dy - size / 2, size, size);
				}
			}
			raf = requestAnimationFrame(frame);
		}

		const move = (e: PointerEvent) => {
			mx = e.clientX;
			my = e.clientY;
		};
		const down = (e: PointerEvent) => {
			ripples.push({ x: e.clientX, y: e.clientY, t: performance.now() });
		};
		const leave = () => {
			mx = my = -9999;
		};

		resize();
		addEventListener('resize', resize);
		addEventListener('pointermove', move);
		addEventListener('pointerdown', down);
		document.addEventListener('pointerleave', leave);
		raf = requestAnimationFrame(frame);
		const tick = setInterval(() => (now = Date.now()), 1000);
		playLog();

		return () => {
			cancelAnimationFrame(raf);
			clearInterval(tick);
			removeEventListener('resize', resize);
			removeEventListener('pointermove', move);
			removeEventListener('pointerdown', down);
			document.removeEventListener('pointerleave', leave);
			runId++;
		};
	});
</script>

<svelte:head>
	<title>svelteploy — it shipped</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		rel="stylesheet"
		href="https://fonts.googleapis.com/css2?family=Instrument+Serif:ital@0;1&family=JetBrains+Mono:wght@400;600&display=swap"
	/>
</svelte:head>

<canvas bind:this={canvas} class="pointer-events-none fixed inset-0 h-full w-full"></canvas>

<main
	class="relative mx-auto flex min-h-dvh max-w-6xl flex-col justify-between px-4 py-6 sm:px-8 sm:py-10"
>
	<header class="mono flex items-center justify-between text-xs tracking-wider text-stone-400">
		<span>svelteploy</span>
		<span class="flex items-center gap-2">
			<span class="relative flex size-2">
				<span class="absolute inset-0 animate-ping rounded-full bg-[#ff3e00] opacity-60"></span>
				<span class="relative size-2 rounded-full bg-[#ff3e00]"></span>
			</span>
			up {uptime}
		</span>
	</header>

	<section class="grid items-end gap-10 py-16 md:grid-cols-[1.4fr_1fr]">
		<div>
			<p class="mono mb-6 text-xs tracking-[0.3em] text-[#ff3e00] uppercase">
				deploy #{String(deploys).padStart(3, '0')}
			</p>
			<h1 class="serif text-[clamp(4rem,13vw,10.5rem)] leading-[0.85] text-stone-100">
				It <em class="text-[#ff3e00]">shipped.</em>
			</h1>
			<p class="mt-8 max-w-md text-lg leading-relaxed text-stone-400">
				A SvelteKit app, one Dockerfile, and a box that runs it. Nothing else between you and
				production.
			</p>
			<button
				onclick={redeploy}
				class="mono group mt-10 inline-flex items-center gap-3 rounded-full border border-stone-700 bg-stone-950/60 px-6 py-3 text-sm text-stone-100 backdrop-blur transition hover:border-[#ff3e00] hover:text-white active:scale-95"
			>
				redeploy
				<span class="transition group-hover:translate-x-1">→</span>
			</button>
		</div>

		<pre
			class="mono min-h-56 rounded-lg border border-stone-800 bg-stone-950/70 p-5 text-[13px] leading-7 text-stone-400 backdrop-blur"
			aria-live="polite">{#each lines as line, i (i)}<span
					class:text-[#ff3e00]={line.startsWith('✓')}
					class="block">{line}</span
				>{/each}<span class="caret">▍</span></pre>
	</section>

	<footer class="mono flex justify-between text-xs text-stone-600">
		<span>sveltekit · adapter-node · dokploy</span>
		<span>move · click</span>
	</footer>
</main>

<style>
	:global(body) {
		background: #0e0d0b;
	}
	.serif {
		font-family: 'Instrument Serif', serif;
	}
	.mono {
		font-family: 'JetBrains Mono', monospace;
	}
	.caret {
		color: #ff3e00;
		animation: blink 1s steps(1) infinite;
	}
	@keyframes blink {
		50% {
			opacity: 0;
		}
	}
</style>
