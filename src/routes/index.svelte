<script lang="ts">
	import { onMount } from 'svelte';

	let width = 512;
	let height = 256;

	onMount(() => {
		const canvas = document.getElementById('fft') as HTMLCanvasElement;
		width = window.innerWidth;
		height = window.innerHeight;
		canvas.width = width;
		canvas.height = height;

		document.body.appendChild(canvas);

		const ctx = canvas.getContext('2d') as CanvasRenderingContext2D;

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

		let context: AudioContext;
		let analyser: AnalyserNode;
		let mediaSource;
		let imageData;
		let timeData: Uint8Array;

		function connectAudioAPI() {
			try {
				context = new AudioContext();
				analyser = context.createAnalyser();
				analyser.fftSize = 2048;

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

		function updateFFT() {
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

				const l = data > 10 ? '50%' : '0%';

				ctx.fillStyle = `hsl(${hslStart - data}, 100%, ${l})`;

				const ratio = height / (dataEnd - dataStart);

				const ypos = Math.floor(ratio * y);
				const next_ypos = Math.floor(ratio * (y + 1)) - ypos;
				const h = next_ypos - ypos;

				// seems to affect wave height
				ctx.fillRect(width - 1, height - Math.floor(ratio * y), 1, h);
			}
		}

		function animate() {
			updateFFT();
			requestAnimationFrame(animate);
		}

		window.onload = function () {
			connectAudioAPI();
		};
	});
</script>

<canvas id="fft" />

<style>
	:global(body, html, canvas) {
		padding: 0;
		margin: 0;

		width: 100vw;
		height: 100vh;

		overflow: hidden;

		background: black;
	}
</style>
