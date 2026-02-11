<script lang="ts">
	import { page } from '$app/state';
	import { onMount } from 'svelte';

	const webviewUrl = page.url.searchParams.get('url');

	let success = $state(false);
	let error = $state<string | null>(null);

	onMount(() => {
		if (!webviewUrl) return;
		fetch('/api/verify', {
			method: 'POST',
			headers: {
				'content-type': 'application/json'
			},
			body: JSON.stringify({ type: 'webview', identifier: webviewUrl })
		})
			.then(async (r) => {
				if (!r.ok) {
					error = await r.text().catch(() => 'unexpected error');
					return;
				}

				success = true;
			})
			.catch((e) => {
				console.error('error sending verify request', e);
				error = 'unexpected error, please check your browser console.';
			});
	});
</script>

<div class="flex min-h-screen w-screen items-center justify-center text-3xl">
	{#if webviewUrl}
		{#if success}
			<p class="text-green-500">your account is now age verified. enjoy</p>
		{:else if error}
			<p class="text-red-500">{error}</p>
		{:else}
			<p>automatically verifying your age</p>
		{/if}
	{:else}
		<p>invalid url</p>
	{/if}
</div>
