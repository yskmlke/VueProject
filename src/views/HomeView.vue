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
        const capabilities = track.getCapabilities();

        if (capabilities.torch) {
          await track.applyConstraints({ advanced: [{ torch: true }] });
          alert('闪光灯已打开');
        } else {
          alert('设备不支持闪光灯');
        }
      } catch (error) {
        alert('无法访问摄像头或用户拒绝了访问权限');
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
