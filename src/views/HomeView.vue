<template>
  <div>
    <video ref="video" autoplay></video>
    <button @click="startCaptureCountdown" :disabled="isCapturing">开始捕获</button>
    <button @click="stopCapture" :disabled="!isCapturing">停止捕获</button>
    <button @click="downloadFramesZip" v-if="frames.length > 0">下载ZIP</button>
    <button @click="toggleFlashlight">切换闪光灯</button>

    <!-- 上传进度条 -->
    <div class="progress-bar" v-if="uploadProgress > 0">
      <div class="progress-fill" :style="{ width: uploadProgress + '%' }"></div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import JSZip from 'jszip';

export default {
  data() {
    return {
      video: null,
      isCapturing: false,
      captureInterval: null,
      countdownInterval: null,
      frames: [],
      frameRate: 30,
      frameDuration: 1000 / 30,
      captureDuration: 10000,
      countdownDuration: 3,
      uploadProgress: 0,
      frames_timestmap: {},
      count: 0,
      flashlightOn: false,
    };
  },
  methods: {
    async downloadFramesZip() {
      if (this.frames.length === 0) {
        return;
      }

      const zip = new JSZip();

      for (let i = 0; i < this.frames.length; i++) {
        const frameData = this.frames[i];
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        canvas.width = this.video.videoWidth;
        canvas.height = this.video.videoHeight;
        context.putImageData(new ImageData(new Uint8ClampedArray(frameData), canvas.width, canvas.height), 0, 0);
        const imgDataUrl = canvas.toDataURL('image/png');
        zip.file(`frame${i + 1}.png`, imgDataUrl.split('base64,')[1], { base64: true });
      }

      zip.generateAsync({ type: 'blob' })
        .then((content) => {
          const url = window.URL.createObjectURL(new Blob([content]));
          const a = document.createElement('a');
          a.href = url;
          a.download = 'captured_frames.zip';
          a.click();
          window.URL.revokeObjectURL(url);
        })
        .catch((error) => {
          console.error('ZIP生成失败', error);
        });
    },

    async startCaptureCountdown() {
      this.video = this.$refs.video;
      this.frames = [];
      this.isCapturing = true;
      this.startVideo();

      this.countdownInterval = setInterval(() => {
        this.countdownDuration--;
        if (this.countdownDuration <= 0) {
          clearInterval(this.countdownInterval);
          this.startCapture();
        }
      }, 1000);
    },

    startVideo() {
      navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: { exact: "environment" },
          torch: this.flashlightOn,
        }
      })
      .then((stream) => {
        this.video.srcObject = stream;
        this.enableFlashlight();
      })
      .catch((error) => {
        console.error('无法打开摄像头：', error);
        this.isCapturing = false;
      });
    },

    enableFlashlight() {
  const tracks = this.video.srcObject.getTracks();
  if (tracks.length > 0) {
    const capabilities = tracks[0].getCapabilities();
    console.log('Capabilities:', capabilities);

    if (capabilities.torch) {
      this.flashlightOn = true;
      tracks[0].applyConstraints({
        advanced: [{ torch: this.flashlightOn }]
      })
      .then(() => {
        console.log('Torch enabled successfully');
      })
      .catch((error) => {
        console.error('Error enabling torch:', error);
        alert('无法启用闪光灯：' + error.message); // 弹出包含错误信息的提示框
      });
    } else {
      console.warn('Torch control is not supported on this device.');
    }
  }
},


    startCapture() {
      this.captureInterval = setInterval(this.captureFrame, this.frameDuration);
      setTimeout(this.stopCapture, this.captureDuration);
    },

    stopCapture() {
      clearInterval(this.captureInterval);
      this.isCapturing = false;
      this.sendFramesToServer();
      this.video.srcObject.getTracks().forEach(track => track.stop());
      this.countdownDuration = 3;
    },

    captureFrame() {
      const canvas = document.createElement('canvas');
      const canvasContext = canvas.getContext('2d');
      canvas.width = this.video.videoWidth;
      canvas.height = this.video.videoHeight;
      canvasContext.drawImage(this.video, 0, 0, canvas.width, canvas.height);
      const frameData = canvasContext.getImageData(0, 0, this.video.videoWidth, this.video.videoHeight);
      this.frames_timestmap[String(this.count++)] = Date.now();
      this.frames.push(frameData.data.buffer);
    },

    sendFramesToServer() {
      if (this.frames.length === 0) {
        return;
      }
      const jsonData = JSON.stringify(this.frames_timestmap);
      axios
        .post('http://127.0.0.1:5000/upload_frames_timestmap', jsonData, {
          headers: {'Content-Type': 'application/json'},
        })
        .then(response => {
          console.log("时间戳已上传");
        })
        .catch(error => {
          console.log(error.message);
        });

      const frameBlob = new Blob(this.frames, { type: 'application/octet-stream' });
      const formData = new FormData();
      formData.append('frames', frameBlob);

      axios
        .post('http://127.0.0.1:5000/upload_frames', formData, {
          onUploadProgress: (progressEvent) => {
            this.uploadProgress = Math.round((progressEvent.loaded / progressEvent.total) * 100);
            if (this.uploadProgress === 100) {
              console.log("上传完毕");
            }
          }
        })
        .then((response) => {
          console.log('上传完成', response);
          this.uploadProgress = 0; // 重置上传进度
        })
        .catch((error) => {
          console.error('上传失败', error);
          this.uploadProgress = 0; // 重置上传进度
        });
    },

    toggleFlashlight() {
      const tracks = this.video.srcObject.getTracks();
      if (tracks.length > 0) {
        const capabilities = tracks[0].getCapabilities();
        if (capabilities.torch) {
          this.flashlightOn = !this.flashlightOn;
          tracks[0].applyConstraints({
            advanced: [{ torch: this.flashlightOn }]
          });
        } else {
          console.warn('Torch control is not supported on this device.');
        }
      }
    },
  },
};
</script>

<style>
/* 进度条样式 */
.progress-bar {
  width: 100%;
  background: #ccc;
  height: 20px;
  position: relative;
}

.progress-fill {
  height: 100%;
  background: #4caf50; /* 进度条填充的颜色 */
  transition: width 0.3s ease-in-out;
}
</style>