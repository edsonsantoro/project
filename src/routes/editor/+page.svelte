<script>
	import WordProbability from './WordProbability.svelte';
	
	let jsonContent = null;
	let fileName = '';
	let videoUrl = null;
	let videoElement;
	let speakers = new Set(['SPEAKER_1', 'SPEAKER_2']);
	let currentPlayingSegment = null;

	function handleFileUpload(event) {
		const file = event.target.files[0];
		if (!file) return;

		fileName = file.name;
		const reader = new FileReader();
		reader.onload = (e) => {
			try {
				jsonContent = JSON.parse(e.target.result);
				// Extract existing speakers from segments
				jsonContent.segments.forEach(segment => {
					if (segment.speaker) {
						speakers.add(segment.speaker);
					}
					// Ensure word probabilities exist
					if (!segment.words) {
						segment.words = segment.text.split(' ').map(text => ({
							text,
							probability: Math.random(), // Fallback for testing
							start: segment.start // Add start time for each word
						}));
					}
				});
				speakers = new Set(speakers); // Trigger reactivity
			} catch (error) {
				alert('Invalid JSON file');
			}
		};
		reader.readAsText(file);
	}

	function handleVideoUpload(event) {
		const file = event.target.files[0];
		if (!file) return;
		videoUrl = URL.createObjectURL(file);
	}

	function saveChanges() {
		if (!jsonContent) return;
		
		// Update the full text based on segments
		jsonContent.text = jsonContent.segments
			.map(segment => segment.text)
			.join(' ');
		
		const blob = new Blob([JSON.stringify(jsonContent, null, 2)], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = fileName.replace('.json', '_edited.json');
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url);
	}

	function updateSegment(index, field, value) {
		if (!jsonContent?.segments) return;
		jsonContent.segments[index][field] = value;
		
		// Update word probabilities if text changes
		if (field === 'text') {
			const words = value.split(' ');
			const oldWords = jsonContent.segments[index].words || [];
			const segmentStart = jsonContent.segments[index].start;
			const segmentDuration = jsonContent.segments[index].end - segmentStart;
			
			jsonContent.segments[index].words = words.map((text, i) => {
				const oldWord = oldWords[i];
				// Calculate approximate start time for new words
				const wordStart = segmentStart + (segmentDuration * (i / words.length));
				return {
					word: text,
					probability: oldWord?.probability || Math.random(),
					start: oldWord?.start || wordStart
				};
			});
		}
		
		jsonContent = { ...jsonContent }; // Trigger reactivity
	}

	function deleteSegment(index) {
		if (!jsonContent?.segments) return;
		jsonContent.segments.splice(index, 1);
		jsonContent = { ...jsonContent };
	}

	function addSegment(index) {
		if (!jsonContent?.segments) return;
		const prevSegment = jsonContent.segments[index];
		const nextSegment = jsonContent.segments[index + 1];
		
		let start = prevSegment ? prevSegment.end : 0;
		let end = nextSegment ? nextSegment.start : (start + 2);

		const newSegment = {
			text: '',
			start,
			end,
			speaker: prevSegment?.speaker || '',
			words: []
		};

		jsonContent.segments.splice(index + 1, 0, newSegment);
		jsonContent = { ...jsonContent };
	}

	function addSpeaker() {
		const name = prompt('Enter new speaker name:');
		if (name) {
			speakers.add(name);
			speakers = new Set(speakers);
		}
	}

	function seekVideo(time) {
		if (videoElement) {
			videoElement.currentTime = time;
		}
	}

	function playWordAudio(word, segmentEnd) {
		if (!videoElement) return;
		
		// Stop any current playback
		if (currentPlayingSegment !== null) {
			videoElement.removeEventListener('timeupdate', currentPlayingSegment);
			currentPlayingSegment = null;
		}

		// Start playback from word start time
		videoElement.currentTime = word.start;
		videoElement.play();

		// Create timeupdate handler to stop at segment end
		const timeUpdateHandler = () => {
			if (videoElement.currentTime >= segmentEnd) {
				videoElement.pause();
				videoElement.removeEventListener('timeupdate', timeUpdateHandler);
				currentPlayingSegment = null;
			}
		};

		// Add timeupdate handler
		videoElement.addEventListener('timeupdate', timeUpdateHandler);
		currentPlayingSegment = timeUpdateHandler;
	}

	function updateSegmentTiming(index, field, value) {
		const time = parseFloat(value);
		if (isNaN(time)) return;

		const segments = jsonContent.segments;
		const current = segments[index];
		const prev = index > 0 ? segments[index - 1] : null;
		const next = index < segments.length - 1 ? segments[index + 1] : null;

		if (field === 'start') {
			if (prev && time < prev.end) return;
			if (time >= current.end) return;
		} else if (field === 'end') {
			if (next && time > next.start) return;
			if (time <= current.start) return;
		}

		updateSegment(index, field, time);
	}
</script>

<svelte:head>
	<title>Whisper JSON Editor</title>
</svelte:head>

