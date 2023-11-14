<template>
  <div id="app">
    <h1>Flashlight App</h1>
    <button @click="toggleFlashlight">{{ flashlightStatus }}</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      flashlightStatus: '打开闪光灯',
      isFlashlightOn: false,
    };
  },
  methods: {
    async toggleFlashlight() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
        const track = stream.getVideoTracks()[0];
        const capabilities = track.getCapabilities();

        if (capabilities.torch) {
          await track.applyConstraints({ advanced: [{ torch: this.isFlashlightOn }] });
          this.isFlashlightOn = !this.isFlashlightOn;
          this.flashlightStatus = this.isFlashlightOn ? '关闭闪光灯' : '打开闪光灯';
        } else {
          console.error('设备不支持闪光灯');
        }
      } catch (error) {
        console.error('无法访问摄像头或用户拒绝了访问权限', error);
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
