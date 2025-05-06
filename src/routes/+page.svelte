<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	import { CohereClientV2 } from 'cohere-ai';

	// App state
	const messages = writable<Array<{ role: 'user' | 'assistant', content: string }>>([]);
	let inputMessage = '';
	let apiKey = '';
	let isLoading = false;
	let showApiInput = true;
	let cohereClient: any = null;
	let lastQueryType = '';
	let isGeneratingNextSet = false;
	
	// Quiz state
	type Question = {
		text: string;
		options: string[];
		correctAnswer: number;
		userAnswer?: number;
	};
	let questions: Question[] = [];
	let currentQuestionIndex = 0;
	let showFeedback = false;
	let isCorrect = false;
	
	// Game mechanics
	let aura = 0;
	let questionsAnswered = 0;
	let isBlitzMode = false;
	let blitzTimeRemaining = 30;
	let blitzQuestionsAnswered = 0;
	let blitzInterval: any = null;
	
	// Rank thresholds and titles
	const ranks = [
		{ threshold: -1000, title: "Flunking Student", icon: "🥺" },
		{ threshold: 0, title: "Beta Calculator", icon: "🧮" },
		{ threshold: 500, title: "Derivative Dabbler", icon: "📊" },
		{ threshold: 1000, title: "Integral Intern", icon: "📈" },
		{ threshold: 2000, title: "Function Finder", icon: "🔍" },
		{ threshold: 3500, title: "Series Savant", icon: "💫" },
		{ threshold: 5000, title: "Limit Lord", icon: "🚀" },
		{ threshold: 7500, title: "Calculus King", icon: "👑" },
		{ threshold: 10000, title: "Analytical Alpha", icon: "💪" },
		{ threshold: 15000, title: "Mathematical Mastermind", icon: "🧠" },
		{ threshold: 25000, title: "Calculus Gigachad", icon: "💎" }
	];
	
	function getCurrentRank() {
		// Find the highest rank whose threshold the user has reached
		for (let i = ranks.length - 1; i >= 0; i--) {
			if (aura >= ranks[i].threshold) {
				return ranks[i];
			}
		}
		return ranks[0]; // Default to lowest rank
	}
	
	function startBlitzMode() {
		isBlitzMode = true;
		blitzTimeRemaining = 30;
		blitzQuestionsAnswered = 0;
		
		// Create interval to count down time
		blitzInterval = setInterval(() => {
			blitzTimeRemaining--;
			if (blitzTimeRemaining <= 0) {
				endBlitzMode();
			}
		}, 1000);
	}
	
	function endBlitzMode() {
		isBlitzMode = false;
		if (blitzInterval) {
			clearInterval(blitzInterval);
			blitzInterval = null;
		}
		showFeedback = false;
	}
	
	// Clean up interval on component unmount
	onMount(() => {
		return () => {
			if (blitzInterval) {
				clearInterval(blitzInterval);
			}
		};
	});

	// Handle form submission
	async function handleSubmit() {
		if (!inputMessage.trim()) return;
		
		// Store the query type for later use when generating more questions
		lastQueryType = inputMessage;
		
		// Add user message to chat
		$messages = [...$messages, { role: 'user', content: inputMessage }];
		isLoading = true;
		
		try {
			// Create new client if not already created
			if (!cohereClient && apiKey) {
				cohereClient = new CohereClientV2({
					token: apiKey
				});
			}

			// Customize the prompt for Calculus BC questions
			const systemPrompt = "You are a helpful AP Calculus BC exam creator. Generate multiple choice questions with 4 options (A, B, C, D) for each question. For each question, clearly indicate the correct answer. Format your response as follows:\nQ1: [Question text]\nA: [Option A]\nB: [Option B]\nC: [Option C]\nD: [Option D]\nCorrect: [Letter of correct answer]\n\nGenerate 3 questions on different Calculus BC topics.";
			
			// Make request to Cohere API using client
			const response = await cohereClient.chat({
				model: 'command-a-03-2025',
				message: inputMessage,
				preamble: systemPrompt
			});

			let responseText = '';
			if (response && response.text) {
				responseText = response.text;
			} else if (response && response.generation) {
				responseText = response.generation.text;
			} else {
				responseText = 'No response received from API.';
			}
			
			// Add assistant response to chat
			$messages = [...$messages, { role: 'assistant', content: responseText }];
			
			// Parse questions from the response
			parseQuestions(responseText, true);
			
		} catch (error: any) {
			console.error('Error:', error);
			$messages = [...$messages, { role: 'assistant', content: `Error: ${error?.message || 'Unknown error occurred'}` }];
		} finally {
			isLoading = false;
			inputMessage = '';
		}
	}

	async function generateNextQuestionSet() {
		if (isGeneratingNextSet || !lastQueryType) return;
		
		isGeneratingNextSet = true;
		
		try {
			// Customize the prompt to generate different questions
			const systemPrompt = "You are a helpful AP Calculus BC exam creator. Generate multiple choice questions with 4 options (A, B, C, D) for each question. For each question, clearly indicate the correct answer. Format your response as follows:\nQ1: [Question text]\nA: [Option A]\nB: [Option B]\nC: [Option C]\nD: [Option D]\nCorrect: [Letter of correct answer]\n\nGenerate 3 NEW and DIFFERENT questions on Calculus BC topics. Make sure these questions are different from any previously generated ones.";
			
			// Make request to Cohere API using client
			const response = await cohereClient.chat({
				model: 'command-a-03-2025',
				message: lastQueryType + " Please provide different questions than before.",
				preamble: systemPrompt
			});

			let responseText = '';
			if (response && response.text) {
				responseText = response.text;
			} else if (response && response.generation) {
				responseText = response.generation.text;
			} else {
				responseText = 'No response received from API.';
			}
			
			// Parse questions from the response and append them to existing questions
			parseQuestions(responseText, false);
			
		} catch (error: any) {
			console.error('Error generating next set:', error);
		} finally {
			isGeneratingNextSet = false;
		}
	}

	function parseQuestions(text: string, resetQuestions: boolean) {
		// Reset questions if needed
		if (resetQuestions) {
			questions = [];
			currentQuestionIndex = 0;
		}
		
		// Store the current question count for appending new questions
		const startingIndex = questions.length;
		
		// Simple regex pattern to extract questions
		const questionBlocks = text.split(/Q\d+:|Question \d+:/);
		
		questionBlocks.forEach((block, index) => {
			if (index === 0) return; // Skip first split which is empty or intro text
			
			const lines = block.trim().split('\n');
			if (lines.length < 6) return; // Need question, 4 options, and correct answer
			
			const questionText = lines[0].trim();
			const options = [
				lines.find(l => l.startsWith('A:'))?.substring(2).trim() || '',
				lines.find(l => l.startsWith('B:'))?.substring(2).trim() || '',
				lines.find(l => l.startsWith('C:'))?.substring(2).trim() || '',
				lines.find(l => l.startsWith('D:'))?.substring(2).trim() || ''
			];
			
			const correctLine = lines.find(l => l.toLowerCase().includes('correct'));
			let correctAnswer = 0;
			if (correctLine) {
				if (correctLine.includes('A')) correctAnswer = 0;
				else if (correctLine.includes('B')) correctAnswer = 1;
				else if (correctLine.includes('C')) correctAnswer = 2;
				else if (correctLine.includes('D')) correctAnswer = 3;
			}
			
			questions = [...questions, {
				text: questionText,
				options,
				correctAnswer
			}];
		});
	}

	function selectAnswer(optionIndex: number) {
		if (currentQuestionIndex < questions.length) {
			questions[currentQuestionIndex].userAnswer = optionIndex;
			isCorrect = optionIndex === questions[currentQuestionIndex].correctAnswer;
			showFeedback = true;
			
			// Track questions answered
			questionsAnswered++;
			
			// Calculate aura based on correctness and game mode
			if (isBlitzMode) {
				const multiplier = Math.pow(2, blitzQuestionsAnswered);
				if (isCorrect) {
					aura += 500 * multiplier;
				} else {
					aura -= 1000 * multiplier;
				}
				blitzQuestionsAnswered++;
			} else {
				if (isCorrect) {
					aura += 100;
				} else {
					aura -= 200;
				}
			}
			
			// If this is the first question in a set (assuming sets of 3), generate the next set
			if (currentQuestionIndex % 3 === 0 && !isGeneratingNextSet) {
				generateNextQuestionSet();
			}
			
			// Maybe start blitz mode? Check roughly every 6 questions
			if (!isBlitzMode && questionsAnswered % 6 === 0 && Math.random() < 0.7) {
				startBlitzMode();
			}
		}
	}

	function nextQuestion() {
		if (currentQuestionIndex < questions.length - 1) {
			currentQuestionIndex++;
			showFeedback = false;
		}
	}

	function previousQuestion() {
		if (currentQuestionIndex > 0) {
			currentQuestionIndex--;
			showFeedback = false;
		}
	}

	function saveApiKey() {
		if (apiKey.trim()) {
			showApiInput = false;
			// Initialize Cohere client when API key is saved
			cohereClient = new CohereClientV2({
				token: apiKey
			});
		}
	}
