<template>
  <div class="container">
    <video ref="videoElement" style="display: none;"></video>
    <canvas ref="threeCanvas"></canvas>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted } from 'vue';
import { SelfieSegmentation } from '@mediapipe/selfie_segmentation';
import * as THREE from 'three';

export default {
  name: 'EnhancedMilkyWaySilhouette',
  setup() {
    const videoElement = ref(null);
    const threeCanvas = ref(null);
    let selfieSegmentation;
    let scene, camera, renderer, silhouetteParticles, starField;
    let clock;

    const initializeMediaPipe = async () => {
      selfieSegmentation = new SelfieSegmentation({
        locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`
      });

      selfieSegmentation.setOptions({
        modelSelection: 1,
        selfieMode: false,
      });

      selfieSegmentation.onResults(onResults);

      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 640, height: 480 } });
          if (videoElement.value) {
            videoElement.value.srcObject = stream;
            videoElement.value.addEventListener('loadeddata', async () => {
              videoElement.value.play();
              detectSegmentation();
            });
          }
        } catch (error) {
          console.error("Error accessing the camera:", error);
        }
      }
    };

    const initializeThreeJS = () => {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
      renderer = new THREE.WebGLRenderer({ canvas: threeCanvas.value, alpha: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      clock = new THREE.Clock();

      // Create particle system for silhouette outline
      const geometry = new THREE.BufferGeometry();
      const positions = new Float32Array(15000 * 3); // Increased particle count
      const velocities = new Float32Array(15000 * 3); // For particle movement
      geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute('velocity', new THREE.BufferAttribute(velocities, 3));
      const material = new THREE.PointsMaterial({ 
        size: 2, 
        color: 0xFFFFFF, 
        transparent: true, 
        opacity: 0.7 // Slightly increased opacity
      });
      silhouetteParticles = new THREE.Points(geometry, material);
      scene.add(silhouetteParticles);

      // Create starfield
      const starGeometry = new THREE.BufferGeometry();
      const starPositions = new Float32Array(5000 * 3);
      for (let i = 0; i < starPositions.length; i += 3) {
        starPositions[i] = Math.random() * 2000 - 1000;
        starPositions[i + 1] = Math.random() * 2000 - 1000;
        starPositions[i + 2] = Math.random() * 2000 - 1000;
      }
      starGeometry.setAttribute('position', new THREE.BufferAttribute(starPositions, 3));
      const starMaterial = new THREE.PointsMaterial({ color: 0xFFFFFF, size: 1, transparent: true, opacity: 0.5 });
      starField = new THREE.Points(starGeometry, starMaterial);
      scene.add(starField);

      // Position camera behind and to the right
      camera.position.set(200, 0, 500);
      camera.lookAt(0, 0, 0);
    };

    const detectSegmentation = async () => {
      if (videoElement.value) {
        await selfieSegmentation.send({ image: videoElement.value });
      }
      requestAnimationFrame(detectSegmentation);
    };

    const onResults = (results) => {
      if (!results.segmentationMask) return;
      updateSilhouetteOutline(results.segmentationMask);
    };

    const updateSilhouetteOutline = (segmentationMask) => {
      const maskCanvas = document.createElement('canvas');
      maskCanvas.width = 640;
      maskCanvas.height = 480;
      const maskCtx = maskCanvas.getContext('2d');
      maskCtx.drawImage(segmentationMask, 0, 0, 640, 480);
      const maskData = maskCtx.getImageData(0, 0, 640, 480).data;

      const positions = silhouetteParticles.geometry.attributes.position.array;
      const velocities = silhouetteParticles.geometry.attributes.velocity.array;
      let j = 0;

      for (let y = 0; y < 480; y += 3) { // Decreased step size for more particles
        for (let x = 0; x < 640; x += 3) {
          const i = (y * 640 + x) * 4;
          if (
            maskData[i] > 0 && (
              x === 0 || y === 0 || x === 637 || y === 477 ||
              maskData[i - 4] === 0 || maskData[i + 4] === 0 ||
              maskData[i - 640 * 4] === 0 || maskData[i + 640 * 4] === 0
            )
          ) {
            positions[j] = (x - 320) * 1; // Increased scale factor
            positions[j + 1] = (240 - y) * 1; // Increased scale factor
            positions[j + 2] = 0;
            
            // Initialize velocity for particle movement along contour
            velocities[j] = (Math.random() - 0.5) * 0.5;
            velocities[j + 1] = (Math.random() - 0.5) * 0.5;
            velocities[j + 2] = 0;
            
            j += 3;
          }
        }
      }

      // Hide unused particles
      for (let i = j; i < positions.length; i++) {
        positions[i] = 10000;
        velocities[i] = 0;
      }

      silhouetteParticles.geometry.attributes.position.needsUpdate = true;
      silhouetteParticles.geometry.attributes.velocity.needsUpdate = true;
    };

    const animate = () => {
      requestAnimationFrame(animate);
      const delta = clock.getDelta();

      // Animate particles along the contour
      const positions = silhouetteParticles.geometry.attributes.position.array;
      const velocities = silhouetteParticles.geometry.attributes.velocity.array;
      for (let i = 0; i < positions.length; i += 3) {
        if (positions[i] !== 10000) {
          positions[i] += velocities[i] * delta * 10;
          positions[i + 1] += velocities[i + 1] * delta * 10;
          
          // Bounce effect when reaching the edge of the silhouette
          if (Math.abs(positions[i]) > 320 || Math.abs(positions[i + 1]) > 240) {
            velocities[i] *= -1;
            velocities[i + 1] *= -1;
          }
        }
      }
      silhouetteParticles.geometry.attributes.position.needsUpdate = true;

      starField.rotation.y += 0.00005; // Very slow rotation
      renderer.render(scene, camera);
    };

    onMounted(() => {
      initializeMediaPipe();
      initializeThreeJS();
      animate();
      window.addEventListener('resize', onWindowResize, false);
    });

    onUnmounted(() => {
      if (selfieSegmentation) selfieSegmentation.close();
      if (videoElement.value && videoElement.value.srcObject) {
        videoElement.value.srcObject.getTracks().forEach(track => track.stop());
      }
      window.removeEventListener('resize', onWindowResize, false);
    });

    const onWindowResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    };

    return { videoElement, threeCanvas };
  }
}
</script>

<style scoped>
.container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: black;
}
canvas {
  width: 100%;
  height: 100%;
}
</style>