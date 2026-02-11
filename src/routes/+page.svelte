<svelte:head>
	<title>discord age verifier</title>
</svelte:head>

<div class="mx-auto w-screen max-w-6xl items-center p-5 pb-16">
	<h1 class="mt-16 text-center text-3xl font-extrabold">discord age verifier</h1>
	<p class="text-center">age verifies your discord account automatically as an adult</p>
	<p class="mt-4 text-center text-white/50">
		made by <a class="underline" href="https://eva.ac" target="_blank">xyzeva</a> and
		<a class="underline" href="https://github.com/Dziurwa14" target="_blank">Dziurwa</a>, greetz to
		<a class="underline" href="https://amplitudes.me/" target="_blank">amplitudes</a> (for previous work)
	</p>

	<p class="mt-8 text-left">
		it <span class="font-bold">doesn't matter</span> if you are in the UK or similar region that
		currently has access to this, this will verify your account for the future global rollout in
		march aswell as current. to use, simply paste this script into your discord console by going to
		<a
			class="font-bold underline"
			href="https://discord.com/app"
			target="_blank"
			rel="noreferer noopener">discord.com/app</a
		>, pressing <span class="font-bold">F12</span>, going to <span class="font-bold">Console</span>
		and copying and pasting and hitting enter on the following script and solving the captcha that pops
		up
		<span class="text-white/70">(typing "allow pasting" before if necessary)</span>:
	</p>
	<pre class="my-4 bg-white/10 p-5 text-wrap">// add a chunk to get all of the webpack chunks
_mods = webpackChunkdiscord_app.push([[Symbol()],&#123;&#125;,r=>r.c]);
webpackChunkdiscord_app.pop(); // cleanup the chunk we added

// utitility to find a webpack chunk by property
findByProps = (...props) => &#123;
    for (let m of Object.values(_mods)) &#123;
        try &#123;
            if (!m.exports || m.exports === window) continue;
            if (props.every((x) => m.exports?.[x])) return m.exports;

            for (let ex in m.exports) &#123;
                if (props.every((x) => m.exports?.[ex]?.[x]) && m.exports[ex][Symbol.toStringTag] !== 'IntlMessagesProxy') return m.exports[ex];
            &#125;
        &#125; catch &#123;&#125;
    &#125;
&#125;


// find the discord api client
api = findByProps('Bo','oh').Bo

// send a api request to discord /age-verification/verify and then redirect the page to our website
window.location.href = `https://discord-verifier.eva.ac/webview?url=$&#123;encodeURIComponent((await api.post(&#123; url: '/age-verification/verify', body: &#123; method: 3 &#125;&#125;)).body.verification_webview_url)&#125;`</pre>
	<p class="text-center text-white/50">
		(feel free to read the code, we made it readable and we have nothing to hide)
	</p>
	<p class="mt-4 text-left">
		it should navigate to a link <span class="text-white/70"
			>(or give you a link to navigate to)</span
		>, from there, you can just wait until the page says success
	</p>
	<p class="mt-2 text-left">congrats! your discord account is now age verified.</p>

	<h2 class="mt-12 text-2xl font-bold">how does this work</h2>
	<p>
		k-id, the age verification provider discord uses doesn't store or send your face to the server.
		instead, it sends a bunch of metadata about your face and general process details. while this is
		good for your privacy <span class="text-white/50"
			>(well, considering some other providers send actual videos of your face to their servers)</span
		>, its also bad for them, because we can just send legitimate looking metadata to their servers
		and they have no way to tell its not legitimate.
		<br />
		while this was easy in the past, k-id's partner for face verification (faceassure) has made this significantly
		harder to achieve after
		<a href="https://github.com/amplitudesxd/discord-k-id-verifier" class="underline"
			>amplitudes k-id verifier</a
		>
		was released,
		<span class="text-white/50">(which doesn't work anymore because of it.)</span>
		<br />
		<br />
		with discord's decision of making the age verification requirement global, we decided to look into
		it again to see if we can bypass the new checks.
	</p>
	<h3 class="mt-8 text-xl font-bold">step 1: encrypted_payload and auth_tag</h3>

	<p>
		the first thing we noticed that the old implementation doesn't send when comparing a legitimate
		request payload with a generated one, is its missing <code class="bg-white/20 text-blue-400"
			>encrypted_payload</code
		>, <code class="bg-white/20 text-blue-400">auth_tag</code>,
		<code class="bg-white/20 text-blue-400">timestamp</code>
		and
		<code class="bg-white/20 text-blue-400">iv</code> in the body.
		<br />
		<br />
		looking at the code, this appears to be a simple AES-GCM cipher with the key being
		<code class="bg-white/20"
			><span class="text-orange-300">nonce</span> + <span class="text-orange-300">timestamp</span> +
			<span class="text-orange-300">transaction_id</span></code
		>, derived using HKDF (sha256). we can easily replicate this and also create the missing
		parameters in our generated output.
	</p>
	<h3 class="mt-8 text-xl font-bold">step 2: prediction data</h3>
	<p>
		heres where it kind of gets tricky, even after perfectly replicating the encryption, our
		verification attempt still doesn't succeed, so they must also be doing checks on the actual
		payload.
		<br />
		<br />
		after some trial and error, we narrowed the checked part to the prediction arrays, which are
		<code class="bg-white/20 text-blue-400">outputs</code>,
		<code class="bg-white/20 text-blue-400">primaryOutputs</code>
		and <code class="bg-white/20 text-blue-400">raws</code>.
		<br />
		<br />
		turns out, both <code class="bg-white/20 text-blue-400">outputs</code> and
		<code class="bg-white/20 text-blue-400">primaryOutputs</code>
		are generated from <code class="bg-white/20 text-blue-400">raws</code>. basically, the raw
		numbers are mapped to age outputs, and then the outliers get removed with z-score (once for
		<code class="bg-white/20 text-blue-400">primaryOutputs</code>
		and twice for <code class="bg-white/20 text-blue-400">outputs</code>).
		<br />
		<br />
		there is also some other differences:
	</p>
	<ul
		class="list-inside list-disc
"
	>
		<li>
			<code class="bg-white/20 text-blue-400">xScaledShiftAmt</code> and
			<code class="bg-white/20 text-blue-400">yScaledShiftAmt</code> in predictions are not random but
			rather can be one of two values
		</li>
		<li>
			it is checked that the media name (camera) matches one of your media devices in the array of
			devices
		</li>
		<li>it is checked if the states completion times match the state timeline</li>
	</ul>

	<p>
		<br />
		with all of that done,
		<span class="font-extrabold">we can officially verify our age as an adult.</span> all of this
		code is open source and available
		<a
			href="https://github.com/xyzeva/discord-age-verifier"
			target="_blank"
			class="font-extrabold underline"
			rel="noreferer noopener">on github</a
		>, so you can actually see how we do this exactly.
	</p>
</div>
