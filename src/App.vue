<template>
<div id="app">
  <h3 v-if="!isVideoStreamReady && !initFailMessage">Initializing webcam stream ...</h3>
  <h3 v-if="!isModelReady && !initFailMessage">loading model ...</h3>
  <h3 v-if="initFailMessage">Failed to init stream and/or model - {{ initFailMessage }}</h3>

  <div class="resultFrame">
    <video ref="video" width="600" height="400" autoplay></video>
    <canvas ref="canvas" width="600" height="400"></canvas>
  </div>

  <select v-model="baseModel" @change="loadModelAndStartDetecting">
    <option
      v-for="modelName in selectableModels"
      :key="modelName"
      :value="modelName">
        {{ modelName }}
    </option>
  </select>
</div>
</template>

<script>
import * as cocoSsd from '@tensorflow-models/coco-ssd'

export default {
  name: 'app',

  data () {
    return {
      // store the promises of initialization
      streamPromise: null,
      modelPromise: null,

      // control the UI visibilities
      isVideoStreamReady: false,
      isModelReady: false,
      initFailMessage: '',

      // tfjs model related
      model: null,
      baseModel: 'lite_mobilenet_v2',
      selectableModels: ['lite_mobilenet_v2', 'mobilenet_v1', 'mobilenet_v2']
    }
  },

  methods: {
    initWebcamStream () {
      // if the browser supports mediaDevices.getUserMedia API
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices.getUserMedia({
          audio: false, // don't capture audio
          video: true
        })
          .then(stream => {
            // set <video> source as the webcam input
            let video = this.$refs.video
            try {
              video.srcObject = stream
            } catch (error) {
              // support older browsers
              video.src = URL.createObjectURL(stream)
            }
            this.isVideoStreamReady = true
            console.log('webcam stream initialized')
          })
          .catch(error => {
            console.log('failed to initialize webcam stream', error)
            throw (error)
          })
      } else {
        return Promise.reject(new Error('Your browser does not support mediaDevices.getUserMedia API'))
      }
    },

    loadModel () {
      this.isModelReady = false
      // if model already exists => dispose it and load a new one
      if (this.model) this.model.dispose()
      // load model with the baseModel
      return cocoSsd.load(this.baseModel)
        .then(model => {
          this.model = model
          this.isModelReady = true
          console.log('model loaded')
        })
        .catch((error) => {
          console.log('failed to load the model', error)
          throw (error)
        })
    },

    async detectObjects () {
      if (!this.isModelReady) return

      let predictions = await this.model.detect(this.$refs.video)
      this.renderPredictions(predictions)
      requestAnimationFrame(() => {
        this.detectObjects()
      })
    },

    loadModelAndStartDetecting () {
      this.modelPromise = this.loadModel()
      // wait for both stream and model promise finished
      // => start detecting objects
      Promise.all([this.streamPromise, this.modelPromise])
        .then(() => {
          this.detectObjects()
        }).catch((error) => {
          console.log('Failed to init stream and/or model')
          this.initFailMessage = error
        })
    },

    renderPredictions (predictions) {
      // get the context of canvas
      const ctx = this.$refs.canvas.getContext('2d')
      // clear the canvas
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)
      predictions.forEach(prediction => {
        ctx.beginPath()
        ctx.rect(...prediction.bbox)
        ctx.lineWidth = 3
        ctx.strokeStyle = 'red'
        ctx.fillStyle = 'red'
        ctx.stroke()
        ctx.shadowColor = 'white'
        ctx.shadowBlur = 10
        ctx.font = '24px Arial bold'
        ctx.fillText(
          `${(prediction.score * 100).toFixed(1)}% ${prediction.class}`,
          prediction.bbox[0],
          prediction.bbox[1] > 10 ? prediction.bbox[1] - 5 : 10)
      })
    }
  },

  mounted () {
    this.streamPromise = this.initWebcamStream()
    this.loadModelAndStartDetecting()
  }
}
</script>

<style lang="scss">
.resultFrame {
  position: relative;

  canvas {
    position: absolute;
    top: 0;
    left: 0
  }
}
</style>
