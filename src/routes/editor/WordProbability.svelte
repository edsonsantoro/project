<script>
	export let words = [];
	export let segmentEnd;
	export let onWordClick;
	
	function getColorForProbability(probability) {
		// Convert probability to a color between red (0) and green (1)
		const red = Math.round(255 * (1 - probability));
		const green = Math.round(255 * probability);
		return `rgb(${red}, ${green}, 0)`;
	}
</script>

<div class="word-probability">
	{#each words as word}
		<span 
			style="color: {getColorForProbability(word.probability)}"
			class="word"
			on:click={() => onWordClick(word)}
			class:low-probability={word.probability < 0.5}
		>
			{word.word}
		</span>
	{/each}
</div>

<style>
	.word-probability {
		padding: 0.5rem;
		background-color: #f8f9fa;
		border-radius: 4px;
		font-family: monospace;
		line-height: 1.5;
		margin-top: 0.5rem;
	}

	.word {
		margin-right: 0.25rem;
		cursor: pointer;
		padding: 2px 4px;
		border-radius: 3px;
		transition: background-color 0.2s;
	}

	.word:last-child {
		margin-right: 0;
	}

	.word:hover {
		background-color: rgba(0, 0, 0, 0.1);
	}

	.low-probability {
		text-decoration: underline dotted;
	}
</style>