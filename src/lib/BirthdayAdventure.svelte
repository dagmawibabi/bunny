<script lang="ts">
	import { onMount } from 'svelte';
	let { onclose }: { onclose: () => void } = $props();
	let canvas: HTMLCanvasElement;
	let stage: HTMLDivElement;
	let width = $state(960);
	let phase = $state<'intro' | 'playing' | 'paused' | 'won'>('intro');
	let score = $state(0);
	let chapter = $state(0);
	let collected = $state(0);
	let combo = $state(0);
	let message = $state('A birthday surprise is waiting at the end of the meadow.');
	let muted = $state(false);
	let best = $state(0);
	const names = ['Clover meadow', 'Mushroom woods', 'The birthday picnic'];
	const keys = new Set<string>();
	const height = 540,
		ground = 438,
		worldEnd = 7200;
	let camera = 0,
		elapsed = 0,
		lastTime = 0,
		frame = 0,
		lastCollect = -10;
	let jumps = 0,
		invincible = 0,
		checkpoint = 90;
	let shield = $state(0);
	let audio: AudioContext | undefined;
	let bunny = $state({ x: 90, y: ground - 48, vx: 0, vy: 0, facing: 1 });
	type Platform = { x: number; y: number; w: number; spring?: boolean };
	type Treat = { x: number; y: number; kind: 'carrot' | 'star'; taken: boolean };
	type Particle = { x: number; y: number; vx: number; vy: number; life: number; color: string };
	let platforms: Platform[] = [],
		treats: Treat[] = [],
		particles: Particle[] = [];
	const bees = [1050, 1580, 2850, 3340, 4030, 5190, 5930, 6380];

	function buildWorld() {
		platforms = [];
		treats = [];
		particles = [];
		for (let x = 440; x < worldEnd - 450; x += 440) {
			const y = Math.floor(x / 440) % 2 === 0 ? 320 : 365;
			platforms.push({ x, y, w: 180 });
			for (let i = 0; i < 3; i++)
				treats.push({ x: x + 35 + i * 50, y: y - 40, kind: 'carrot', taken: false });
			if (x % 1320 === 0) treats.push({ x: x + 90, y: y - 115, kind: 'star', taken: false });
		}
		for (let x = 220; x < worldEnd - 200; x += 220)
			treats.push({ x, y: ground - 30, kind: 'carrot', taken: false });
		[800, 2250, 3700, 4900, 6100].forEach((x) =>
			platforms.push({ x, y: ground, w: 64, spring: true })
		);
	}
	function tone(freq: number, duration = 0.12) {
		if (muted || !audio) return;
		const osc = audio.createOscillator(),
			gain = audio.createGain(),
			now = audio.currentTime;
		osc.type = 'sine';
		osc.frequency.setValueAtTime(freq, now);
		gain.gain.setValueAtTime(0.065, now);
		gain.gain.exponentialRampToValueAtTime(0.0001, now + duration);
		osc.connect(gain).connect(audio.destination);
		osc.start();
		osc.stop(now + duration);
	}
	function start() {
		buildWorld();
		bunny = { x: 90, y: ground - 48, vx: 0, vy: 0, facing: 1 };
		score = 0;
		collected = 0;
		combo = 0;
		chapter = 0;
		camera = 0;
		elapsed = 0;
		lastCollect = -10;
		jumps = 0;
		shield = 0;
		invincible = 0;
		checkpoint = 90;
		keys.clear();
		phase = 'playing';
		message = 'Follow the carrot trail. You can jump twice!';
		try {
			audio ??= new AudioContext();
			void audio.resume();
		} catch {
			/* Sound is optional. */
		}
	}
	function jump() {
		if (phase !== 'playing' || jumps >= 2) return;
		bunny.vy = jumps === 0 ? -510 : -440;
		jumps += 1;
		tone(jumps === 1 ? 430 : 630);
		emit(bunny.x, bunny.y + 45, '#fff6db', 7);
	}
	function emit(x: number, y: number, color: string, count: number) {
		for (let i = 0; i < count; i++)
			particles.push({
				x,
				y,
				vx: (Math.random() - 0.5) * 200,
				vy: -Math.random() * 200,
				life: 0.7,
				color
			});
	}
	function pause() {
		if (phase === 'playing') {
			phase = 'paused';
			keys.clear();
		} else if (phase === 'paused') phase = 'playing';
	}
	function keydown(e: KeyboardEvent) {
		if (!['ArrowLeft', 'ArrowRight', 'ArrowUp', ' ', 'a', 'd', 'w', 'Escape'].includes(e.key))
			return;
		e.preventDefault();
		if (e.key === 'Escape') {
			if (!e.repeat) pause();
			return;
		}
		if ([' ', 'ArrowUp', 'w'].includes(e.key)) {
			if (!e.repeat) jump();
		} else keys.add(e.key);
	}
	function keyup(e: KeyboardEvent) {
		keys.delete(e.key);
	}
	function hold(e: PointerEvent, direction: string) {
		e.preventDefault();
		(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
		keys.add(direction);
	}
	function release() {
		keys.clear();
	}
	function advance(dt: number) {
		elapsed += dt;
		shield = Math.max(0, shield - dt);
		invincible = Math.max(0, invincible - dt);
		if (elapsed - lastCollect > 2.5) combo = 0;
		const direction =
			Number(keys.has('ArrowRight') || keys.has('d')) -
			Number(keys.has('ArrowLeft') || keys.has('a'));
		bunny.vx += (direction * 280 - bunny.vx) * Math.min(1, dt * 13);
		if (direction) bunny.facing = direction;
		bunny.x = Math.max(30, Math.min(worldEnd - 60, bunny.x + bunny.vx * dt));
		const previousBottom = bunny.y + 48;
		bunny.vy += 1250 * dt;
		bunny.y += bunny.vy * dt;
		let floor = ground;
		for (const p of platforms) {
			if (
				bunny.x + 20 > p.x &&
				bunny.x - 20 < p.x + p.w &&
				previousBottom <= p.y + 3 &&
				bunny.y + 48 >= p.y &&
				bunny.vy >= 0
			) {
				floor = Math.min(floor, p.y);
				if (p.spring) {
					bunny.y = p.y - 49;
					bunny.vy = -720;
					jumps = 1;
					emit(bunny.x, p.y, '#e8a0bd', 14);
					tone(900, 0.2);
					message = 'Boing! Mushroom express!';
				}
			}
		}
		if (bunny.y + 48 >= floor && bunny.vy >= 0) {
			bunny.y = floor - 48;
			bunny.vy = 0;
			jumps = 0;
		}
		camera = Math.max(0, Math.min(worldEnd - width, bunny.x - width * 0.3));
		const nextChapter = Math.min(2, Math.floor(bunny.x / 2400));
		if (nextChapter > chapter) {
			chapter = nextChapter;
			checkpoint = chapter * 2400 + 40;
			message = `${names[chapter]} — checkpoint saved!`;
			tone(880, 0.25);
		}
		for (const t of treats) {
			if (!t.taken && Math.abs(t.x - bunny.x) < 35 && Math.abs(t.y - (bunny.y + 22)) < 39) {
				t.taken = true;
				lastCollect = elapsed;
				combo = Math.min(5, combo + 1);
				collected += 1;
				score += t.kind === 'star' ? 100 : 10 * combo;
				emit(t.x, t.y, t.kind === 'star' ? '#fff082' : '#fcb58b', 10);
				if (t.kind === 'star') {
					shield = 7;
					message = 'Lucky star! Seven seconds of bunny magic.';
				}
				tone(550 + combo * 90);
			}
		}
		for (const x of bees) {
			const y = ground - 35 + Math.sin(elapsed * 3 + x) * 17;
			if (
				Math.abs(x - bunny.x) < 34 &&
				Math.abs(y - (bunny.y + 24)) < 30 &&
				invincible <= 0 &&
				shield <= 0
			) {
				invincible = 2;
				combo = 0;
				score = Math.max(0, score - 30);
				bunny.vy = -320;
				jumps = 1;
				bunny.x = Math.max(checkpoint, bunny.x - 65);
				message = 'Oops, a tickly bee! Hop over the next one.';
				tone(180, 0.15);
				emit(bunny.x, bunny.y, '#e6aed0', 12);
			}
		}
		particles = particles.filter((p) => p.life > 0);
		for (const p of particles) {
			p.x += p.vx * dt;
			p.y += p.vy * dt;
			p.vy += 450 * dt;
			p.life -= dt;
		}
		if (bunny.x >= worldEnd - 170) {
			phase = 'won';
			keys.clear();
			best = Math.max(best, score);
			try {
				localStorage.setItem('rihanna-bunny-best', String(best));
			} catch {
				/* Play works without storage. */
			}
			emit(bunny.x, bunny.y, '#ee95b1', 60);
			tone(1046, 0.5);
		}
	}
	function draw(ctx: CanvasRenderingContext2D) {
		ctx.clearRect(0, 0, width, height);
		const skies = ['#bde7e7', '#d4c7ec', '#f7d8c3'];
		ctx.fillStyle = skies[chapter];
		ctx.fillRect(0, 0, width, height);
		// Slow-moving clouds and hills make the meadow feel like a journey.
		for (let i = 0; i < 7; i++) {
			const x = ((((i * 220 - camera * 0.16) % 1400) + 1400) % 1400) - 180;
			ctx.fillStyle = '#fff8ee';
			ellipse(ctx, x, 90 + (i % 3) * 35, 68, 18);
			ellipse(ctx, x - 20, 80 + (i % 3) * 35, 28, 25);
			ellipse(ctx, x + 23, 75 + (i % 3) * 35, 32, 30);
		}
		for (let i = -1; i < 8; i++) {
			ctx.fillStyle = i % 2 ? '#8fc7a3' : '#a4d3aa';
			ellipse(ctx, i * 230 - ((camera * 0.3) % 230), 445, 220, 145);
		}
		ctx.fillStyle = '#80b977';
		ctx.fillRect(0, ground, width, height - ground);
		ctx.fillStyle = '#659f69';
		ctx.fillRect(0, ground, width, 8);
		ctx.save();
		ctx.translate(-camera, 0);
		for (let x = 30; x < worldEnd; x += 95) {
			ctx.fillStyle = x % 3 ? '#ffe7a1' : '#f5a9be';
			ellipse(ctx, x, ground + 28 + (x % 25), 5, 5);
			ctx.fillStyle = '#487e59';
			ctx.fillRect(x - 1, ground + 32 + (x % 25), 2, 12);
		}
		for (const p of platforms) {
			if (p.x + p.w < camera || p.x > camera + width) continue;
			ctx.fillStyle = p.spring ? '#f8e4bd' : '#9d866e';
			ctx.fillRect(p.x + 8, p.y + 6, p.w - 16, p.spring ? 25 : 20);
			ctx.fillStyle = p.spring ? '#e794ad' : '#6fa878';
			ctx.beginPath();
			ctx.roundRect(p.x, p.y - 5, p.w, 15, 8);
			ctx.fill();
			if (p.spring) {
				ctx.fillStyle = '#fff3d9';
				ellipse(ctx, p.x + 17, p.y, 4, 3);
				ellipse(ctx, p.x + 42, p.y, 5, 3);
			}
		}
		for (const t of treats) {
			if (t.taken || t.x < camera - 40 || t.x > camera + width + 40) continue;
			const bob = Math.sin(elapsed * 3 + t.x) * 4;
			ctx.save();
			ctx.translate(t.x, t.y + bob);
			if (t.kind === 'star') {
				ctx.fillStyle = '#fff181';
				ctx.strokeStyle = '#bf926c';
				ctx.lineWidth = 2;
				star(ctx, 0, 0, 18);
			} else {
				ctx.rotate(0.3);
				ctx.fillStyle = '#f4a35e';
				ctx.strokeStyle = '#b77859';
				ctx.lineWidth = 2;
				ctx.beginPath();
				ctx.moveTo(-9, -9);
				ctx.quadraticCurveTo(14, -15, 9, 1);
				ctx.lineTo(-4, 19);
				ctx.closePath();
				ctx.fill();
				ctx.stroke();
				ctx.strokeStyle = '#529669';
				ctx.lineWidth = 4;
				ctx.beginPath();
				ctx.moveTo(0, -10);
				ctx.lineTo(-3, -22);
				ctx.moveTo(2, -10);
				ctx.lineTo(10, -20);
				ctx.stroke();
			}
			ctx.restore();
		}
		for (const x of bees) {
			if (x < camera - 40 || x > camera + width + 40) continue;
			const y = ground - 35 + Math.sin(elapsed * 3 + x) * 17;
			ctx.fillStyle = '#fff9e4';
			ellipse(ctx, x - 7, y - 15, 8, 12);
			ellipse(ctx, x + 7, y - 15, 8, 12);
			ctx.fillStyle = '#ffd984';
			ellipse(ctx, x, y, 20, 14);
			ctx.fillStyle = '#765264';
			ctx.fillRect(x - 4, y - 12, 6, 24);
			ellipse(ctx, x + 11, y - 2, 2, 3);
		}
		// The gift is the destination, not a timer: explore at your own pace.
		ctx.fillStyle = '#f29eaf';
		ctx.beginPath();
		ctx.roundRect(worldEnd - 150, ground - 85, 80, 85, 8);
		ctx.fill();
		ctx.fillStyle = '#fff0ba';
		ctx.fillRect(worldEnd - 117, ground - 85, 14, 85);
		ellipse(ctx, worldEnd - 124, ground - 91, 16, 10);
		ellipse(ctx, worldEnd - 95, ground - 91, 16, 10);
		ctx.fillStyle = '#61465d';
		ctx.font = '16px Fredoka, sans-serif';
		ctx.textAlign = 'center';
		ctx.fillText('For Rihanna ♡', worldEnd - 110, ground - 112);
		const squash = bunny.vy === 0 && Math.abs(bunny.vx) > 20 ? Math.sin(elapsed * 18) * 2 : 0;
		ctx.save();
		ctx.translate(bunny.x, bunny.y + 25 + squash);
		ctx.scale(bunny.facing, 1);
		if (shield > 0) {
			ctx.strokeStyle = '#fff09b';
			ctx.lineWidth = 4;
			ellipse(ctx, 0, -3, 39, 49, true);
		}
		ctx.globalAlpha = invincible > 0 && Math.floor(elapsed * 12) % 2 ? 0.45 : 1;
		ctx.strokeStyle = '#795466';
		ctx.lineWidth = 2.5;
		ctx.fillStyle = '#fff9ed';
		ellipse(ctx, -11, -29, 8, 25);
		ellipse(ctx, 11, -29, 8, 25);
		ctx.fillStyle = '#f2b6c2';
		ellipse(ctx, -11, -31, 3, 16);
		ellipse(ctx, 11, -31, 3, 16);
		ctx.fillStyle = '#fff9ed';
		ellipse(ctx, -24, 13, 10, 10);
		ellipse(ctx, 0, 4, 25, 24);
		ellipse(ctx, -12, 23, 11, 6);
		ellipse(ctx, 12, 23, 11, 6);
		ctx.fillStyle = '#5c3e54';
		ellipse(ctx, -8, 0, 2, 3);
		ellipse(ctx, 8, 0, 2, 3);
		ctx.beginPath();
		ctx.arc(0, 5, 4, 0, Math.PI);
		ctx.stroke();
		ctx.fillStyle = '#f0b0bd';
		ellipse(ctx, -15, 7, 5, 3);
		ellipse(ctx, 15, 7, 5, 3);
		ctx.restore();
		for (const p of particles) {
			ctx.globalAlpha = Math.max(0, p.life / 0.7);
			ctx.fillStyle = p.color;
			ellipse(ctx, p.x, p.y, 4, 4);
		}
		ctx.globalAlpha = 1;
		ctx.restore();
	}
	function ellipse(
		ctx: CanvasRenderingContext2D,
		x: number,
		y: number,
		rx: number,
		ry: number,
		stroke = false
	) {
		ctx.beginPath();
		ctx.ellipse(x, y, rx, ry, 0, 0, Math.PI * 2);
		if (stroke) ctx.stroke();
		else ctx.fill();
	}
	function star(ctx: CanvasRenderingContext2D, x: number, y: number, radius: number) {
		ctx.beginPath();
		for (let i = 0; i < 10; i++) {
			const angle = (i * Math.PI) / 5 - Math.PI / 2,
				r = i % 2 ? radius * 0.45 : radius;
			const px = x + Math.cos(angle) * r,
				py = y + Math.sin(angle) * r;
			if (i === 0) ctx.moveTo(px, py);
			else ctx.lineTo(px, py);
		}
		ctx.closePath();
		ctx.fill();
		ctx.stroke();
	}
	function loop(time: number) {
		const dt = Math.min((time - lastTime) / 1000 || 0, 0.032);
		lastTime = time;
		if (phase === 'playing') advance(dt);
		const ctx = canvas?.getContext('2d');
		if (ctx) draw(ctx);
		frame = requestAnimationFrame(loop);
	}
	onMount(() => {
		buildWorld();
		try {
			best = Number(localStorage.getItem('rihanna-bunny-best')) || 0;
		} catch {
			/* Optional. */
		}
		const observer = new ResizeObserver(([entry]) => {
			width = Math.round(
				Math.max(
					280,
					Math.min(1600, (height * entry.contentRect.width) / Math.max(1, entry.contentRect.height))
				)
			);
		});
		observer.observe(stage);
		const previousOverflow = document.body.style.overflow;
		document.body.style.overflow = 'hidden';
		frame = requestAnimationFrame(loop);
		return () => {
			observer.disconnect();
			cancelAnimationFrame(frame);
			keys.clear();
			void audio?.close();
			document.body.style.overflow = previousOverflow;
		};
	});
</script>

<svelte:window
	onkeydown={keydown}
	onkeyup={keyup}
	onblur={() => {
		if (phase === 'playing') pause();
	}}
/>
<section class="adventure" aria-label="Rihanna's bunny birthday adventure">
	<header class="adventure-hud">
		<div><small>RIHANNA'S BIRTHDAY ADVENTURE</small><strong>{names[chapter]}</strong></div>
		<div class="adventure-score">
			<span>🥕 {collected}</span><strong>{score} points</strong>{#if combo > 1}<span class="combo"
					>×{combo} combo!</span
				>{/if}{#if shield > 0}<span>✦ magic</span>{/if}
		</div>
		<div class="adventure-actions">
			<button
				onclick={() => (muted = !muted)}
				aria-label={muted ? 'Enable game sounds' : 'Mute game sounds'}
				>{muted ? 'Sound off' : 'Sound on'}</button
			>{#if phase === 'playing' || phase === 'paused'}<button onclick={pause}
					>{phase === 'paused' ? 'Resume' : 'Pause'}</button
				>{/if}<button onclick={onclose} aria-label="Return to birthday garden">×</button>
		</div>
	</header>
	<div class="journey-track" aria-label="Journey progress">
		<div style={`width:${Math.min(100, (bunny.x / (worldEnd - 170)) * 100)}%`}></div>
	</div>
	<div bind:this={stage} class="adventure-stage">
		<canvas
			bind:this={canvas}
			{width}
			{height}
			aria-label="Meadow platform game. Move with arrows or A and D; jump with Space, W or up arrow. Collect carrots and reach Rihanna's gift."
		></canvas>
		{#if phase === 'intro'}<div class="adventure-overlay">
				<div class="adventure-card">
					<span class="card-flower">✿</span>
					<p class="eyebrow">a little bunny. a big birthday mission.</p>
					<h2>The birthday<br />meadow adventure</h2>
					<p>
						The bunnies hid Rihanna's present beyond the mushroom woods. Follow the carrot trail and
						bring it home!
					</p>
					<div class="instruction-grid">
						<span>← → / A D<br /><b>run & explore</b></span><span
							>Space / ↑<br /><b>double jump</b></span
						><span>✦ lucky stars<br /><b>bunny magic</b></span>
					</div>
					<p class="adventure-tip">
						Bounce on pink mushrooms. Hop over tickly bees.<br />Collect carrots quickly to build a
						×5 combo.
					</p>
					<button class="adventure-primary" onclick={start}>Let's go, little bunny →</button>
				</div>
			</div>
		{:else if phase === 'paused'}<div class="adventure-overlay">
				<div class="adventure-card">
					<span class="card-flower">☁</span>
					<h2>A little breather</h2>
					<p>Your bunny is waiting right here.</p>
					<button class="adventure-primary" onclick={pause}>Back to the meadow →</button>
				</div>
			</div>
		{:else if phase === 'won'}<div class="adventure-overlay celebration">
				<div class="adventure-card">
					<span class="card-flower">🎁</span>
					<p class="eyebrow">special delivery, just for you</p>
					<h2>Happy birthday,<br />Rihanna!</h2>
					<div class="birthday-letter">
						<p>Hey, wishing you an amazing birthday and new year.</p>
						<p>
							You've been someone that made the past year bearable and enjoyable. Everyday I talked
							and hung out with you was memorable. I loved every part of it. You're so amazing, past
							what anyone can tell you. I'm always rooting for u and caring for you. I hope you do
							amazing-er things this new year of yours, hope you experience more joy, more rest,
							more peace and health.
						</p>
						<p>
							I'm very, very grateful that I got to experience you up close and personal. We've been
							so close and it's really healed some parts of me and I can't thank u enough for that.
							You, being you, this caring, loving and careful person. Is just something so
							mesmerizing.
						</p>
						<p>
							I still remember the first day we texted, the first day we met up, the first day lots
							of things happened. Your mark in my life is a beautiful one. It puts a smile on my
							face.
						</p>
						<p>Happy birthday Rihanna. 🎂</p>
					</div>
					<div class="final-score">
						<strong>{score}</strong><span>points · {collected} treasures · best {best}</span>
					</div>
					<button class="adventure-primary" onclick={start}>Another bunny adventure ♡</button
					><button class="back-garden" onclick={onclose}>Back to my birthday garden</button>
				</div>
			</div>{/if}
	</div>
	<div class="adventure-bottom">
		<p aria-live="polite">{message}</p>
		<div class="adventure-controls">
			<button
				onpointerdown={(e) => hold(e, 'ArrowLeft')}
				onpointerup={release}
				onpointercancel={release}
				onlostpointercapture={release}
				aria-label="Move left">←</button
			><button
				onpointerdown={(e) => hold(e, 'ArrowRight')}
				onpointerup={release}
				onpointercancel={release}
				onlostpointercapture={release}
				aria-label="Move right">→</button
			><button
				class="jump-control"
				onpointerdown={(e) => {
					e.preventDefault();
					jump();
				}}
				aria-label="Jump or double jump">hop ↑</button
			>
		</div>
	</div>
</section>

<style>
	.adventure {
		position: fixed;
		inset: 0;
		z-index: 60;
		background: #fff5dc;
		display: flex;
		flex-direction: column;
		color: #593e54;
		font-family: 'Fredoka', sans-serif;
		overflow: auto;
		overscroll-behavior: contain;
	}
	.adventure-hud {
		width: 100%;
		max-width: none;
		align-items: center;
		padding: 16px 25px;
		gap: 16px;
		flex-wrap: wrap;
		font-family: inherit;
		letter-spacing: 0;
		color: inherit;
		flex-shrink: 0;
	}
	.adventure-hud small {
		display: block;
		font-family: 'DM Mono', monospace;
		font-size: 9px;
		letter-spacing: 0.1em;
		color: #ae6b80;
	}
	.adventure-hud strong {
		font-size: 20px;
	}
	.adventure-score {
		display: flex;
		align-items: center;
		gap: 14px;
	}
	.adventure-actions {
		display: flex;
		gap: 8px;
	}
	.adventure-actions button {
		background: #fff9eb;
		border: 1px solid #d7a8a6;
		border-radius: 15px;
		padding: 7px 12px;
		color: #68465a;
		font-size: 12px;
	}
	.combo {
		background: #f5bc9e;
		padding: 5px 10px;
		border-radius: 14px;
	}
	.journey-track {
		height: 5px;
		background: #efdac7;
		flex-shrink: 0;
	}
	.journey-track div {
		height: 100%;
		background: #d68fa5;
		transition: width 0.2s;
	}
	.adventure-stage {
		position: relative;
		flex: 1;
		min-height: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		background: #bde7e7;
	}
	.adventure-stage canvas {
		display: block;
		width: 100%;
		height: 100%;
		object-fit: contain;
	}
	.adventure-overlay {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		background: #61476538;
		overflow: auto;
		padding: 15px;
	}
	.adventure-card {
		width: min(560px, 100%);
		background: #fff8e7;
		border: 3px solid #785468;
		border-radius: 30px;
		text-align: center;
		padding: 22px 30px;
		box-shadow: 8px 9px 0 #78546830;
		position: relative;
	}
	.card-flower {
		font-size: 42px;
		color: #df8ea6;
		line-height: 1;
	}
	.adventure-card h2 {
		font-family: 'Pacifico', cursive;
		font-weight: 400;
		font-size: clamp(25px, 4vw, 40px);
		line-height: 1.18;
		color: #d78199;
		margin: 8px 0 14px;
	}
	.adventure-card p {
		font-size: 15px;
		line-height: 1.45;
		margin: 10px 0;
	}
	.adventure-card .eyebrow {
		font-size: 9px;
		margin: 8px 0;
	}
	.instruction-grid {
		display: flex;
		justify-content: center;
		gap: 10px;
		margin: 17px 0;
	}
	.instruction-grid span {
		flex: 1;
		border: 1px dashed #d19c9f;
		border-radius: 12px;
		padding: 10px 5px;
		font-size: 14px;
		background: #fffdf4;
	}
	.instruction-grid b {
		font-size: 11px;
		font-weight: 500;
		color: #a76879;
	}
	.adventure-card .adventure-tip {
		font-size: 12px;
		color: #a66c79;
	}
	.adventure-primary {
		border: 2px solid #805466;
		background: #f0acac;
		color: #603e53;
		border-radius: 20px;
		box-shadow: 3px 4px 0 #d58a94;
		padding: 12px 20px;
		font-size: 16px;
		margin-top: 7px;
	}
	.back-garden {
		display: block;
		margin: 14px auto 0;
		background: none;
		border: 0;
		color: #a26b7c;
		font-size: 13px;
		text-decoration: underline;
	}
	.adventure-bottom {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 15px;
		padding: 10px 24px;
		background: #fff5dc;
		flex-shrink: 0;
	}
	.adventure-bottom p {
		font-size: 13px;
		margin: 0;
		color: #996577;
	}
	.adventure-controls {
		display: flex;
		gap: 9px;
		flex-shrink: 0;
	}
	.adventure-controls button {
		touch-action: none;
		user-select: none;
		border: 2px solid #795569;
		background: #fffaf0;
		color: #795569;
		border-radius: 15px;
		width: 58px;
		height: 48px;
		font-size: 24px;
		box-shadow: 2px 3px 0 #dbb4a4;
	}
	.adventure-controls button:active {
		background: #f5c1ae;
		transform: translateY(2px);
	}
	.adventure-controls .jump-control {
		width: 90px;
		font-size: 18px;
		background: #f2bab4;
	}
	.final-score strong {
		display: block;
		color: #d68199;
		font-size: 44px;
	}
	.final-score span {
		font-size: 12px;
		color: #a27180;
	}
	button:focus-visible {
		outline: 3px solid #967bc3;
		outline-offset: 3px;
	}
	@media (max-width: 650px) {
		.adventure-hud {
			padding: 10px 13px;
			gap: 8px;
		}
		.adventure-hud strong {
			font-size: 15px;
		}
		.adventure-hud small {
			font-size: 7px;
		}
		.adventure-score {
			gap: 6px;
			font-size: 12px;
		}
		.adventure-actions button {
			padding: 5px 7px;
			font-size: 10px;
		}
		.adventure-bottom {
			flex-direction: column;
			padding: 10px;
			gap: 8px;
		}
		.adventure-bottom p {
			font-size: 11px;
			text-align: center;
		}
		.adventure-stage {
			min-height: 250px;
		}
		.adventure-card {
			padding: 14px 17px;
			border-radius: 23px;
		}
		.adventure-card p {
			font-size: 12px;
		}
		.card-flower {
			font-size: 30px;
		}
		.instruction-grid {
			margin: 10px 0;
		}
		.instruction-grid span {
			font-size: 11px;
			padding: 7px 3px;
		}
		.adventure-primary {
			font-size: 14px;
		}
		.combo {
			padding: 3px 5px;
		}
	}
	@media (max-height: 600px) and (min-width: 651px) {
		.adventure-card {
			padding: 10px 24px;
		}
		.adventure-card h2 {
			font-size: 26px;
			margin: 5px 0;
		}
		.adventure-card p {
			font-size: 12px;
			margin: 6px 0;
		}
		.card-flower {
			font-size: 25px;
		}
		.instruction-grid {
			margin: 7px 0;
		}
		.adventure-hud {
			padding: 8px 20px;
		}
		.adventure-bottom {
			padding: 5px 20px;
		}
	}
	.celebration {
		align-items: flex-start;
		overscroll-behavior: contain;
	}
	.celebration .adventure-card {
		width: min(680px, 100%);
		flex-shrink: 0;
		margin: auto 0;
	}
	.birthday-letter {
		text-align: left;
		padding: 4px 0 16px;
		border-bottom: 1px dashed #d7a8a6;
	}
	.birthday-letter p {
		font-size: 16px;
		line-height: 1.75;
		margin: 0 0 18px;
	}
</style>
