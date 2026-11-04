<script lang="ts">
	import { Check, Eye, EyeOff, RefreshCw, RotateCcw, Settings2, Sparkles } from '@lucide/svelte';

	type Level = 'one' | 'negative' | 'two';
	type PracticeMode = 'guided' | 'challenge';
	type Problem = {
		expression: string;
		values: { name: string; value: number }[];
		terms: { coefficient: number; variable: string }[];
		constant: number;
		substitution: string;
		combine: string;
		answer: number;
		steps: string[];
	};

	const random = (min: number, max: number) => Math.floor(Math.random() * (max - min + 1)) + min;
	const sign = (value: number) => (value < 0 ? '−' : '+');
	const absolute = (value: number) => Math.abs(value);
	const signed = (value: number) => `${sign(value)} ${absolute(value)}`;

	function makeProblem(level: Level): Problem {
		const coefficient = random(2, 6);
		if (level === 'two') {
			const secondCoefficient = random(2, 5);
			const x = random(2, 8);
			const y = random(2, 8);
			const constant = random(-9, 9) || 4;
			const firstPart = coefficient * x;
			const secondPart = secondCoefficient * y;
			const answer = firstPart + secondPart + constant;
			return {
				expression: `${coefficient}x + ${secondCoefficient}y ${signed(constant)}`,
				values: [
					{ name: 'x', value: x },
					{ name: 'y', value: y }
				],
				terms: [
					{ coefficient, variable: 'x' },
					{ coefficient: secondCoefficient, variable: 'y' }
				],
				constant,
				substitution: `${coefficient}(${x}) + ${secondCoefficient}(${y}) ${signed(constant)}`,
				combine: `${firstPart} + ${secondPart} ${signed(constant)}`,
				answer,
				steps: ['Read the values for x and y.', 'Substitute both values.', 'Multiply.', 'Add or subtract.', 'State the answer.']
			};
		}

		const x = level === 'negative' ? random(-8, 8) || -3 : random(2, 9);
		const constant = level === 'negative' ? random(-12, 12) || -5 : random(2, 12);
		const product = coefficient * x;
		const answer = product + constant;
		return {
			expression: `${coefficient}x ${signed(constant)}`,
			values: [{ name: 'x', value: x }],
			terms: [{ coefficient, variable: 'x' }],
			constant,
			substitution: `${coefficient}(${x}) ${signed(constant)}`,
			combine: `${product} ${signed(constant)}`,
			answer,
			steps: ['Read the value of x.', 'Substitute the value for x.', 'Multiply.', 'Add or subtract.', 'State the answer.']
		};
	}

	let level = $state<Level>('one');
	let mode = $state<PracticeMode>('guided');
	let problem = $state<Problem>(makeProblem('one'));
	let complete = $state<boolean[]>(Array(5).fill(false));
	let revealed = $state(false);
	let settingsOpen = $state(false);
	let problemNumber = $state(1);
	let substitutionAnswers = $state<Record<string, string>>({});
	let substitutionFeedback = $state<'idle' | 'correct' | 'incorrect'>('idle');

	function nextProblem() {
		problem = makeProblem(level);
		complete = Array(5).fill(false);
		revealed = false;
		substitutionAnswers = {};
		substitutionFeedback = 'idle';
		problemNumber += 1;
	}

	function resetBoard() {
		complete = Array(5).fill(false);
		revealed = false;
		substitutionAnswers = {};
		substitutionFeedback = 'idle';
	}

	function toggleStep(index: number) {
		if (index === 1) return;
		complete[index] = !complete[index];
	}

	function checkSubstitution() {
		const correct = problem.values.every(
			(variable) => substitutionAnswers[variable.name]?.trim() === String(variable.value)
		);
		substitutionFeedback = correct ? 'correct' : 'incorrect';
		if (correct) complete[1] = true;
	}

	function editSubstitution() {
		substitutionFeedback = 'idle';
		complete[1] = false;
	}

	function revealNext() {
		const next = complete.findIndex((done) => !done);
		if (next === -1) revealed = true;
		else if (next === 1) {
			for (const variable of problem.values) substitutionAnswers[variable.name] = String(variable.value);
			substitutionFeedback = 'correct';
			complete[next] = true;
		} else complete[next] = true;
	}
</script>

<svelte:head>
	<title>Substitution Board · Algebra Tiles</title>
	<meta
		name="description"
		content="A teacher-led substitution practice board for grade 7 math classrooms."
	/>
</svelte:head>