</script>

<main>
	<h1>AP Calculus BC Quiz Generator</h1>
	
	{#if showApiInput}
		<div class="api-key-container">
			<h2>Enter your Cohere API Key</h2>
			<p>Your API key will be stored in the browser session only and never sent to our servers.</p>
			<div class="input-group">
				<input type="password" bind:value={apiKey} placeholder="Enter your Cohere API key..." />
				<button on:click={saveApiKey}>Save</button>
			</div>
		</div>
	{:else}
		{#if questions.length === 0}
			<div class="prompt-container">
				<p>What type of Calculus BC questions would you like to practice?</p>
				<div class="examples">
					<button class="example-button" on:click={() => inputMessage = "Generate questions about integration techniques."}>Integration Techniques</button>
					<button class="example-button" on:click={() => inputMessage = "Create questions about series and convergence tests."}>Series & Convergence</button>
					<button class="example-button" on:click={() => inputMessage = "Make questions about parametric and polar equations."}>Parametric & Polar</button>
				</div>
				
				<form on:submit|preventDefault={handleSubmit} class="input-area">
					<input 
						type="text" 
						bind:value={inputMessage} 
						placeholder="Describe what type of questions you want..." 
						disabled={isLoading}
					/>
					<button type="submit" disabled={isLoading || !inputMessage.trim()}>
						Generate Quiz
					</button>
				</form>
				
				{#if isLoading}
					<div class="loading-indicator">
						Generating your quiz questions...
					</div>
				{/if}
			</div>
		{:else}
			<!-- Rank Display -->
			<div class="rank-display" class:blitz={isBlitzMode}>
				<div class="rank-title">
					<span class="rank-icon">{getCurrentRank().icon}</span>
					<span>Rank: {getCurrentRank().title}</span>
				</div>
				<div class="aura-meter">
					<div class="aura-label">AURA: {aura}</div>
					<div class="aura-progress-container">
						<div 
							class="aura-progress" 
							style="width: {Math.min(100, Math.max(0, (aura - getCurrentRank().threshold) / 
								(ranks[ranks.findIndex(r => r.title === getCurrentRank().title) + 1]?.threshold - getCurrentRank().threshold || 1000) * 100))}%"
						></div>
					</div>
				</div>
				
				<!-- Blitz Mode Indicator -->
				{#if isBlitzMode}
					<div class="blitz-indicator">
						<div class="blitz-timer">⏱️ {blitzTimeRemaining}s</div>
						<div class="blitz-label">BLITZ MODE! 🔥</div>
						<div class="blitz-multiplier">x{Math.pow(2, blitzQuestionsAnswered)}</div>
					</div>
				{/if}
			</div>
			
			<div class="quiz-container" class:blitz={isBlitzMode}>
				<div class="question-navigation">
					<span>Question {currentQuestionIndex + 1} of {questions.length}</span>
					<div>
						<button on:click={previousQuestion} disabled={currentQuestionIndex === 0 || isBlitzMode}>Previous</button>
						<button on:click={nextQuestion} disabled={currentQuestionIndex === questions.length - 1 || isBlitzMode}>Next</button>
					</div>
				</div>
				
				<div class="question-card">
					<h3>{questions[currentQuestionIndex].text}</h3>
					
					<div class="options">
						{#each questions[currentQuestionIndex].options as option, index}
							<button 
								class="option-button"
								class:selected={questions[currentQuestionIndex].userAnswer === index}
								class:correct={showFeedback && questions[currentQuestionIndex].correctAnswer === index}
								class:incorrect={showFeedback && questions[currentQuestionIndex].userAnswer === index && questions[currentQuestionIndex].correctAnswer !== index}
								class:blitz={isBlitzMode}
								disabled={showFeedback}
								on:click={() => selectAnswer(index)}
							>
								{String.fromCharCode(65 + index)}. {option}
							</button>
						{/each}
					</div>
					
					{#if showFeedback}
						<div class="feedback {isCorrect ? 'correct-feedback' : 'incorrect-feedback'} {isBlitzMode ? 'blitz' : ''}">
							{#if isCorrect}
								<p>Correct! {isBlitzMode ? `+${500 * Math.pow(2, blitzQuestionsAnswered-1)} Aura` : '+100 Aura'}</p>
							{:else}
								<p>Incorrect. The correct answer is {String.fromCharCode(65 + questions[currentQuestionIndex].correctAnswer)}. 
									{isBlitzMode ? `-${1000 * Math.pow(2, blitzQuestionsAnswered-1)} Aura` : '-200 Aura'}</p>
							{/if}
						</div>
						
						{#if !isBlitzMode}
							{#if currentQuestionIndex < questions.length - 1}
								<button class="next-button" on:click={nextQuestion}>Next Question</button>
							{:else}
								<button class="restart-button" on:click={() => {questions = []; showFeedback = false;}}>Start New Quiz</button>
							{/if}
						{:else}
							<button class="next-button blitz" on:click={() => {showFeedback = false; nextQuestion();}}>Continue Blitz!</button>
						{/if}
					{/if}
					
					{#if isGeneratingNextSet && currentQuestionIndex % 3 === 0}
						<div class="next-set-indicator">
							Preparing more questions in the background...
						</div>
					{/if}
				</div>
			</div>
		{/if}
	{/if}
</main>

<style>
	main {
		max-width: 800px;
		margin: 0 auto;
		padding: 2rem;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
			Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
	}
	
	h1 {
		color: #333;
		text-align: center;
		margin-bottom: 2rem;
	}
	
	.api-key-container {
		background: #f9f9f9;
		padding: 2rem;
		border-radius: 8px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
	}
	
	.input-group {
		display: flex;
		margin-top: 1rem;
	}
	
	.prompt-container {
		background: #fff;
		padding: 2rem;
		border-radius: 8px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
		text-align: center;
	}
	
	.examples {
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		gap: 1rem;
		margin: 1.5rem 0;
	}
	
	.example-button {
		padding: 0.75rem 1rem;
		background: #e3f2fd;
		color: #0039cb;
		border: 1px solid #bbdefb;
		border-radius: 4px;
		cursor: pointer;
		transition: all 0.2s;
	}
	
	.example-button:hover {
		background: #bbdefb;
	}
	
	/* Rank Display */
	.rank-display {
		background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
		border-radius: 8px;
		padding: 1rem;
		margin-bottom: 1.5rem;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
		transition: all 0.3s ease;
	}
	
	.rank-display.blitz {
		background: linear-gradient(135deg, #ff9a9e 0%, #fad0c4 99%, #fad0c4 100%);
		animation: pulse 1.5s infinite;
	}
	
	.rank-title {
		display: flex;
		align-items: center;
		font-size: 1.5rem;
		font-weight: bold;
		margin-bottom: 0.5rem;
	}
	
	.rank-icon {
		font-size: 2rem;
		margin-right: 0.5rem;
	}
	
	.aura-meter {
		margin-top: 0.5rem;
	}
	
	.aura-label {
		font-weight: bold;
		margin-bottom: 0.25rem;
	}
	
	.aura-progress-container {
		height: 0.75rem;
		background: rgba(255, 255, 255, 0.5);
		border-radius: 1rem;
		overflow: hidden;
	}
	
	.aura-progress {
		height: 100%;
		background: linear-gradient(90deg, #4facfe 0%, #00f2fe 100%);
		border-radius: 1rem;
		transition: width 0.5s ease;
	}
	
	/* Blitz Mode */
	.blitz-indicator {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-top: 0.75rem;
		padding: 0.5rem;
		background: rgba(255, 255, 255, 0.3);
		border-radius: 4px;
		font-weight: bold;
	}
	
	.blitz-timer {
		font-size: 1.2rem;
	}
	
	.blitz-label {
		font-size: 1.25rem;
		color: #e91e63;
		text-shadow: 0 0 5px rgba(255, 255, 255, 0.7);
		animation: blink 1s infinite;
	}
	
	.blitz-multiplier {
		font-size: 1.2rem;
		color: #9c27b0;
	}
	
	.quiz-container {
		background: #fff;
		border-radius: 8px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
		padding: 2rem;
		transition: all 0.3s ease;
	}
	
	.quiz-container.blitz {
		background: #fff9f9;
		box-shadow: 0 0 15px rgba(255, 0, 0, 0.2);
	}
	
	.question-navigation {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1.5rem;
		padding-bottom: 1rem;
		border-bottom: 1px solid #eee;
	}
	
	.question-card {
		padding: 1rem;
	}
	
	.options {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		margin: 1.5rem 0;
	}
	
	.option-button {
		padding: 1rem;
		text-align: left;
		background: #f9f9f9;
		border: 1px solid #ddd;
		border-radius: 8px;
		cursor: pointer;
		transition: all 0.2s;
	}
	
	.option-button:hover:not(:disabled) {
		background: #f0f0f0;
		border-color: #ccc;
	}
	
	.option-button.blitz {
		animation: highlight 2s infinite;
		transition: all 0.1s ease;
	}
	
	.option-button.blitz:hover:not(:disabled) {
		transform: scale(1.02);
	}
	
	.option-button.selected {
		background: #e3f2fd;
		border-color: #2962ff;
	}
	
	.option-button.correct {
		background: #e8f5e9;
		border-color: #4caf50;
	}
	
	.option-button.incorrect {
		background: #ffebee;
		border-color: #f44336;
	}
	
	.feedback {
		padding: 1rem;
		border-radius: 8px;
		margin: 1.5rem 0;
	}
	
	.feedback.blitz {
		font-size: 1.2rem;
		animation: appear 0.3s ease-out;
	}
	
	.correct-feedback {
		background: #e8f5e9;
		color: #2e7d32;
		border-left: 4px solid #4caf50;
	}
	
	.incorrect-feedback {
		background: #ffebee;
		color: #c62828;
		border-left: 4px solid #f44336;
	}
	
	.next-button, .restart-button {
		display: block;
		width: 100%;
		padding: 1rem;
		margin-top: 1.5rem;
		background: #2962ff;
		color: white;
		border: none;
		border-radius: 4px;
		font-size: 1rem;
		cursor: pointer;
		transition: background 0.2s;
	}
	
	.next-button.blitz {
		background: linear-gradient(45deg, #ff416c, #ff4b2b);
		animation: pulse 1.5s infinite;
		font-weight: bold;
		font-size: 1.1rem;
	}
	
	.next-button:hover, .restart-button:hover {
		background: #0039cb;
	}
	
	.next-button.blitz:hover {
		background: linear-gradient(45deg, #ff416c, #ff4b2b);
		filter: brightness(1.1);
	}
	
	.input-area {
		display: flex;
		margin-top: 2rem;
	}
	
	input {
		flex: 1;
		padding: 0.75rem 1rem;
		border: 1px solid #ddd;
		border-radius: 4px 0 0 4px;
		font-size: 1rem;
	}
	
	button {
		padding: 0.75rem 1.5rem;
		background: #2962ff;
		color: white;
		border: none;
		border-radius: 0 4px 4px 0;
		font-size: 1rem;
		cursor: pointer;
		transition: background 0.2s;
	}
	
	button:hover:not(:disabled) {
		background: #0039cb;
	}
	
	button:disabled {
		background: #ccc;
		cursor: not-allowed;
	}
	
	.loading-indicator {
		margin-top: 2rem;
		color: #666;
		font-style: italic;
	}
	
	.next-set-indicator {
		margin-top: 1rem;
		font-size: 0.9rem;
		color: #666;
		font-style: italic;
		text-align: center;
	}
	
	/* Animations */
	@keyframes pulse {
		0% {
			transform: scale(1);
		}
		50% {
			transform: scale(1.02);
		}
		100% {
			transform: scale(1);
		}
	}
	
	@keyframes blink {
		0% {
			opacity: 1;
		}
		50% {
			opacity: 0.7;
		}
		100% {
			opacity: 1;
		}
	}
	
	@keyframes highlight {
		0% {
			box-shadow: 0 0 0 rgba(255, 0, 0, 0);
		}
		50% {
			box-shadow: 0 0 7px rgba(255, 0, 0, 0.3);
		}
		100% {
			box-shadow: 0 0 0 rgba(255, 0, 0, 0);
		}
	}
	
	@keyframes appear {
		0% {
			transform: translateY(-10px);
			opacity: 0;
		}
		100% {
			transform: translateY(0);
			opacity: 1;
		}
	}
</style>
