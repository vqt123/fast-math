<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

	type Screen = 'menu' | 'game' | 'result' | 'history';
	type Config = { add: boolean; sub: boolean; mul: boolean; div: boolean; negatives: boolean; doubleDigits: boolean };
	type Problem = { question: string; answer: number };
	type HistoryItem = { q: string; correct: number; user: number; isCorrect: boolean };
	type GameRecord = { id: number; date: string; score: number; accuracy: number; config: Config };

	let screen: Screen = 'menu';
	let config: Config = { add: true, sub: true, mul: true, div: false, negatives: false, doubleDigits: false };
	let score = 0;
	let timeLeft = 60;
	let timerInterval: ReturnType<typeof setInterval> | null = null;
	let currentProblem: Problem = { question: '', answer: 0 };
	let sessionHistory: HistoryItem[] = [];
	let pastGames: GameRecord[] = [];
	let userInput = '';
	let inputEl: HTMLInputElement;
	let inputClass = 'border-zinc-700 focus:border-purple-500 text-white';
	let shaking = false;

	onMount(() => {
		if (browser) {
			const saved = localStorage.getItem('mathSprintHistory');
			if (saved) pastGames = JSON.parse(saved);
		}
	});

	function toggleConfig(key: keyof Config) {
		if (key !== 'negatives' && key !== 'doubleDigits') {
			const activeOps = (['add', 'sub', 'mul', 'div'] as const).filter(k => config[k]).length;
			if (activeOps === 1 && config[key]) return;
		}
		config[key] = !config[key];
	}

	function startGame() {
		score = 0;
		timeLeft = 60;
		sessionHistory = [];
		userInput = '';
		inputClass = 'border-zinc-700 focus:border-purple-500 text-white';
		screen = 'game';
		generateProblem();
		if (timerInterval) clearInterval(timerInterval);
		timerInterval = setInterval(() => {
			timeLeft--;
			if (timeLeft <= 0) endGame();
		}, 1000);
		setTimeout(() => inputEl?.focus(), 100);
	}

	function abandonGame() {
		if (timerInterval) clearInterval(timerInterval);
		screen = 'menu';
	}

	function generateProblem() {
		const ops: string[] = [];
		if (config.add) ops.push('+');
		if (config.sub) ops.push('-');
		if (config.mul) ops.push('*');
		if (config.div) ops.push('/');
		const operator = ops[Math.floor(Math.random() * ops.length)];
		const max = config.doubleDigits ? 99 : 12;
		const min = config.doubleDigits ? 10 : 1;
		let num1 = Math.floor(Math.random() * (max - min + 1)) + min;
		let num2 = Math.floor(Math.random() * (max - min + 1)) + min;
		if (config.negatives) {
			if (Math.random() > 0.5) num1 *= -1;
			if (Math.random() > 0.5) num2 *= -1;
		}
		let answer = 0;
		switch (operator) {
			case '+': answer = num1 + num2; break;
			case '-': answer = num1 - num2; break;
			case '*': answer = num1 * num2; break;
			case '/': num1 = num1 * num2; answer = num1 / num2; break;
		}
		const fmt = (n: number) => n < 0 ? `(${n})` : `${n}`;
		const opSymbol = operator === '/' ? '÷' : operator === '*' ? '×' : operator;
		currentProblem = { question: `${fmt(num1)} ${opSymbol} ${fmt(num2)}`, answer };
		userInput = '';
		inputClass = 'border-zinc-700 focus:border-purple-500 text-white';
	}

	function handleAnswer(e: Event) {
		e.preventDefault();
		const val = parseInt(userInput);
		if (isNaN(val)) return;
		const isCorrect = val === currentProblem.answer;
		sessionHistory = [...sessionHistory, { q: currentProblem.question, correct: currentProblem.answer, user: val, isCorrect }];
		if (isCorrect) {
			score++;
			inputClass = 'border-emerald-500 text-emerald-500';
			setTimeout(generateProblem, 150);
		} else {
			inputClass = 'border-red-500 text-red-500';
			shaking = true;
			setTimeout(() => { shaking = false; generateProblem(); }, 400);
		}
	}

	function endGame() {
		if (timerInterval) clearInterval(timerInterval);
		const correctCount = sessionHistory.filter(h => h.isCorrect).length;
		const accuracy = sessionHistory.length > 0 ? Math.round((correctCount / sessionHistory.length) * 100) : 0;
		const record: GameRecord = { id: Date.now(), date: new Date().toISOString(), score, accuracy, config: { ...config } };
		pastGames = [record, ...pastGames].slice(0, 50);
		if (browser) localStorage.setItem('mathSprintHistory', JSON.stringify(pastGames));
		screen = 'result';
	}

	$: progressPct = (timeLeft / 60) * 100;
	$: accuracy = sessionHistory.length > 0 ? Math.round((sessionHistory.filter(h => h.isCorrect).length / sessionHistory.length) * 100) : 0;
	$: missed = sessionHistory.filter(h => !h.isCorrect).slice(0, 3);

	function configKey(c: Config): string {
		const ops = [];
		if (c.add) ops.push('+');
		if (c.sub) ops.push('-');
		if (c.mul) ops.push('×');
		if (c.div) ops.push('÷');
		const mods = [];
		if (c.negatives) mods.push('neg');
		if (c.doubleDigits) mods.push('2x');
		return ops.join('') + (mods.length ? ` (${mods.join(', ')})` : '');
	}

	function getLeaderboards(games: GameRecord[]): { key: string; config: Config; entries: GameRecord[] }[] {
		const map = new Map<string, { config: Config; entries: GameRecord[] }>();
		for (const g of games) {
			const k = configKey(g.config);
			if (!map.has(k)) map.set(k, { config: g.config, entries: [] });
			map.get(k)!.entries.push(g);
		}
		// Sort entries by score desc, then by date desc
		for (const v of map.values()) {
			v.entries.sort((a, b) => b.score - a.score || new Date(b.date).getTime() - new Date(a.date).getTime());
			v.entries = v.entries.slice(0, 10); // Top 10
		}
		// Sort leaderboards by most recent activity
		return Array.from(map.entries())
			.map(([key, val]) => ({ key, ...val }))
			.sort((a, b) => new Date(b.entries[0]?.date || 0).getTime() - new Date(a.entries[0]?.date || 0).getTime());
	}

	$: leaderboards = getLeaderboards(pastGames);
