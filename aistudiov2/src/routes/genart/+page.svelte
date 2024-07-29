<script>
	import { onMount, onDestroy } from 'svelte';
	import logoSvg from '$lib/images/logo.svg';
  
	let sketchElement;
	let p5Instance;
	let videoElement;
  
	onMount(async () => {
	  const p5 = await import('p5');
	  
	  const sketch = (p) => {
		let nodes = [];
		let maxNodes = 200;
		let maxDistance = 100;
		let heartRate = 0;
  
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
		  p.background(0);
		  p.stroke(255);
		  p.noFill();
  
		  let heartbeat = (p.sin(heartRate * p.TWO_PI) + 1) / 2;
		  heartRate += 0.005;
  
		  nodes.forEach(node => {
			node.position.x = p.map(p.noise(node.noiseOffset.x + heartRate), 0, 1, 0, p.width);
			node.position.y = p.map(p.noise(node.noiseOffset.y + heartRate), 0, 1, 0, p.height);
  
			p.fill(255);
			p.ellipse(node.position.x, node.position.y, 4, 4);
		  });
  
		  p.stroke(255, 100);
		  for (let i = 0; i < nodes.length; i++) {
			for (let j = i + 1; j < nodes.length; j++) {
			  let distance = p.dist(nodes[i].position.x, nodes[i].position.y, nodes[j].position.x, nodes[j].position.y);
			  if (distance < maxDistance) {
				p.line(nodes[i].position.x, nodes[i].position.y, nodes[j].position.x, nodes[j].position.y);
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
		}
	  } catch (err) {
		console.error("ウェブカメラへのアクセスに失敗しました:", err);
	  }
	});
  
	onDestroy(() => {
	  if (p5Instance) {
		p5Instance.remove();
	  }
	  // ウェブカメラのストリームを停止
	  if (videoElement && videoElement.srcObject) {
		const tracks = videoElement.srcObject.getTracks();
		tracks.forEach(track => track.stop());
	  }
	});
  </script>
  
  <div class="layers-container">
	<div class="genAiLayer" bind:this={sketchElement}></div>
	<div class="webCamLayer">
	  <video bind:this={videoElement} autoplay playsinline></video>
	</div>
	<div class="logoLayer">
	  <img src={logoSvg} alt="Logo" />
	</div>
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
  
	.genAiLayer {
	  position: absolute;
	  width: 100%;
	  height: 100%;
	  z-index: 1;
	}
  
	.webCamLayer {
	  position: absolute;
	  top: 20%;
	  left: 20%;
	  width: 60%;
	  height: 60%;
	  z-index: 2;
	  overflow: hidden;
	}
  
	.webCamLayer video {
	  width: 100%;
	  height: 100%;
	  object-fit: cover;
	  opacity: 0.6;
	  filter: brightness(180%) contrast(80%); /* 映像をより白っぽくする */
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
	  max-width: 100%;
	  max-height: 100%;
	  object-fit: contain;
	}
  </style>