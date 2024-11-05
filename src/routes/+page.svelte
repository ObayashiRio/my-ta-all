<script>
	let userMessage = '';
	let problems = [];
	let chatHistory = [];
	let knowledgeBase = [];
	let currentProblemIndex = 0;
	let showHint = false;
	let isCorrect = false;

	const problemPrompt = {
		role: 'user',
		content: `
			学習のためにバグのある例を含むpythonコードをLevel1〜10の難易度ごとに1つ作成してください．
			回答は以下のJSONを埋めてください．コメントも一切不要です．
			どこがバグであるかは明示する必要はありません．
			JSON:
			{
				"problems" : [
					{
						"level": "1",
						"expected_behavior": "", # 期待する動作についての説明
						"buggy_code": "def buggy_function():\\n    return 1 + 1", #example
						"hint": "", # 行き詰まったときのヒント
						"answer": "", # バグの原因についての説明
					},
					{
						"level": "2",
						"expected_behavior": "", # 期待する動作についての説明
						"buggy_code": "def buggy_function():\\n    return 1 + 1", #example
						"hint": "", # 行き詰まったときのヒント
						"answer": "", # バグの原因についての説明
					},
					{
						"level": "3",
						"expected_behavior": "", # 期待する動作についての説明
						"buggy_code": "def buggy_function():\\n    return 1 + 1", #example
						"hint": "", # 行き詰まったときのヒント
						"answer": "", # バグの原因についての説明
					}
				]
			}
			`,
	};

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
			problems = JSON.parse(response).problems;
		});

		const initialBehaviorPrompt = {
			role: 'user',
			content: `
			以下の文章をもとに，学習者の立場から，「期待する動作が得られないため教えてほしい」という主旨の文章にしてくだい．
			${problems[currentProblemIndex].expected_behavior}

			JSON:
			{
				"message": ""
			}
			`,
		};

		await getAgentResponse([initialBehaviorPrompt]).then((response) => {
			chatHistory = [{ role: 'assistant', content: JSON.parse(response).message }];
		});
	}

	async function updateKnowledgeBase() {
		const updatePrompt = {
			role: 'user',
			content: `
			あなたは問題「${problems[currentProblemIndex].buggy_code}」に直面しています．
			ユーザーからの発話は，「${userMessage}」です．
			この発話から問題について得られた知識があれば，あなたの知識を更新してください．
			なければ，更新しないでください．

			JSON:
			{
				"knowldege": [
					${knowledgeBase.map((knowledge) => `"${knowledge}"`).join(',')}
					"",
				]
			}
			`,
		};

		await getAgentResponse([updatePrompt]).then((response) => {
			knowledgeBase = JSON.parse(response).knowledge;
			console.log(knowledgeBase);
		});
	}

	async function sendMessage() {
		chatHistory = [...chatHistory, { role: 'user', content: userMessage }];

		await updateKnowledgeBase();

		const responsePrompt = {
			role: 'system',
			content: `
			バグのあるコード（buggy_code）を今の知識（knowledge）だけを使って修正してください．
			知識が不十分な場合は何も修正しないでください．
			JSON:
			{
				"buggy_code": "${problems[currentProblemIndex].buggy_code}",
				"knowledge": [
					${knowledgeBase.map((knowledge) => `"${knowledge}"`).join(',')}
				],
				"edited_code": "" # 修正後のコード
			}
			`,
		};

		let agentResponse = '';
		await getAgentResponse([responsePrompt]).then((response) => {
			agentResponse = JSON.parse(response);
		});
		problems[currentProblemIndex].buggy_code = agentResponse.edited_code;

		const checkPrompt = {
			role: 'system',
			content: `
			修正後のコード（edited_code）が正しいか（期待する動作をしているか）どうかを判定してください．
			JSON:
			{
				"edited_code": "${problems[currentProblemIndex].buggy_code}",
				"expected_behavior": "${problems[currentProblemIndex].expected_behavior}",
				"is_correct": false
			}
			`,
		};

		await getAgentResponse([checkPrompt]).then((response) => {
			isCorrect = JSON.parse(response).is_correct;
		});

		let agentMessage = "";
		if (isCorrect) {
			agentMessage = "正しく動きました！ありがとうございます！";
		} else {
			agentMessage = "まだうまく動きません...";
			showHint = true;
		}

		chatHistory = [
			...chatHistory,
			{ role: 'assistant', content: agentMessage },
		];

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
	{#if isCorrect}
		<button on:click={() => {
			currentProblemIndex += 1;
			showHint = false;
			isCorrect = false;
			initializeAgent();
		}}>次の問題へ</button>
	{/if}

	{#if currentProblemIndex === problems.length}
		<p>全ての問題を解き終えました．</p>
	{/if}

	{#if chatHistory.length === 0}
		{#await initializeAgent()}
			<p>エージェントを初期化中...</p>
		{:catch error}
			<p>エージェントの初期化に失敗しました．</p>
			<p>エラー: {error.message}</p>
		{/await}
	{/if}
{#if problems.length > 0}
<p>Level: {problems[currentProblemIndex].level}</p>
<p>バグのあるコード:</p>
<pre><code>{problems[currentProblemIndex].buggy_code}</code></pre>
{/if}
{#if showHint}
	<p>ヒント:</p>
	<pre><code>{problems[currentProblemIndex].hint}</code></pre>
{/if}

	{#each chatHistory.slice().reverse() as { role, content }, i}
		{#if role !== 'system'}
			<div style="color: {role === 'user' ? 'white' : 'skyblue'}">
				{role === 'user' ? 'あなた' : 'エージェント'}: {content}
			</div>
		{/if}
	{/each}
</main>
