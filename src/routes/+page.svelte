<script>
	let userMessage = "";
	let agentResponse = "";

	const problemPrompt = {
		role: 'user',
		content: `
			学習のためにバグのある例を含むpythonコードを1つ作成してください．
			回答はコードのみにしてください．コメントアウトも一切不要です．
			どこがバグであるかは明示する必要はありません．
			(コードは適切に<code>タグで囲み，改行もしてください。)
			`,
	};

	let chatHistory = [];
	let knowledgeBase = "";

	async function getAgentResponse(messages) {
		const response = await fetch('/api/agent', {
			method: 'POST',
			body: JSON.stringify(messages),
			headers: {
				'content-type': 'application/json',
			},
		});

		return await response.json();
	}

	async function initializeAgent() {
		await getAgentResponse([problemPrompt]).then((response) => {
			const systemPrompt = {
				role: 'system',
				content:
					`あなたはpythonプログラムを学ぶ初学者です．
					あなたは以下の状態で行き詰まっています．解決策は何も思いつきません．
					状態: ${response}

					ユーザーからの教えを請うために，あなたから会話を開始し，現状を伝えてください．
					`,
			};
			chatHistory = [systemPrompt];
		});

		await getAgentResponse(chatHistory).then((response) => {
			chatHistory = [...chatHistory, { role: 'assistant', content: response }];
		});
	}

	async function updateKnowledgeBase() {
		const updatePrompt = {
			role: 'system',
			content: `
			これまでのユーザーからの教えを踏まえて，今の知識「${knowledgeBase}」を更新して返答してください．
			`,
		};
		const contexts = [...chatHistory, updatePrompt];

		await getAgentResponse(contexts).then((response) => {
			knowledgeBase = response;
			console.log(knowledgeBase);
		});
	}

	async function sendMessage() {
		chatHistory = [...chatHistory, { role: 'user', content: userMessage }];

		await updateKnowledgeBase();

		const responsePrompt = {
			role: 'system',
			content: `
			これまでのユーザーからの教えを踏まえて，今の知識「${knowledgeBase}」だけを使って返答してください．
			`,
		};
		const contexts = [...chatHistory, responsePrompt];
		agentResponse = await getAgentResponse(contexts);

		chatHistory = [
			...chatHistory,
			{ role: 'assistant', content: agentResponse },
		];
		console.log(chatHistory);

		userMessage = '';
	}
</script>

<main>
	<h1>My Teachable Agent</h1>
	<textarea
		rows="4"
		cols="50"
		bind:value={userMessage}
		placeholder="メッセージを入力してください"></textarea>
	<br />
	<button on:click={() => sendMessage()}>送信</button>

	{#if chatHistory.length === 0}
		{#await initializeAgent()}
			<p>エージェントを初期化中...</p>
		{/await}
	{/if}

	{#each chatHistory.slice().reverse() as { role, content }, i}
		{#if role !== 'system'}
			<div style="color: {role === 'user' ? 'white' : 'skyblue'}">
				{role === 'user' ? 'あなた' : 'エージェント'}: {@html content}
			</div>
		{/if}
	{/each}
</main>
