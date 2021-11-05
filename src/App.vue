<template>
<div id="app">
  <h3 v-if="!isVideoStreamReady && !initFailMessage">Initializing webcam stream ...</h3>
  <h3 v-if="!isModelReady && !initFailMessage">loading model ...</h3>
  <h3 v-if="initFailMessage">Failed to init stream and/or model - {{ initFailMessage }}</h3>

  <div class="resultFrame">
    <video ref="video" autoplay></video>
    <canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
  </div>

  <select v-model="modelConfig.base" @change="loadModelAndStartDetecting">
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
import '@tensorflow/tfjs-backend-cpu'
import '@tensorflow/tfjs-backend-webgl'
import * as cocoSsd from '@tensorflow-models/coco-ssd'

export default {
  name: 'app',

  data () {
    return {
      // store the promises of initializations
      streamPromise: null,
      modelPromise: null,

      // control the UI visibilities
      isVideoStreamReady: false,
      isModelReady: false,
      initFailMessage: '',

      // tfjs model related
      model: null,
      modelConfig: {
        base: 'lite_mobilenet_v2'
      },
      selectableModels: ['lite_mobilenet_v2', 'mobilenet_v1', 'mobilenet_v2'],

      videoRatio: 1,
      resultWidth: 0,
      resultHeight: 0
    }
  },

  methods: {
    initWebcamStream () {
      // if the browser supports mediaDevices.getUserMedia API
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        return navigator.mediaDevices.getUserMedia({
          audio: false, // don't capture audio
          video: { facingMode: 'environment' } // use the rear camera if there is
        })
          .then(stream => {
            // set <video> source as the webcam input
            const video = this.$refs.video
            try {
              video.srcObject = stream
            } catch (error) {
              // support older browsers
              video.src = URL.createObjectURL(stream)
            }

            /*
              model.detect uses tf.fromPixels to create tensors.
              tf.fromPixels api will get the <video> size from the width and height attributes,
                which means <video> width and height attributes needs to be set before called model.detect

              To make the <video> responsive, I get the initial video ratio when it's loaded (onloadedmetadata)
              Then addEventListener on resize, which will adjust the size but remain the ratio
              At last, resolve the Promise.
            */
            return new Promise((resolve, reject) => {
              // when video is loaded
              video.onloadedmetadata = () => {
                // calculate the video ratio
                this.videoRatio = video.offsetHeight / video.offsetWidth
                // add event listener on resize to reset the <video> and <canvas> sizes
                window.addEventListener('resize', this.setResultSize)
                // set the initial size
                this.setResultSize()

                this.isVideoStreamReady = true
                console.log('webcam stream initialized')
                resolve()
              }
            })
          })
          .catch(error => {
            console.log('failed to initialize webcam stream', error)
            throw (error)
          })
      } else {
        return Promise.reject(new Error('Your browser does not support mediaDevices.getUserMedia API'))
      }
    },

    setResultSize () {
      // get the current browser window size
      const clientWidth = document.documentElement.clientWidth

      // set max width as 600
      this.resultWidth = Math.min(600, clientWidth)
      // set the height according to the video ratio
      this.resultHeight = this.resultWidth * this.videoRatio

      // set <video> width and height
      /*
        Doesn't use vue binding :width and :height,
          because the initial value of resultWidth and resultHeight
          will affect the ratio got from the initWebcamStream()
      */
      const video = this.$refs.video
      video.width = this.resultWidth
      video.height = this.resultHeight
    },

    loadModel () {
      this.isModelReady = false
      // if model already exists => dispose it and load a new one
      if (this.model) this.model.dispose()
      // load model with the baseModel
      return cocoSsd.load(this.modelConfig)
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

      const predictions = await this.model.detect(this.$refs.video)
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
          prediction.bbox[1] + 20)
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
body {
  margin: 0;
}

.resultFrame {
  display: grid;

  video {
    grid-area: 1 / 1 / 2 / 2;
  }
  canvas {
    grid-area: 1 / 1 / 2 / 2;
  }
}
</style>
