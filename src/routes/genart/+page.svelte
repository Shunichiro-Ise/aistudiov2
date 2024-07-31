<script>
	import { onMount, onDestroy } from 'svelte';
	import logoSvg from '$lib/images/aistudio.svg';
	import audioSrc from '$lib/Evolving_Code.mp3';
  
	let sketchElement;
	let p5Instance;
	let videoElement;
	let canvasElement;
	let ctx;
	let previousImageData;
	let frameCount = 0;
	const FRAME_SKIP = 10;
	const MOVEMENT_THRESHOLDS = [2, 5, 10];
	let heartRateMultiplier = 0.5;
	let audioElement;
	let isAudioPlaying = false;
	let isAppStarted = false;
  
	onMount(async () => {
	  // オーディオの設定
	  if (audioElement) {
		audioElement.src = audioSrc;
		audioElement.loop = true;
		console.log("Audio element set up with source:", audioSrc);
	  } else {
		console.error("Audio element not found");
	  }
	});

	async function startApp() {
	  isAppStarted = true;
	  await playAudio();
	  initializeApp();
	}

	async function playAudio() {
	  if (audioElement && !isAudioPlaying) {
		try {
		  await audioElement.play();
		  console.log("Audio started playing");
		  isAudioPlaying = true;
		} catch (error) {
		  console.error("音楽の再生に失敗しました:", error);
		}
	  } else if (isAudioPlaying) {
		console.log("Audio is already playing");
	  } else {
		console.error("Audio element not initialized");
	  }
	}

	async function initializeApp() {
	  const p5 = await import('p5');
	  
	  const sketch = (p) => {
		let nodes = [];
		let maxNodes = 175;
		let maxDistance = 175;
		let heartRate = 0;
		const baseHeartRateSpeed = 0.005;

		// 色と大きさの設定 TCS Blue:#4E84C4 TCS Yellow:#F0B34F
		const BACKGROUND_COLOR = '#000000';  
		const BACKGROUND_ALPHA = 255;  // 不透明度 (0-255の範囲)
		const NODE_SIZE = 8;
		const NODE_COLOR = '#2F9BFF';  // 
		const EDGE_WEIGHT = 1;
		const EDGE_COLOR = '#FFFFFF';  // 緑色
		const EDGE_ALPHA = 100;  // 不透明度 (0-255の範囲)
  
		p.setup = () => {
		  p.createCanvas(p.windowWidth, p.windowHeight);
		  for (let i = 0; i < maxNodes; i++) {
			nodes.push({
			  position: p.createVector(p.random(p.width), p.random(p.height)),
			  noiseOffset: p.createVector(p.random(1000), p.random(1000))
			});
		  }
		};
  
		p.draw = () => {
			// 背景色の設定
			const bgColor = p.color(BACKGROUND_COLOR);
			bgColor.setAlpha(BACKGROUND_ALPHA);
			p.background(bgColor);

			p.noStroke();

			let heartbeat = (p.sin(heartRate * p.TWO_PI) + 1) / 2;
			heartRate += baseHeartRateSpeed * heartRateMultiplier;

			const mappingWidth = p.width * 2;
			const mappingHeight = p.height * 2;
			
			const offsetX = (mappingWidth - p.width) / 2;
			const offsetY = (mappingHeight - p.height) / 2;

			nodes.forEach(node => {
				node.position.x = p.map(p.noise(node.noiseOffset.x + heartRate), 0, 1, -offsetX, mappingWidth - offsetX);
				node.position.y = p.map(p.noise(node.noiseOffset.y + heartRate), 0, 1, -offsetY, mappingHeight - offsetY);

				if (node.position.x >= 0 && node.position.x <= p.width && 
					node.position.y >= 0 && node.position.y <= p.height) {
				p.fill(NODE_COLOR);
				p.ellipse(node.position.x, node.position.y, NODE_SIZE, NODE_SIZE);
				}
			});

			p.strokeWeight(EDGE_WEIGHT);
			const edgeColor = p.color(EDGE_COLOR);
			edgeColor.setAlpha(EDGE_ALPHA);
			p.stroke(edgeColor);
			for (let i = 0; i < nodes.length; i++) {
				for (let j = i + 1; j < nodes.length; j++) {
				if (nodes[i].position.x >= 0 && nodes[i].position.x <= p.width && 
					nodes[i].position.y >= 0 && nodes[i].position.y <= p.height &&
					nodes[j].position.x >= 0 && nodes[j].position.x <= p.width && 
					nodes[j].position.y >= 0 && nodes[j].position.y <= p.height) {
					let distance = p.dist(nodes[i].position.x, nodes[i].position.y, nodes[j].position.x, nodes[j].position.y);
					if (distance < maxDistance) {
					p.line(nodes[i].position.x, nodes[i].position.y, nodes[j].position.x, nodes[j].position.y);
					}
				}
				}
			}
		};
  
		p.windowResized = () => {
		  p.resizeCanvas(p.windowWidth, p.windowHeight);
		};
	  };
  
	  p5Instance = new p5.default(sketch, sketchElement);
  
	  // ウェブカメラの設定
	  try {
		const stream = await navigator.mediaDevices.getUserMedia({ video: true });
		if (videoElement) {
		  videoElement.srcObject = stream;
		  videoElement.onloadedmetadata = () => {
			canvasElement.width = videoElement.videoWidth;
			canvasElement.height = videoElement.videoHeight;
			ctx = canvasElement.getContext('2d', { willReadFrequently: true });
			requestAnimationFrame(processFrame);
		  };
		}
	  } catch (err) {
		console.error("ウェブカメラへのアクセスに失敗しました:", err);
	  }
	}
  
	function processFrame() {
	  frameCount++;
	  if (videoElement.readyState === videoElement.HAVE_ENOUGH_DATA) {
		ctx.drawImage(videoElement, 0, 0, canvasElement.width, canvasElement.height);
		
		if (frameCount % FRAME_SKIP === 0) {
		  const imageData = ctx.getImageData(0, 0, canvasElement.width, canvasElement.height);
		  
		  if (previousImageData) {
			const diff = pixelDiff(imageData.data, previousImageData.data);
			const normalizedDiff = diff / (canvasElement.width * canvasElement.height);
			
			// 4段階の動き検出
			if (normalizedDiff < MOVEMENT_THRESHOLDS[0]) {
			  heartRateMultiplier = 0.3; // 非常に遅い
			} else if (normalizedDiff < MOVEMENT_THRESHOLDS[1]) {
			  heartRateMultiplier = 0.6; // 遅い
			} else if (normalizedDiff < MOVEMENT_THRESHOLDS[2]) {
			  heartRateMultiplier = 0.9; // 通常
			} else {
			  heartRateMultiplier = 1.2; // 速い
			}
		  }
		  
		  previousImageData = imageData;
		}
	  }
	  
	  requestAnimationFrame(processFrame);
	}
  
	function pixelDiff(data1, data2) {
	  let diff = 0;
	  for (let i = 0; i < data1.length; i += 64) { // サンプリングレートを下げる
		diff += Math.abs(data1[i] - data2[i]);
	  }
	  return diff * 16; // サンプリングレートの調整を補正
	}
  
	onDestroy(() => {
	  if (p5Instance) {
		p5Instance.remove();
	  }
	  // ウェブカメラのストリームを停止
	  if (videoElement && videoElement.srcObject) {
		const tracks = videoElement.srcObject.getTracks();
		tracks.forEach(track => track.stop());
	  }
	  // オーディオの停止
	  if (audioElement) {
		audioElement.pause();
		audioElement.currentTime = 0;
		isAudioPlaying = false;
		console.log("Audio stopped");
	  }
	});