<div class="editor-container">
	<h1>Whisper JSON Editor</h1>

	<div class="controls">
		<div class="file-input">
			<label for="json-file">Upload Whisper JSON file:</label>
			<input
				type="file"
				id="json-file"
				accept=".json"
				on:change={handleFileUpload}
			/>
		</div>

		<div class="file-input">
			<label for="video-file">Upload Video/Audio (optional):</label>
			<input
				type="file"
				id="video-file"
				accept="video/*,audio/*"
				on:change={handleVideoUpload}
			/>
		</div>

		{#if jsonContent}
			<button class="save-button" on:click={saveChanges}>
				Save Changes
			</button>
		{/if}
	</div>

	{#if videoUrl}
		<div class="video-container">
			<video
				bind:this={videoElement}
				src={videoUrl}
				controls
				preload="metadata"
			>
				<track kind="captions" />
			</video>
		</div>
	{/if}

	{#if jsonContent}
		<div class="json-info">
			<p><strong>Text:</strong> {jsonContent.text}</p>
			<p><strong>Language:</strong> {jsonContent.language}</p>
		</div>

		<div class="speaker-controls">
			<h3>Speakers</h3>
			<button class="add-speaker" on:click={addSpeaker}>
				Add Speaker
			</button>
		</div>

		<div class="segments">
			<h2>Segments</h2>
			<button class="add-segment" on:click={() => addSegment(-1)}>
				Add Segment at Start
			</button>
			
			{#each jsonContent.segments as segment, i}
				<div class="segment">
					<div class="segment-header">
						<div class="segment-info">
							<span>Segment {i + 1}</span>
							<div class="timing-controls">
								<input
									type="number"
									step="0.01"
									value={segment.start}
									on:input={(e) => updateSegmentTiming(i, 'start', e.target.value)}
								/>
								<span>-</span>
								<input
									type="number"
									step="0.01"
									value={segment.end}
									on:input={(e) => updateSegmentTiming(i, 'end', e.target.value)}
								/>
								<button class="time-seek" on:click={() => seekVideo(segment.start)}>
									▶
								</button>
							</div>
						</div>
						<div class="segment-controls">
							<select
								value={segment.speaker || ''}
								on:change={(e) => updateSegment(i, 'speaker', e.target.value)}
							>
								<option value="">No Speaker</option>
								{#each [...speakers] as speaker}
									<option value={speaker}>{speaker}</option>
								{/each}
							</select>
							<button class="delete-segment" on:click={() => deleteSegment(i)}>
								Delete
							</button>
						</div>
					</div>
					<textarea
						value={segment.text}
						on:input={(e) => updateSegment(i, 'text', e.target.value)}
						rows="2"
					/>
					<WordProbability 
						words={segment.words || []} 
						segmentEnd={segment.end}
						onWordClick={(word) => playWordAudio(word, segment.end)}
					/>
					<button class="add-segment" on:click={() => addSegment(i)}>
						Add Segment Below
					</button>
				</div>
			{/each}
		</div>
	{:else}
		<p class="placeholder">Upload a Whisper JSON file to start editing</p>
	{/if}
</div>

<style>
	.editor-container {
		max-width: 1000px;
		margin: 0 auto;
		padding: 2rem;
	}

	h1 {
		color: var(--color-theme-1);
		margin-bottom: 2rem;
	}

	.controls {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		margin-bottom: 2rem;
		gap: 1rem;
		flex-wrap: wrap;
	}

	.file-input {
		flex: 1;
		min-width: 250px;
	}

	.file-input label {
		display: block;
		margin-bottom: 0.5rem;
		color: var(--color-text);
	}

	input[type="file"] {
		width: 100%;
		padding: 0.5rem;
		border: 1px solid #ccc;
		border-radius: 4px;
	}

	.video-container {
		margin-bottom: 2rem;
		background: #000;
		border-radius: 4px;
		overflow: hidden;
	}

	video {
		width: 100%;
		max-height: 400px;
	}

	.save-button, .add-speaker, .add-segment {
		background-color: var(--color-theme-1);
		color: white;
		border: none;
		padding: 0.5rem 1rem;
		border-radius: 4px;
		cursor: pointer;
		transition: background-color 0.2s;
	}

	.save-button:hover, .add-speaker:hover, .add-segment:hover {
		background-color: #ff4b15;
	}

	.add-segment {
		background-color: var(--color-theme-2);
		width: 100%;
		margin: 0.5rem 0;
	}

	.add-segment:hover {
		background-color: #4f8ac7;
	}

	.json-info {
		background-color: var(--color-bg-2);
		padding: 1rem;
		border-radius: 4px;
		margin-bottom: 2rem;
	}

	.json-info p {
		margin: 0.5rem 0;
	}

	.speaker-controls {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1rem;
	}

	.speaker-controls h3 {
		margin: 0;
	}

	.segments {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.segment {
		background-color: white;
		padding: 1rem;
		border-radius: 4px;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
	}

	.segment-header {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		margin-bottom: 0.5rem;
		color: var(--color-text);
		font-size: 0.9rem;
	}

	.segment-info {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.timing-controls {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.timing-controls input {
		width: 80px;
		padding: 0.25rem;
		border: 1px solid #ccc;
		border-radius: 4px;
	}

	.time-seek {
		background: var(--color-theme-2);
		color: white;
		border: none;
		border-radius: 4px;
		padding: 0.25rem 0.5rem;
		cursor: pointer;
	}

	.segment-controls {
		display: flex;
		gap: 0.5rem;
	}

	.segment-controls select {
		padding: 0.25rem;
		border: 1px solid #ccc;
		border-radius: 4px;
		flex: 1;
	}

	.delete-segment {
		background-color: #dc3545;
		color: white;
		border: none;
		padding: 0.25rem 0.5rem;
		border-radius: 4px;
		cursor: pointer;
	}

	.delete-segment:hover {
		background-color: #c82333;
	}

	textarea {
		width: 100%;
		padding: 0.5rem;
		border: 1px solid #ccc;
		border-radius: 4px;
		font-family: inherit;
		font-size: 1rem;
		resize: vertical;
	}

	textarea:focus {
		outline: none;
		border-color: var(--color-theme-1);
	}

	.placeholder {
		text-align: center;
		color: var(--color-text);
		font-style: italic;
		padding: 2rem;
		background-color: var(--color-bg-2);
		border-radius: 4px;
	}
</style>