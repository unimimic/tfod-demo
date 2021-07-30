<template>
  <div id="web-camera-container">
    <div class="container shadow min-vh-100">
      <div class="row text-center">
        <h5 class="card-title">Tensorflow.js Demo (Options API)</h5>
      </div>
      <div class="row text-center p-3">
        <select class="form-select" aria-label="Default select example" v-model="selected" :disabled='isOpen'>
          <option disabled value="">Please select one</option>
          <option v-for="camera in camList" :value="camera.deviceId" :key="camera.deviceId">{{ camera.label }}</option>
        </select>
      </div>
      <div class="row text-center p-3">
        <div class="camera-button">
          <span v-if="!isReady">loading AI Model... </span>
          <button class="btn btn-primary" type="button" :disabled='!isReady' @click="toggleCamera">
            <span v-if="!isOpen">Open Camera</span>
            <span v-else>Close Camera</span>
          </button>
        </div>
      </div>
      <div class="row position-relative text-center">
        <video autoplay ref="camera" class="position-absolute mw-100"/>
        <canvas :width="camWidth" :height="camHeight" ref="canvas" class="position-absolute mw-100"/>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * Options API
 * >> 共同處理頁面邏輯
 **/
import '@tensorflow/tfjs';
import * as cocoSsd from '@tensorflow-models/coco-ssd';
import "bootstrap/dist/css/bootstrap.min.css";
let AImodel = null;

export default {
  name: 'optionsAPI',
  data() {
    return {
      isOpen: false,
      isReady: false,
      camWidth: "640",
      camHeight: "480",
      camList: [],
      selected: ''
    }
  },
  methods: {
    /**
     * load AI Model
     **/
    async loadModel(){
      try{
        AImodel = await cocoSsd.load();
        this.isReady = true
        console.log('model loaded')
      }catch(e){
          console.log('loaded model fail')
          console.log(e)
      }
    },
    /**
     * load available cameras
     **/
    async loadCameras(){
      try{
        /* open a generic stream to get permission to see devices; 
          * Mobile Safari insists */
        await navigator.mediaDevices.getUserMedia({audio: false, video: true});
        let deviceInfos = await navigator.mediaDevices.enumerateDevices()
        for (let i = 0; i !== deviceInfos.length; ++i) {
          let deviceInfo = deviceInfos[i];
          if (deviceInfo.kind === "videoinput") {
            this.camList.push(deviceInfo);
          }
        }
      }catch(e){
        console.log("not supported",e)
      }
    },
    /**
     * get camera
     **/
    createCameraElement() {
      // 1.check
      if (this.selected === ""){
        alert("selct one device")
        return
      }
      // 2.setting
      let constraints = { video: { deviceId: { exact: this.selected } } };
      // 3.加載
			navigator.mediaDevices
				.getUserMedia(constraints)
				.then(stream => {
					this.$refs.camera.srcObject = stream;
          this.isOpen = true;
				})
				.catch(error => {
          console.log(error)
					alert("May the browser didn't support or there is some errors.");
				});
    },
    /**
     * stop Camera
     **/
    stopCameraStream() {
      let tracks = this.$refs.camera.srcObject.getTracks();
			tracks.forEach(track => {
				track.stop();
			});
    },
    /**
     * toggle Camera
     **/
    toggleCamera() {
      try{
        if(this.isOpen) {
          this.stopCameraStream();
          this.isOpen = false;
        } else {
          this.createCameraElement();
        }
      }catch(e){
        console.log(e);
      }
    },
    /**
     * detect Objects
     **/
    async detectObjects() {
      // check
      if (!this.isReady) return
      if (!this.isOpen) return
      // predict
      let predictions = await AImodel.detect(this.$refs.camera);
      // go loop
      this.renderPredictions(predictions);
      requestAnimationFrame(() => {
        this.detectObjects()
      })
    },
    /**
     * render on canvas
     **/
    renderPredictions (predictions) {
      // get the context of canvas
      const ctx = this.$refs.canvas.getContext('2d')
      ctx.font = "24px helvetica";
      // clear the canvas
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
      predictions.forEach(prediction => {
        const x = prediction.bbox[0];
        const y = prediction.bbox[1];
        // const width = prediction.bbox[2];

        ctx.strokeStyle = "#2fff00";
        ctx.lineWidth = 1;
        ctx.strokeRect(...prediction.bbox)

        ctx.fillStyle="#2fff00";
        let Text = `${(prediction.score * 100).toFixed(1)}% ${prediction.class}`
        const textWidth = ctx.measureText(Text).width;
        const textHeight = parseInt("24px helvetica", 10);
        ctx.fillRect(x,y,textWidth+10, textHeight+10);  

        ctx.fillStyle="#000000";
        ctx.fillText(Text, x, y+textHeight)
      })
    }
  },
  mounted(){
    if(navigator.mediaDevices.getUserMedia
      ||navigator.mediaDevices.webkitGetUserMedia){
      // init
      this.loadModel()
      this.loadCameras()

      this.$refs.camera.addEventListener('loadeddata', () => {
          this.camHeight = this.$refs.camera.videoHeight
          this.camWidth = this.$refs.camera.videoWidth
          this.detectObjects();
      })
    }else{
      alert("not supported");
    }
    // // Optional: Go full screen.
    // document.documentElement.requestFullscreen({
    //   navigationUI: "hide"
    // });
  }
}
</script>