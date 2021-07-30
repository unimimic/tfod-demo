<template>
  <div id="web-camera-container">
    <div class="container shadow min-vh-100">
      <div class="row text-center">
        <h5 class="card-title">Tensorflow.js Demo (Composition API)</h5>
      </div>
      <div class="row text-center p-3">
        <select class="form-select" aria-label="Default select example" v-model="camera.selected" :disabled='camera.isOpen'>
          <option disabled value="">Please select one</option>
          <option v-for="item in camera.camList" :value="item.deviceId" :key="item.deviceId">{{ item.label }}</option>
        </select>
      </div>
      <div class="row text-center p-3">
        <div class="camera-button">
          <span v-if="!model.isReady">loading AI Model... </span>
          <button class="btn btn-primary" type="button" :disabled='!model.isReady' @click="toggleCamera">
            <span v-if="!camera.isOpen">Open Camera</span>
            <span v-else>Close Camera</span>
          </button>
        </div>
      </div>
      <div class="row position-relative text-center">
        <video id="v" autoplay v-bind="camObj" class="position-absolute mw-100"/>
        <canvas id="c" :width="camera.width" :height="camera.height" class="position-absolute mw-100"/>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * Composition API
 * >> 根據邏輯功能來組織程式，一個功能所定義的所有api會放在一起。
 **/
import '@tensorflow/tfjs';
import * as cocoSsd from "@tensorflow-models/coco-ssd";
import "bootstrap/dist/css/bootstrap.min.css";
import { reactive, onMounted } from 'vue';

export default {
  name: 'compositionAPI',
  setup(){
    /**
     * declear
     **/
    let AImodel

    const model = reactive({
      isReady: false,
    });

    const camera = reactive({
      width: "640",
      height: "480",
      camList: [],
      selected: '',
      isOpen : false,
    });

    const camObj = reactive({
      srcObject: null,
    });
    
    /**
     * load AI Model
     **/
    const loadModel = async() => {
      try{
        AImodel = await cocoSsd.load();
        model.isReady = true;
        console.log('model loaded')
      }
      catch(e){
        console.log('loaded model fail')
        console.log(e);
      }
    }
    /**
     * load available cameras
     **/
    const loadCameras = async() => {
      try{
        /* open a generic stream to get permission to see devices; 
          * Mobile Safari insists */
        await navigator.mediaDevices.getUserMedia({audio: false, video: true});
        let deviceInfos = await navigator.mediaDevices.enumerateDevices()
        for (let i = 0; i !== deviceInfos.length; ++i) {
          let deviceInfo = deviceInfos[i];
          if (deviceInfo.kind === "videoinput") {
            camera.camList.push(deviceInfo);
          }
        }
      }catch(e){
        console.log("not supported",e)
      }
    }
    /**
     * get camera
     **/
    const createCameraElement = () => {
      // 1.check
      if (camera.selected === ""){
        alert("selct one device")
        return
      }
      // 2.setting
      let constraints = { video: { deviceId: { exact: camera.selected } } };
      // 3.加載
			navigator.mediaDevices
				.getUserMedia(constraints)
				.then(stream => {
					camObj.srcObject = stream;
          camera.isOpen = true;
				})
				.catch(error => {
          console.log(error)
					alert("May the browser didn't support or there is some errors.");
				});
    }
    /**
     * stop Camera
     **/
    const stopCameraStream = () => {
      let tracks = camObj.srcObject.getTracks();
			tracks.forEach(track => {
				track.stop();
			});
    }
    /**
     * toggle Camera
     **/
    const toggleCamera = () => {
      try{
        if(camera.isOpen) {
          stopCameraStream();
          camera.isOpen = false;
        } else {
          createCameraElement();
        }
      }catch(e){
        console.log(e);
      }
    }
    /**
     * detect Objects
     **/
    const detectObjects = async() => {
      // check
      if (!model.isReady) return
      if (!camera.isOpen) return
      // predict
      const cam = document.getElementById('v');
      let predictions = await AImodel.detect(cam);
      renderPredictions(predictions);
      // go loop
      requestAnimationFrame(()=>{
        detectObjects();
      });
    }
    /**
     * render on canvas
     **/
    const renderPredictions = predictions => {
      // get the context of canvas
      const c = document.getElementById("c");
      const ctx = c.getContext("2d");
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

    onMounted(()=>{
      if(navigator.mediaDevices.getUserMedia
        ||navigator.mediaDevices.webkitGetUserMedia){
        // init
        loadCameras();
        loadModel();

        const cam = document.getElementById("v");
        cam.addEventListener('loadeddata', () => {
            camera.height = cam.videoHeight
            camera.width = cam.videoWidth
            detectObjects();
        })
      }else{
        alert("not supported");
      }
    });

    return{
      model,
      camera,
      camObj,
      toggleCamera,
    }
  }
}
</script>