</script>

<svelte:window on:keydown={(e) => { if (e.key === 'Enter' && screen === 'menu') startGame(); }} />

<div class="min-h-screen flex flex-col items-center justify-center overflow-hidden selection:bg-purple-500/30">
	<!-- MENU -->
	{#if screen === 'menu'}
		<div class="w-full max-w-md p-4 fade-in">
			<div class="mb-8 text-center space-y-2">
				<div class="w-16 h-16 bg-gradient-to-br from-purple-600 to-blue-600 rounded-2xl mx-auto flex items-center justify-center shadow-lg shadow-purple-900/20 mb-6">
					<svg class="w-8 h-8 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24"><rect x="4" y="2" width="16" height="20" rx="2"/><line x1="8" y1="6" x2="16" y2="6"/><line x1="8" y1="10" x2="16" y2="10"/><line x1="8" y1="14" x2="12" y2="14"/><line x1="8" y1="18" x2="12" y2="18"/></svg>
				</div>
				<h1 class="text-3xl font-semibold tracking-tight">Mental Sprint</h1>
				<p class="text-zinc-500">Master arithmetic in 60 seconds.</p>
			</div>
			<div class="space-y-6">
				<div class="space-y-3">
					<div class="flex items-center gap-2 text-xs font-medium text-zinc-500 uppercase tracking-wider mb-2">
						<svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/></svg>
						<span>Operations</span>
					</div>
					<div class="grid grid-cols-2 gap-3">
						{#each [['add', 'Addition (+)'], ['sub', 'Subtraction (-)'], ['mul', 'Multiplication (×)'], ['div', 'Division (÷)']] as [key, label]}
							<button on:click={() => toggleConfig(key as keyof Config)} class="group flex items-center justify-between w-full p-4 rounded-lg border transition-all {config[key as keyof Config] ? 'bg-zinc-800/50 border-purple-500/50 shadow-[0_0_15px_rgba(168,85,247,0.15)]' : 'bg-zinc-900 border-zinc-800 hover:border-zinc-700'}">
								<span class="text-sm font-medium {config[key as keyof Config] ? 'text-zinc-100' : 'text-zinc-400'}">{label}</span>
								<div class="w-5 h-5 rounded flex items-center justify-center transition-colors {config[key as keyof Config] ? 'bg-purple-600 border-purple-600' : 'border border-zinc-700 bg-zinc-800'}">
									{#if config[key as keyof Config]}<svg class="w-3 h-3 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24"><polyline points="20 6 9 17 4 12"/></svg>{/if}
								</div>
							</button>
						{/each}
					</div>
				</div>
				<div class="space-y-3">
					<div class="flex items-center gap-2 text-xs font-medium text-zinc-500 uppercase tracking-wider mb-2">
						<svg class="w-3 h-3" fill="none" stroke="currentColor" viewBox="0 0 24 24"><polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/></svg>
						<span>Difficulty</span>
					</div>
					<div class="grid grid-cols-2 gap-3">
						<button on:click={() => toggleConfig('negatives')} class="group flex items-center justify-between w-full p-4 rounded-lg border transition-all {config.negatives ? 'bg-zinc-800/50 border-purple-500/50 shadow-[0_0_15px_rgba(168,85,247,0.15)]' : 'bg-zinc-900 border-zinc-800 hover:border-zinc-700'}">
							<span class="text-sm font-medium {config.negatives ? 'text-zinc-100' : 'text-zinc-400'}">Negatives</span>
							<div class="w-5 h-5 rounded flex items-center justify-center transition-colors {config.negatives ? 'bg-purple-600 border-purple-600' : 'border border-zinc-700 bg-zinc-800'}">
								{#if config.negatives}<svg class="w-3 h-3 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24"><polyline points="20 6 9 17 4 12"/></svg>{/if}
							</div>
						</button>
						<button on:click={() => toggleConfig('doubleDigits')} class="group flex items-center justify-between w-full p-4 rounded-lg border transition-all {config.doubleDigits ? 'bg-zinc-800/50 border-purple-500/50 shadow-[0_0_15px_rgba(168,85,247,0.15)]' : 'bg-zinc-900 border-zinc-800 hover:border-zinc-700'}">
							<span class="text-sm font-medium {config.doubleDigits ? 'text-zinc-100' : 'text-zinc-400'}">Double Digits</span>
							<div class="w-5 h-5 rounded flex items-center justify-center transition-colors {config.doubleDigits ? 'bg-purple-600 border-purple-600' : 'border border-zinc-700 bg-zinc-800'}">
								{#if config.doubleDigits}<svg class="w-3 h-3 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24"><polyline points="20 6 9 17 4 12"/></svg>{/if}
							</div>
						</button>
					</div>
				</div>
				<div class="flex gap-3">
					<button on:click={() => screen = 'history'} class="h-12 w-14 bg-zinc-900 border border-zinc-800 hover:bg-zinc-800 hover:border-zinc-700 text-zinc-400 rounded-lg transition-all flex items-center justify-center" title="View History">
						<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg>
					</button>
					<button on:click={startGame} class="flex-1 h-12 bg-zinc-100 hover:bg-white text-zinc-950 font-medium rounded-lg transition-all flex items-center justify-center gap-2 shadow-[0_0_20px_rgba(255,255,255,0.1)] active:scale-[0.98]">
						Start Sprint <span class="hidden md:inline-flex items-center justify-center min-w-[20px] h-5 px-1.5 ml-2 text-[10px] font-bold text-zinc-400 bg-zinc-800 border border-zinc-700 rounded font-mono">↵</span>
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- GAME -->
	{#if screen === 'game'}
		<div class="w-full h-screen flex flex-col items-center justify-center relative fade-in">
			<button on:click={abandonGame} class="absolute top-4 left-4 p-2 text-zinc-500 hover:text-white hover:bg-zinc-900 rounded-full transition-all z-20" title="Abandon">
				<svg class="w-7 h-7" fill="none" stroke="currentColor" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10"/><line x1="15" y1="9" x2="9" y2="15"/><line x1="9" y1="9" x2="15" y2="15"/></svg>
			</button>
			<div class="fixed top-0 left-0 w-full h-1 bg-zinc-900">
				<div class="h-full bg-gradient-to-r from-purple-500 to-blue-500 transition-all duration-1000 ease-linear" style="width: {progressPct}%"></div>
			</div>
			<div class="w-full max-w-2xl px-4">
				<div class="flex justify-between items-center mb-12">
					<div class="flex flex-col">
						<span class="text-zinc-500 text-xs uppercase tracking-wider font-medium">Time</span>
						<span class="text-2xl font-mono {timeLeft < 10 ? 'text-red-400' : 'text-zinc-200'}">00:{timeLeft.toString().padStart(2, '0')}</span>
					</div>
					<div class="flex flex-col items-end">
						<span class="text-zinc-500 text-xs uppercase tracking-wider font-medium">Score</span>
						<span class="text-2xl font-mono text-emerald-400">{score}</span>
					</div>
				</div>
				<div class="text-center space-y-12">
					<div class="h-32 flex items-center justify-center">
						<h2 class="text-7xl md:text-8xl font-medium tracking-tight">{currentProblem.question}</h2>
					</div>
					<form on:submit={handleAnswer} class="relative max-w-xs mx-auto">
						<input bind:this={inputEl} bind:value={userInput} type="text" inputmode="numeric" pattern="[0-9\-]*" class="w-full bg-transparent border-b-2 text-center text-4xl py-4 focus:outline-none transition-colors font-mono placeholder-zinc-800 {inputClass} {shaking ? 'animate-shake' : ''}" placeholder="?" autocomplete="off" />
						<div class="absolute right-0 top-1/2 -translate-y-1/2 translate-x-12 opacity-50 hidden md:block">
							<span class="inline-flex items-center justify-center min-w-[20px] h-5 px-1.5 ml-2 text-[10px] font-bold text-zinc-400 bg-zinc-800 border border-zinc-700 rounded font-mono">↵</span>
						</div>
					</form>
				</div>
			</div>
		</div>
	{/if}

	<!-- RESULT -->
	{#if screen === 'result'}
		<div class="w-full max-w-md p-4 fade-in">
			<div class="bg-zinc-900/50 border border-zinc-800 rounded-2xl p-8 mb-6 text-center">
				<div class="w-16 h-16 bg-zinc-800 rounded-full flex items-center justify-center mx-auto mb-6">
					<svg class="w-8 h-8 text-yellow-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>
				</div>
				<h2 class="text-3xl font-semibold mb-2">Time's Up!</h2>
				<p class="text-zinc-500 mb-8">Here is how you performed</p>
				<div class="grid grid-cols-2 gap-4 mb-8">
					<div class="bg-zinc-950 border border-zinc-800 p-4 rounded-xl">
						<div class="text-zinc-500 text-xs uppercase tracking-wider mb-1">Total Score</div>
						<div class="text-3xl font-mono text-purple-400">{score}</div>
					</div>
					<div class="bg-zinc-950 border border-zinc-800 p-4 rounded-xl">
						<div class="text-zinc-500 text-xs uppercase tracking-wider mb-1">Accuracy</div>
						<div class="text-3xl font-mono {accuracy > 80 ? 'text-emerald-400' : 'text-zinc-300'}">{accuracy}%</div>
					</div>
				</div>
				<div class="grid grid-cols-2 gap-3">
					<button on:click={() => screen = 'menu'} class="w-full py-3 bg-zinc-800 hover:bg-zinc-700 text-zinc-300 rounded-lg transition-colors font-medium text-sm">Settings</button>
					<button on:click={startGame} class="w-full py-3 bg-white hover:bg-zinc-200 text-zinc-950 rounded-lg transition-colors font-medium text-sm flex items-center justify-center gap-2">
						<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/><path d="M3 3v5h5"/></svg> Play Again
					</button>
				</div>
			</div>
			{#if missed.length > 0}
				<div class="space-y-2">
					<p class="text-xs font-medium text-zinc-500 uppercase tracking-wider pl-2">Missed Problems</p>
					{#each missed as item}
						<div class="flex justify-between items-center bg-zinc-900/30 border border-red-500/20 px-4 py-3 rounded-lg text-sm">
							<span class="text-zinc-300">{item.q}</span>
							<div class="flex gap-4">
								<span class="text-red-400 line-through decoration-red-500/50">{item.user}</span>
								<span class="text-emerald-400">{item.correct}</span>
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{/if}

	<!-- LEADERBOARDS -->
	{#if screen === 'history'}
		<div class="w-full max-w-md p-4 fade-in h-screen flex flex-col">
			<div class="flex items-center gap-4 mb-6 pt-4 shrink-0">
				<button on:click={() => screen = 'menu'} class="p-2 hover:bg-zinc-900 rounded-full transition-colors text-zinc-400 hover:text-white" aria-label="Back">
					<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><line x1="19" y1="12" x2="5" y2="12"/><polyline points="12 19 5 12 12 5"/></svg>
				</button>
				<h2 class="text-xl font-semibold">Leaderboards</h2>
			</div>
			{#if leaderboards.length === 0}
				<div class="text-center py-20 text-zinc-600">
					<svg class="w-12 h-12 mx-auto mb-4 opacity-20" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>
					<p>No games played yet.</p>
				</div>
			{:else}
				<div class="space-y-6 overflow-y-auto flex-1 pb-4">
					{#each leaderboards as board}
						<div class="bg-zinc-900/30 border border-zinc-800/50 rounded-xl overflow-hidden">
							<div class="px-4 py-3 border-b border-zinc-800/50 flex items-center gap-2">
								<svg class="w-4 h-4 text-yellow-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/><path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/><path d="M4 22h16"/><path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/><path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/><path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/></svg>
								<span class="font-medium text-sm">{board.key}</span>
							</div>
							<div class="divide-y divide-zinc-800/30">
								{#each board.entries as entry, i}
									<div class="px-4 py-2.5 flex items-center gap-3 {i === 0 ? 'bg-yellow-500/5' : i === 1 ? 'bg-zinc-400/5' : i === 2 ? 'bg-amber-700/5' : ''}">
										<div class="w-6 text-center font-mono text-sm {i === 0 ? 'text-yellow-500' : i === 1 ? 'text-zinc-400' : i === 2 ? 'text-amber-600' : 'text-zinc-600'}">
											{#if i === 0}🥇{:else if i === 1}🥈{:else if i === 2}🥉{:else}{i + 1}{/if}
										</div>
										<div class="flex-1 min-w-0">
											<div class="text-xs text-zinc-500">{new Date(entry.date).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}</div>
										</div>
										<div class="text-right">
											<span class="font-mono font-medium {i === 0 ? 'text-yellow-500' : 'text-emerald-400'}">{entry.score}</span>
											<span class="text-xs text-zinc-600 ml-1">pts</span>
										</div>
									</div>
								{/each}
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{/if}
</div>
