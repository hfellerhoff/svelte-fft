<script lang="ts">
	import { onMount } from 'svelte';

	let width = 512;
	let height = 256;

	let isActive = true;

	let context: AudioContext;
	let analyser: AnalyserNode;
	let mediaSource;
	let imageData;
	let timeData: Uint8Array;
	let ctx: CanvasRenderingContext2D;
	let fftSize = 2048;
	let coloring = 'linear';

	function updateFFT() {
		if (!ctx) return;
		timeData = new Uint8Array(analyser.frequencyBinCount);

		analyser.getByteFrequencyData(timeData);

		imageData = ctx.getImageData(1, 0, width - 1, height);
		ctx.putImageData(imageData, 0, 0);

		const dataLength = 1024;
		const dataStart = 0;
		const dataEnd = 256;

		for (let y = dataStart; y < dataEnd; y++) {
			const data = timeData[y];

			const hslStart = 255;
			const dataFloor = 25;
			const hslfactor = hslStart / (hslStart - dataFloor);

			const x = hslStart - data;
			let color = x;
			if (coloring === 'logarithmic') {
				color = Math.pow(x, 2.3) / 1000;
			}

			if (color < 0) color = 0;
			if (color > 255) color = 255;
			const l = data > dataFloor && color < 255 ? '50%' : '0%';

			ctx.fillStyle = `hsl(${color}, 100%, ${l})`;

			const ratio = height / (dataEnd - dataStart);

			const ypos = Math.floor(ratio * y);
			const next_ypos = Math.floor(ratio * (y + 1)) - ypos;
			const h = next_ypos - ypos;

			// seems to affect wave height
			ctx.fillRect(width - 1, height - Math.floor(ratio * y), 1, h);
		}
	}

	function animate() {
		if (!isActive) return;

		updateFFT();
		requestAnimationFrame(animate);
	}

	onMount(() => {
		const canvas = document.getElementById('fft') as HTMLCanvasElement;
		width = window.innerWidth;
		height = window.innerHeight;
		canvas.width = width;
		canvas.height = height;

		document.body.appendChild(canvas);

		ctx = canvas.getContext('2d') as CanvasRenderingContext2D;

		ctx.fillStyle = `rgba(0,0,0)`;
		ctx.fillRect(0, 0, width, height);

		window.addEventListener('resize', onWindowResize, false);
		function onWindowResize() {
			width = window.innerWidth;
			height = window.innerHeight;
			canvas.width = width;
			canvas.height = height;
			animate();
		}

		function connectAudioAPI() {
			try {
				context = new AudioContext();
				analyser = context.createAnalyser();
				analyser.fftSize = fftSize;

				navigator.mediaDevices
					.getUserMedia({ audio: true, video: false })
					.then(function (stream) {
						mediaSource = context.createMediaStreamSource(stream);
						mediaSource.connect(analyser);
						animate();
					})
					.catch(function (err) {
						alert(err);
					});
			} catch (e) {
				alert(e);
			}
		}

		window.onload = function () {
			connectAudioAPI();
		};

		window.addEventListener('keydown', (e) => {
			if (e.key === ' ') toggleActive();
		});
	});

	const toggleActive = () => {
		isActive = !isActive;
		animate();
	};

	const updateFFTSize = () => {
		analyser.fftSize = fftSize;
	};

	const sampleCounts = [32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384, 32768];
	const colors = ['linear', 'logarithmic'];
</script>

<canvas id="fft" />

<div class="controls">
	<label for="fft-playback">Playback</label>
	<button id="fft-playback" on:click={toggleActive}>
		{#if isActive}
			Stop
		{:else}
			Start
		{/if}
	</button>
	<label for="fft-samplesize">Sample Size</label>
	<select id="fft-samplesize" bind:value={fftSize} on:change={updateFFTSize}>
		{#each sampleCounts as count}
			<option value={count}>{count}</option>
		{/each}
	</select>
	<label for="fft-coloring">Coloring Function</label>
	<select id="fft-coloring" bind:value={coloring} on:change={updateFFTSize}>
		{#each colors as color}
			<option value={color}>{color[0].toUpperCase() + color.substring(1)}</option>
		{/each}
	</select>
</div>

<style>
	:global(body, html, canvas) {
		padding: 0;
		margin: 0;

		width: 100vw;
		height: 100vh;

		overflow: hidden;

		background: black;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
			'Open Sans', 'Helvetica Neue', sans-serif;
	}

	div {
		position: absolute;
		top: 0;
		left: 0;
		padding: 1rem;
		color: whitesmoke;
		background: #222;

		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 0.5rem;
		place-items: center;
	}
	label {
		width: 100%;
		font-size: 0.9rem;
	}

	button,
	select {
		background: #111;
		border: none;

		width: 7rem;
		height: 2rem;
		color: inherit;
	}

	select {
		padding-left: 0.5rem;
	}
</style>
