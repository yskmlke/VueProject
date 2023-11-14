<template>
  <div id="app">
    <h1>Flashlight App</h1>
    <button @click="toggleFlashlight">Toggle Flashlight</button>
  </div>
</template>

<script>
export default {
  methods: {
    async toggleFlashlight() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
        const track = stream.getVideoTracks()[0];
        
        if (!track) {
          alert('未找到摄像头轨道，可能设备不支持闪光灯');
          return;
        }

        const capabilities = track.getCapabilities();

        if (!capabilities.torch) {
          alert('设备不支持闪光灯');
          return;
        }

        await track.applyConstraints({ advanced: [{ torch: true }] });
        alert('闪光灯已打开');
      } catch (error) {
        if (error.name === 'NotAllowedError' || error.name === 'NotFoundError') {
          alert('无法访问摄像头或用户拒绝了访问权限');
        } else {
          alert('发生未知错误: ' + error.message);
        }
      }
    },
  },
};
</script>

<style>
#app {
  text-align: center;
  margin-top: 60px;
}

button {
  padding: 10px;
  font-size: 16px;
  cursor: pointer;
}
</style>
