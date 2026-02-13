<script lang="ts">
	import jsQR from 'jsqr';

	let qrCodeUrl = $state<string | null>(null);
	let qrCodeError = $state<string | null>(null);
	let qrCodeSuccess = $state(false);

	let dragOver = $state(false);

	async function decodeImage(file: File) {
		qrCodeError = null;
		try {
			const bitmap = await createImageBitmap(file);
			const canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
			const ctx = canvas.getContext('2d')!;
			ctx.drawImage(bitmap, 0, 0);
			const imageData = ctx.getImageData(0, 0, bitmap.width, bitmap.height);
			const result = jsQR(imageData.data, imageData.width, imageData.height);
			if (result) {
				qrCodeUrl = result.data;
			} else {
				qrCodeError = 'no qr code found in image';
			}
		} catch {
			qrCodeError = 'could not read image';
		}
	}
</script>

<div class="mt-4 flex gap-2">
	<input
		class="min-w-0 flex-1 border-2 border-white/50 p-2 {dragOver ? 'border-dashed' : ''}"
		bind:value={qrCodeUrl}
		placeholder="https://... (or drop a qr code image)"
		ondrop={(e) => {
			e.preventDefault();
			dragOver = false;
			const file = e.dataTransfer?.files[0];
			if (file) decodeImage(file);
		}}
		ondragover={(e) => {
			e.preventDefault();
			dragOver = true;
		}}
		ondragleave={() => {
			dragOver = false;
		}}
	/>
	<label class="w-16 border-2 border-white/50 p-2 text-center hover:cursor-pointer">
		scan
		<input
			type="file"
			accept="image/*"
			class="hidden"
			onchange={(e) => {
				const file = e.currentTarget.files?.[0];
				if (file) decodeImage(file);
				e.currentTarget.value = '';
			}}
		/>
	</label>
	<button
		class="w-24 border-2 border-white/50 p-2 hover:cursor-pointer"
		onclick={(e) => {
			e.preventDefault();
			qrCodeError = null;
			qrCodeSuccess = false;

			if (!qrCodeUrl) {
				qrCodeError = "you didn't enter a qr code url";
				return;
			}

			fetch('/api/verify', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					type: 'qr_link',
					identifier: qrCodeUrl
				})
			})
				.then(async (r) => {
					if (!r.ok) {
						qrCodeError = await r.text().catch(() => 'unexpected error');
						return;
					}

					qrCodeSuccess = true;
				})
				.catch((e) => {
					console.error('error sending verify request', e);
					qrCodeError = 'unexpected error, please check your browser console.';
				});
		}}>verify</button
	>
</div>
{#if qrCodeSuccess}
	<p class="mt-4 text-green-500">
		your account has successfully been verified. go back to the site tab to continue
	</p>
{:else if qrCodeError}
	<p class="mt-4 text-red-500">
		{qrCodeError}
	</p>
{/if}
