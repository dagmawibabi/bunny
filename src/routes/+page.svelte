<script lang="ts">
	import BirthdayAdventure from '#lib/BirthdayAdventure.svelte';
	let treats = $state(0);
	let litCandles = $state(0);
	let wish = $state('');
	let wishSent = $state(false);
	let bunnyMood = $state('ready for cake');
	let hop = $state(0);
	let sparkles = $state<{ id: number; x: number; y: number; emoji: string }[]>([]);
	let musicPlaying = $state(false);
	let audioContext: AudioContext | null = null;
	let musicTimer: ReturnType<typeof setTimeout> | null = null;
	let melodyStep = 0;
	let gameOpen = $state(false);
	const melody = [523.25, 659.25, 783.99, 659.25, 698.46, 783.99, 1046.5, 783.99];
	function feedBunny() {
		treats += 1;
		bunnyMood =
			treats === 1 ? 'crunching happily' : treats < 5 ? 'so very loved' : 'carrot-powered!';
		burst(['✦', '♡', '•'], 7);
	}
	function lightCandle(index: number) {
		if (index < litCandles) return;
		litCandles = index + 1;
		bunnyMood = litCandles === 5 ? 'making a birthday wish!' : 'watching the candles glow';
		if (litCandles === 5) burst(['✦', '★', '♡', '✧'], 20);
	}
	function burst(emojis: string[], total = 12) {
		const created = Array.from({ length: total }, (_, i) => ({
			id: Date.now() + i,
			x: 20 + Math.random() * 60,
			y: 25 + Math.random() * 45,
			emoji: emojis[Math.floor(Math.random() * emojis.length)]
		}));
		sparkles = [...sparkles, ...created];
		setTimeout(() => (sparkles = sparkles.filter((s) => !created.includes(s))), 1100);
	}
	function makeWish() {
		if (!wish.trim()) return;
		wishSent = true;
		bunnyMood = 'keeping your wish extra safe';
		burst(['✦', '★', '♡', '❀'], 28);
	}
	function hopAway() {
		hop += 1;
		bunnyMood = 'doing a tiny happy hop';
		burst(['♡', '✦'], 8);
	}
	function playMelodyNote() {
		if (!audioContext || !musicPlaying) return;
		const now = audioContext.currentTime;
		const oscillator = audioContext.createOscillator();
		const gain = audioContext.createGain();
		oscillator.type = 'sine';
		oscillator.frequency.value = melody[melodyStep % melody.length];
		gain.gain.setValueAtTime(0.0001, now);
		gain.gain.exponentialRampToValueAtTime(0.055, now + 0.025);
		gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.28);
		oscillator.connect(gain).connect(audioContext.destination);
		oscillator.start(now);
		oscillator.stop(now + 0.3);
		melodyStep += 1;
		musicTimer = setTimeout(playMelodyNote, 360);
	}
	function toggleMusic() {
		musicPlaying = !musicPlaying;
		if (!musicPlaying) {
			if (musicTimer) clearTimeout(musicTimer);
			musicTimer = null;
			return;
		}
		audioContext ??= new AudioContext();
		audioContext.resume();
		burst(['♪', '♫', '✦'], 10);
		playMelodyNote();
	}
	function openGame() {
		gameOpen = true;
	}
	function closeGame() {
		gameOpen = false;
	}
</script>

<svelte:head
	><title>A birthday burrow for Rihanna</title><meta
		name="description"
		content="A little birthday bunny game made especially for Rihanna."
	/><link rel="preconnect" href="https://fonts.googleapis.com" /><link
		rel="preconnect"
		href="https://fonts.gstatic.com"
		crossorigin="anonymous"
	/><link
		href="https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=Fredoka:wght@500;600;700&family=Pacifico&display=swap"
		rel="stylesheet"
	/></svelte:head
