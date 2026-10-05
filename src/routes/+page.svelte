<script lang="ts">
	let treats = 0;
	let litCandles = 0;
	let wish = '';
	let wishSent = false;
	let bunnyMood = 'ready for cake';
	let hop = 0;
	let sparkles: { id: number; x: number; y: number; emoji: string }[] = [];
	function feedBunny() { treats += 1; bunnyMood = treats === 1 ? 'crunching happily' : treats < 5 ? 'so very loved' : 'carrot-powered!'; burst(['✦', '♡', '•'], 7); }
	function lightCandle(index: number) { if (index < litCandles) return; litCandles = index + 1; bunnyMood = litCandles === 5 ? 'making a birthday wish!' : 'watching the candles glow'; if (litCandles === 5) burst(['✦', '★', '♡', '✧'], 20); }
	function burst(emojis: string[], total = 12) { const created = Array.from({ length: total }, (_, i) => ({ id: Date.now() + i, x: 20 + Math.random() * 60, y: 25 + Math.random() * 45, emoji: emojis[Math.floor(Math.random() * emojis.length)] })); sparkles = [...sparkles, ...created]; setTimeout(() => (sparkles = sparkles.filter((s) => !created.includes(s))), 1100); }
	function makeWish() { if (!wish.trim()) return; wishSent = true; bunnyMood = 'keeping your wish extra safe'; burst(['✦', '★', '♡', '❀'], 28); }
	function hopAway() { hop += 1; bunnyMood = 'doing a tiny happy hop'; burst(['♡', '✦'], 8); }
</script>
<svelte:head><title>A birthday burrow for Rihanna</title><meta name="description" content="A little birthday bunny game made especially for Rihanna." /><link rel="preconnect" href="https://fonts.googleapis.com" /><link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" /><link href="https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=Fredoka:wght@500;600;700&family=Pacifico&display=swap" rel="stylesheet" /></svelte:head>
<main class="burrow">
	<div class="paper-grain"></div><div class="cloud cloud-one"></div><div class="cloud cloud-two"></div>
	<header><div class="postmark">BUNNY MAIL<br /><span>delivered with love</span></div><div class="tiny-date">OCTOBER 05 · A VERY GOOD DAY</div><button class="sound" aria-label="Make a tiny chime" onclick={() => burst(['♪', '✦'], 8)}>♪</button></header>
	<section class="hero" aria-labelledby="birthday-title"><p class="eyebrow">a birthday burrow, made for</p><h1 id="birthday-title">Rihanna!</h1><p class="subtitle">Somebunny thinks you deserve the softest,<br />sweetest, most sparkly day.</p><div class="ribbon">today's special: <strong>you</strong> ♡</div></section>
	<section class="playground" aria-label="Birthday bunny game"><div class="side-note left-note">tap things!<br /><span>the bunny is curious</span></div><div class="side-note right-note">birthday rule #1<br /><span>make a wish</span></div><div class="garden"><div class="grass grass-back"></div><div class="flowers f1">✿</div><div class="flowers f2">✦</div><div class="flowers f3">✿</div><div class="bush b1"></div><div class="bush b2"></div><button class:jump={hop % 2 === 1} class="bunny" onclick={hopAway} aria-label="Make the birthday bunny hop"><span class="ear ear-left"></span><span class="ear ear-right"></span><span class="face"><i></i><i></i><b>ᴗ</b></span><span class="cheek cheek-left"></span><span class="cheek cheek-right"></span><span class="paws">⌣ &nbsp;⌣</span></button><div class="speech">{bunnyMood} <span>♡</span></div><div class="cake-area"><div class="cake-label">tap each candle!</div><div class="cake"><div class="candles">{#each Array(5) as _, i}<button class:lit={i < litCandles} class="candle" onclick={() => lightCandle(i)} aria-label={`Light candle ${i + 1}`}><span class="flame">♥</span></button>{/each}</div><div class="icing">♡ &nbsp; ✦ &nbsp; ♡</div><div class="cake-layer"></div><div class="plate"></div></div></div><button class="carrot" onclick={feedBunny} aria-label="Give the bunny a carrot"><span>✦</span> <b>carrot snack</b><em>+{treats}</em></button><div class="grass grass-front"></div></div></section>
	<section class="wish-card" aria-labelledby="wish-heading"><div class="stamp">♥<br /><small>R</small></div><div><p class="eyebrow">one last birthday ritual</p><h2 id="wish-heading">Make a little wish</h2>{#if wishSent}<p class="sent">Wish tucked safely beneath the clover. May it find you soon. ✦</p>{:else}<div class="wish-form"><input bind:value={wish} onkeydown={(e) => e.key === 'Enter' && makeWish()} placeholder="write it here..." aria-label="Your birthday wish" /><button onclick={makeWish}>send it ✦</button></div>{/if}</div></section>
	<footer>made with a whole garden of love for <strong>Rihanna</strong> · happy birthday ♡</footer>
	{#each sparkles as sparkle (sparkle.id)}<span class="sparkle" style={`left:${sparkle.x}%; top:${sparkle.y}%`}>{sparkle.emoji}</span>{/each}
</main>