</script>
  
<div class="layers-container">
	{#if !isAppStarted}
		<div class="start-screen">
			<h1>Welcome to the Interactive Visualization</h1>
			<button on:click={startApp}>Start Experience</button>
		</div>
	{:else}
		<div class="genAiLayer" bind:this={sketchElement}></div>
		<div class="webCamLayer">
		<video bind:this={videoElement} autoplay playsinline></video>
		<canvas bind:this={canvasElement} style="display: none;"></canvas>
		</div>
		<div class="logoLayer">
		<img src={logoSvg} alt="Logo" />
		</div>
	{/if}
	<audio bind:this={audioElement}></audio>
</div>
  
<style>
	:global(body) {
	  margin: 0;
	  padding: 0;
	  overflow: hidden;
	}
  
	.layers-container {
	  position: fixed;
	  top: 0;
	  left: 0;
	  width: 100%;
	  height: 100%;
	}
  
	.start-screen {
	  display: flex;
	  flex-direction: column;
	  justify-content: center;
	  align-items: center;
	  height: 100%;
	  background-color: #000000;
	  color: #FFFFFF;
	}

	.start-screen h1 {
	  font-size: 2em;
	  margin-bottom: 20px;
	}

	.start-screen button {
	  padding: 10px 20px;
	  font-size: 1.2em;
	  cursor: pointer;
	  background-color: #2F9BFF;
	  color: #FFFFFF;
	  border: none;
	  border-radius: 5px;
	}

	.genAiLayer {
	  position: absolute;
	  width: 100%;
	  height: 100%;
	  z-index: 1;
	}
  
	.webCamLayer {
	  position: absolute;
	  top: 17.5%;
	  left: 17.5%;
	  width: 65%;
	  height: 65%;
	  z-index: 2;
	  overflow: hidden;
	  border-radius: 20px;
	}
  
	.webCamLayer video {
	  width: 100%;
	  height: 100%;
	  object-fit: cover;
	  opacity: 0.7;
	  filter: brightness(150%) contrast(50%) saturate(50%);
	}
  
	.logoLayer {
	  position: absolute;
	  top: 25%;
	  left: 25%;
	  width: 50%;
	  height: 50%;
	  z-index: 3;
	  display: flex;
	  justify-content: center;
	  align-items: center;
	}
  
	.logoLayer img {
	  max-width: 70%;
	  max-height: 70%;
	  object-fit: contain;
	  filter: brightness(0) invert(1);
	}
</style>