>
<main class="burrow">
	<div class="paper-grain"></div>
	<div class="cloud cloud-one"></div>
	<div class="cloud cloud-two"></div>
	<header>
		<div class="postmark">BUNNY MAIL<br /><span>delivered with love</span></div>
		<div class="tiny-date">OCTOBER 05 · A VERY GOOD DAY</div>
		<button
			class:playing={musicPlaying}
			class="sound"
			aria-pressed={musicPlaying}
			aria-label={musicPlaying ? 'Pause birthday music' : 'Play birthday music'}
			onclick={toggleMusic}>{musicPlaying ? 'Ⅱ' : '♪'}</button
		>
	</header>
	<section class="hero" aria-labelledby="birthday-title">
		<p class="eyebrow">a birthday burrow, made for</p>
		<h1 id="birthday-title">Rihanna!</h1>
		<p class="subtitle">
			Somebunny thinks you deserve the softest,<br />sweetest, most sparkly day.
		</p>
		<div class="ribbon">today's special: <strong>you</strong> ♡</div>
	</section>
	<section class="play-guide" aria-label="How to play">
		<span><b>01</b> feed a carrot</span><span><b>02</b> light five candles</span><span
			><b>03</b> tap bunny to hop</span
		><span><b>04</b> send a wish</span>
	</section>
	<section class="playground" aria-label="Birthday bunny game">
		<div class="side-note left-note">tap things!<br /><span>the bunny is curious</span></div>
		<div class="side-note right-note">birthday rule #1<br /><span>make a wish</span></div>
		<div class="garden">
			<div class="grass grass-back"></div>
			<div class="flowers f1">✿</div>
			<div class="flowers f2">✦</div>
			<div class="flowers f3">✿</div>
			<div class="bush b1"></div>
			<div class="bush b2"></div>
			<button
				class:jump={hop % 2 === 1}
				class="bunny"
				onclick={hopAway}
				aria-label="Make the birthday bunny hop"
				><span class="ear ear-left"></span><span class="ear ear-right"></span><span class="face"
					><i></i><i></i><b>ᴗ</b></span
				><span class="cheek cheek-left"></span><span class="cheek cheek-right"></span><span
					class="paws">⌣ &nbsp;⌣</span
				></button
			>
			<div class="speech">{bunnyMood} <span>♡</span></div>
			<div class="cake-area">
				<div class="cake-label">tap each candle!</div>
				<div class="cake">
					<div class="candles">
						{#each Array(5) as _, i}<button
								class:lit={i < litCandles}
								class="candle"
								onclick={() => lightCandle(i)}
								aria-label={`Light candle ${i + 1}`}><span class="flame">♥</span></button
							>{/each}
					</div>
					<div class="icing">♡ &nbsp; ✦ &nbsp; ♡</div>
					<div class="cake-layer"></div>
					<div class="plate"></div>
				</div>
			</div>
			<button class="carrot" onclick={feedBunny} aria-label="Give the bunny a carrot"
				><span>✦</span> <b>carrot snack</b><em>+{treats}</em></button
			>
			<div class="grass grass-front"></div>
		</div>
	</section>
	<section class="wish-card" aria-labelledby="wish-heading">
		<div class="stamp">♥<br /><small>R</small></div>
		<div>
			<p class="eyebrow">one last birthday ritual</p>
			<h2 id="wish-heading">Make a little wish</h2>
			{#if wishSent}<p class="sent">
					Wish tucked safely beneath the clover. May it find you soon. ✦
				</p>{:else}<div class="wish-form">
					<input
						bind:value={wish}
						onkeydown={(e) => e.key === 'Enter' && makeWish()}
						placeholder="write it here..."
						aria-label="Your birthday wish"
					/><button onclick={makeWish}>send it ✦</button>
				</div>{/if}
		</div>
	</section>
	{#if litCandles === 5 && treats > 0 && wishSent}<section class="game-unlock">
			<p>the birthday meadow is open!</p>
			<button onclick={openGame}>Enter the bunny adventure <span>→</span></button>
		</section>{/if}
	<footer>made with a whole garden of love for <strong>Rihanna</strong> · happy birthday ♡</footer>
	{#each sparkles as sparkle (sparkle.id)}<span
			class="sparkle"
			style={`left:${sparkle.x}%; top:${sparkle.y}%`}>{sparkle.emoji}</span
		>{/each}
	{#if gameOpen}<BirthdayAdventure onclose={closeGame} />{/if}
</main>