<main class="board">
	<header class="topbar">
		<a class="brand" href="/" aria-label="Substitution Board home">
			<span class="brand-mark" aria-hidden="true">x</span>
			<span>Substitution Board</span>
		</a>
		<div class="topbar-actions">
			<span class="call-up"><Sparkles size={17} aria-hidden="true" /> Teacher call-up</span>
			<button class="icon-button" type="button" aria-label="Show or hide teacher settings" onclick={() => (settingsOpen = !settingsOpen)}>
				<Settings2 size={21} />
			</button>
		</div>
	</header>

	{#if settingsOpen}
		<section class="settings" aria-label="Teacher settings">
			<div>
				<p class="eyebrow">Problem type</p>
				<div class="choice-row" role="group" aria-label="Choose problem type">
					<button class:chosen={level === 'one'} type="button" onclick={() => { level = 'one'; nextProblem(); }}>One variable</button>
					<button class:chosen={level === 'negative'} type="button" onclick={() => { level = 'negative'; nextProblem(); }}>Negative values</button>
					<button class:chosen={level === 'two'} type="button" onclick={() => { level = 'two'; nextProblem(); }}>Two variables</button>
				</div>
			</div>
			<div>
				<p class="eyebrow">Display</p>
				<div class="choice-row" role="group" aria-label="Choose display mode">
					<button class:chosen={mode === 'guided'} type="button" onclick={() => (mode = 'guided')}>Guided steps</button>
					<button class:chosen={mode === 'challenge'} type="button" onclick={() => (mode = 'challenge')}>Answer challenge</button>
				</div>
			</div>
		</section>
	{/if}

	<section class="hero" aria-labelledby="problem-heading">
		<div class="problem-label">
			<span>Today’s problem</span>
			<span class="number">#{String(problemNumber).padStart(2, '0')}</span>
		</div>
		<h1 id="problem-heading">Evaluate the expression</h1>
		<div class="expression" aria-label={`Expression: ${problem.expression}`}>{problem.expression}</div>
		<div class="value-cards" aria-label="Variable values">
			{#each problem.values as variable (variable.name)}
				<div class="value-card"><span>{variable.name}</span><strong>= {variable.value}</strong></div>
			{/each}
		</div>
		<p class="prompt">Say what you replace <strong>each variable</strong> with before you calculate.</p>
	</section>

	<section class="work-area" aria-labelledby="steps-heading">
		<div class="steps-heading">
			<div>
				<p class="eyebrow">Classroom checklist</p>
				<h2 id="steps-heading">Work through the steps</h2>
			</div>
			<p class="progress" aria-live="polite">{complete.filter(Boolean).length} of {problem.steps.length} checked</p>
		</div>

		<ol class="steps">
			{#each problem.steps as step, index (step)}
				<li class:done={complete[index]}>
					{#if index === 1}
						<span class:complete-mark={complete[index]} class="step-number" aria-hidden="true">
							{#if complete[index]}<Check size={22} strokeWidth={3} />{:else}{index + 1}{/if}
						</span>
					{:else}
						<button
							type="button"
							class="check"
							aria-label={`${complete[index] ? 'Uncheck' : 'Check'} step ${index + 1}: ${step}`}
							aria-pressed={complete[index]}
							onclick={() => toggleStep(index)}
						>
							{#if complete[index]}<Check size={22} strokeWidth={3} />{:else}<span>{index + 1}</span>{/if}
						</button>
					{/if}
					<div class="step-copy">
						<strong>{step}</strong>
						{#if index === 1 && mode === 'guided'}
							<div class="substitution-area">
								<div class="substitution-equation" aria-label="Enter the values to substitute">
									{#each problem.terms as term, termIndex (term.variable)}
										{#if termIndex > 0}<span>+</span>{/if}
										<span>{term.coefficient}(</span>
										<input
											bind:value={substitutionAnswers[term.variable]}
											class:correct={substitutionFeedback === 'correct'}
											class:incorrect={substitutionFeedback === 'incorrect'}
											aria-label={`Value substituted for ${term.variable}`}
											inputmode="numeric"
											oninput={editSubstitution}
											onkeydown={(event) => event.key === 'Enter' && checkSubstitution()}
										/>
										<span>)</span>
									{/each}
									<span>{signed(problem.constant)}</span>
								</div>
								<div class="substitution-actions">
									<button type="button" class="check-substitution" onclick={checkSubstitution}>Check substitution</button>
									{#if substitutionFeedback === 'incorrect'}<span class="feedback incorrect-text">Try those value cards again.</span>{/if}
									{#if substitutionFeedback === 'correct'}<span class="feedback correct-text">Correct substitution!</span>{/if}
								</div>
							</div>
						{:else if mode === 'guided' && (index === 2 || index === 3 || index === 4) && (revealed || complete[index])}
							<span class="working">
								{index === 2 ? problem.combine : index === 3 ? `${problem.answer}` : `The value is ${problem.answer}.`}
							</span>
						{/if}
					</div>
				</li>
			{/each}
		</ol>

		{#if mode === 'challenge' && revealed}
			<div class="answer-reveal" aria-live="polite"><span>Answer</span><strong>{problem.answer}</strong></div>
		{:else if mode === 'guided' && revealed}
			<div class="answer-reveal" aria-live="polite"><span>Final value</span><strong>{problem.answer}</strong></div>
		{/if}
	</section>

	<footer class="controls">
		<button class="secondary-control" type="button" onclick={resetBoard}><RotateCcw size={19} /> Reset checks</button>
		<div class="primary-controls">
			<button class="reveal" type="button" onclick={revealNext}>
				{#if revealed}<EyeOff size={19} />{:else}<Eye size={19} />{/if}
				{revealed ? 'Answer shown' : complete.every(Boolean) ? 'Show answer' : 'Check next step'}
			</button>
			<button class="next" type="button" onclick={nextProblem}>New problem <RefreshCw size={19} /></button>
		</div>
	</footer>
</main>

<style>
	:global(body) { background: #edf2f6; }
	.board { min-height: 100dvh; color: #172435; }
	.topbar { display: flex; align-items: center; justify-content: space-between; min-height: 4.75rem; padding: 0 4vw; border-bottom: 1px solid #d6e0e8; background: rgb(255 255 255 / 0.9); }
	.brand { display: inline-flex; align-items: center; gap: .65rem; color: #172435; font-size: 1.1rem; font-weight: 800; text-decoration: none; }
	.brand-mark { display: grid; place-items: center; width: 2rem; height: 2rem; border-radius: .55rem; background: #2764c5; color: white; font-family: Georgia, serif; font-size: 1.45rem; font-style: italic; font-weight: 700; }
	.topbar-actions, .call-up, .primary-controls, .settings, .choice-row { display: flex; align-items: center; }
	.topbar-actions { gap: 1rem; }
	.call-up { gap: .4rem; color: #506477; font-size: .9rem; font-weight: 700; }
	.icon-button { display: grid; place-items: center; width: 2.5rem; height: 2.5rem; border: 1px solid #c9d5df; border-radius: .55rem; background: white; color: #324c67; cursor: pointer; }
	.icon-button:hover { background: #f2f6fa; }
	.settings { justify-content: center; gap: 3rem; padding: 1rem 4vw; border-bottom: 1px solid #d6e0e8; background: #f9fbfd; }
	.eyebrow { margin: 0 0 .45rem; color: #61778b; font-size: .72rem; font-weight: 800; letter-spacing: .1em; text-transform: uppercase; }
	.choice-row { gap: .35rem; flex-wrap: wrap; }
	.choice-row button { padding: .48rem .72rem; border: 1px solid #cad7e2; border-radius: .45rem; background: white; color: #425a70; font: inherit; font-size: .86rem; font-weight: 700; cursor: pointer; }
	.choice-row button.chosen { border-color: #2764c5; background: #e6effe; color: #174b9f; }
	.hero { display: grid; justify-items: center; padding: clamp(2.5rem, 6vw, 5rem) 1.25rem clamp(2rem, 4vw, 3.5rem); background: radial-gradient(circle at center, #f8fbff 0%, #eaf1f7 72%); text-align: center; }
	.problem-label { display: flex; gap: .65rem; align-items: center; color: #55708a; font-size: .82rem; font-weight: 800; letter-spacing: .09em; text-transform: uppercase; }
	.number { padding: .2rem .46rem; border-radius: 99px; background: #dce8f5; color: #315979; font-variant-numeric: tabular-nums; }
	h1 { margin: .7rem 0 1.7rem; font-size: clamp(1.55rem, 3.2vw, 2.3rem); line-height: 1.1; }
	.expression { padding: .55rem 1.15rem; color: #173862; font-family: Georgia, 'Times New Roman', serif; font-size: clamp(3.5rem, 10vw, 7rem); font-weight: 600; letter-spacing: .02em; line-height: 1.1; }
	.value-cards { display: flex; flex-wrap: wrap; justify-content: center; gap: .8rem; margin-top: 1.75rem; }
	.value-card { min-width: 8.2rem; padding: .7rem 1.1rem; border: 1px solid #bdd0e3; border-radius: .7rem; background: white; box-shadow: 0 2px 5px rgb(40 74 110 / .06); color: #35536f; font-size: 1.1rem; }
	.value-card span { margin-right: .35rem; font-family: Georgia, serif; font-size: 1.35rem; font-style: italic; }
	.value-card strong { color: #173862; font-size: 1.35rem; }
	.prompt { margin: 1.4rem 0 0; color: #526a80; font-size: 1.03rem; }
	.work-area { width: min(100% - 2.5rem, 56rem); margin: 0 auto; padding: 2.5rem 0 8.5rem; }
	.steps-heading { display: flex; justify-content: space-between; align-items: end; gap: 1rem; margin-bottom: 1rem; }
	h2 { margin: 0; font-size: clamp(1.35rem, 2.6vw, 1.85rem); }
	.progress { margin: 0; color: #557087; font-size: .9rem; font-weight: 700; white-space: nowrap; }
	.steps { display: grid; gap: .7rem; padding: 0; margin: 0; list-style: none; }
	.steps li { display: flex; align-items: flex-start; gap: 1rem; min-height: 5.15rem; padding: 1rem 1.15rem; border: 1px solid #d3dfe8; border-radius: .8rem; background: white; box-shadow: 0 2px 4px rgb(28 57 89 / .035); transition: border-color .15s, background .15s; }
	.steps li.done { border-color: #abd4bd; background: #f2faf5; }
	.check, .step-number { display: grid; flex: none; place-items: center; width: 2.2rem; height: 2.2rem; padding: 0; border: 2px solid #a9bbca; border-radius: 50%; background: white; color: #4f667c; font: inherit; font-weight: 800; }
	.check { cursor: pointer; }
	.done .check, .complete-mark { border-color: #318456; background: #318456; color: white; }
	.step-copy { display: grid; gap: .3rem; padding-top: .24rem; }
	.step-copy strong { font-size: clamp(1rem, 2vw, 1.16rem); }
	.working { color: #305b89; font-family: Georgia, 'Times New Roman', serif; font-size: clamp(1.2rem, 3vw, 1.65rem); font-weight: 600; }
	.substitution-area { display: grid; gap: .6rem; margin-top: .25rem; }
	.substitution-equation { display: flex; flex-wrap: wrap; align-items: center; gap: .35rem; color: #204e7d; font-family: Georgia, 'Times New Roman', serif; font-size: clamp(1.35rem, 3vw, 1.75rem); font-weight: 600; }
	.substitution-equation input { width: 3.7rem; height: 2.3rem; border: 2px solid #9eb6cc; border-radius: .35rem; background: #fff; color: #173862; font: inherit; font-size: 1.25rem; font-weight: 700; text-align: center; outline-offset: 2px; }
	.substitution-equation input:focus { border-color: #2764c5; outline: 2px solid #cbdcff; }
	.substitution-equation input.correct { border-color: #318456; background: #f1faf4; }
	.substitution-equation input.incorrect { border-color: #c44c3a; background: #fff4f1; }
	.substitution-actions { display: flex; flex-wrap: wrap; align-items: center; gap: .7rem; }
	.check-substitution { min-height: 2.25rem; padding: 0 .8rem; border: 1px solid #2764c5; border-radius: .45rem; background: #2764c5; color: white; font: inherit; font-size: .9rem; font-weight: 800; cursor: pointer; }
	.check-substitution:hover { background: #1e549f; }
	.feedback { font-size: .9rem; font-weight: 750; }
	.incorrect-text { color: #a23a2b; }
	.correct-text { color: #287044; }
	.answer-reveal { display: flex; align-items: baseline; justify-content: center; gap: 1rem; margin-top: 1rem; padding: 1.2rem; border: 2px solid #75b28e; border-radius: .8rem; background: #eaf7ef; color: #276542; }
	.answer-reveal span { font-weight: 800; text-transform: uppercase; letter-spacing: .08em; font-size: .78rem; }
	.answer-reveal strong { font-family: Georgia, serif; font-size: 2.2rem; }
	.controls { position: fixed; right: 0; bottom: 0; left: 0; display: flex; align-items: center; justify-content: space-between; gap: 1rem; padding: .85rem 4vw calc(.85rem + env(safe-area-inset-bottom)); border-top: 1px solid #cfdae4; background: rgb(255 255 255 / .96); box-shadow: 0 -4px 18px rgb(27 57 90 / .08); backdrop-filter: blur(10px); }
	.controls button { display: inline-flex; align-items: center; justify-content: center; gap: .45rem; min-height: 2.75rem; border-radius: .55rem; font: inherit; font-weight: 800; cursor: pointer; }
	.secondary-control { padding: 0 .9rem; border: 1px solid #c7d4df; background: white; color: #455f76; }
	.primary-controls { gap: .65rem; }
	.reveal { padding: 0 1rem; border: 1px solid #b7c9dc; background: #e9f1fa; color: #24527f; }
	.next { padding: 0 1.1rem; border: 1px solid #1c559f; background: #2764c5; color: white; }
	.next:hover { background: #1e549f; }
	@media (max-width: 650px) { .call-up { display: none; } .settings { align-items: stretch; flex-direction: column; gap: 1rem; } .expression { font-size: clamp(3rem, 17vw, 5.5rem); } .controls { align-items: stretch; flex-direction: column-reverse; } .primary-controls { width: 100%; } .primary-controls button { flex: 1; } .secondary-control { width: 100%; } .work-area { padding-bottom: 11rem; } }
	@media (prefers-reduced-motion: reduce) { .steps li { transition: none; } }
</style>
