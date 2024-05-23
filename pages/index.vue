<template>
  <div class="object-detector-container">
    <button @click="openFilePicker" :disabled="isLoading">{{ isLoading ? 'Recognizing...' : 'Select Image' }}</button>

    <div class="detector-container">

      <img v-if="imgData" :src="imgData" ref="imageRef" alt="Target Image" />
      <div v-if="!isEmptyPredictions" v-for="(prediction, idx) in predictions" :key="idx" class="target-box"
        :style="{ left: prediction.bbox[0] + 'px', top: prediction.bbox[1] + 'px', width: prediction.bbox[2] + 'px', height: prediction.bbox[3] + 'px' }">
        {{ prediction.class }} {{ (prediction.score * 100).toFixed(1) }}%
      </div>
    </div>
    <input type="file" @change="onSelectImage" ref="fileInputRef" style="display: none;" />

  </div>
</template>

<script>
import * as cocoSsd from "@tensorflow-models/coco-ssd";
import "@tensorflow/tfjs-backend-cpu";

export default {
  data() {
    return {
      fileInputRef: null,
      imageRef: null,
      imgData: null,
      predictions: [],
      isLoading: false
    };
  },
  computed: {
    isEmptyPredictions() {
      return !this.predictions || this.predictions.length === 0;
    }
  },
  methods: {
    async openFilePicker() {
      this.$refs.fileInputRef.click();
    },
    async normalizePredictions(predictions, imgSize) {
      if (!predictions || !imgSize || !this.$refs.imageRef) return predictions || [];
      return predictions.map((prediction) => {
        const { bbox } = prediction;
        const oldX = bbox[0];
        const oldY = bbox[1];
        const oldWidth = bbox[2];
        const oldHeight = bbox[3];

        const imgWidth = this.$refs.imageRef.width;
        const imgHeight = this.$refs.imageRef.height;

        const x = (oldX * imgWidth) / imgSize.width;
        const y = (oldY * imgHeight) / imgSize.height;
        const width = (oldWidth * imgWidth) / imgSize.width;
        const height = (oldHeight * imgHeight) / imgSize.height;

        return { ...prediction, bbox: [x, y, width, height] };
      });
    },
    async detectObjectsOnImage(imageElement, imgSize) {
      const model = await cocoSsd.load({});
      const predictions = await model.detect(imageElement, 6);
      const normalizedPredictions = await this.normalizePredictions(predictions, imgSize);
      console.log(predictions)
      
      this.predictions = normalizedPredictions;
    },
    readImage(file) {
      return new Promise((resolve, reject) => {
        const fileReader = new FileReader();
        fileReader.onload = () => resolve(fileReader.result);
        fileReader.onerror = () => reject(fileReader.error);
        fileReader.readAsDataURL(file);
      });
    },
    async onSelectImage(e) {
      this.predictions = [];
      this.isLoading = true;

      const file = e.target.files[0];
      const imgData = await this.readImage(file);
      this.imgData = imgData;

      const imageElement = document.createElement("img");
      imageElement.src = imgData;

      imageElement.onload = async () => {
        const imgSize = {
          width: imageElement.width,
          height: imageElement.height,
        };
        await this.detectObjectsOnImage(imageElement, imgSize);
        this.isLoading = false;
      };
    }
  }
};
</script>

<style scoped>
.object-detector-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.detector-container {
  min-width: 200px;
  height: 700px;
  border: 3px solid #fff;
  border-radius: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.target-box {
  position: absolute;
  border: 4px solid #1ac71a;
  background-color: transparent;
  z-index: 20;
  font-weight: 500;
  font-size: 17px;
  color: #1ac71a;
  top: -1.5em;
  left: -5px;
  padding: 0.5em;
}

input[type="file"] {
  display: none;
}

button {
  padding: 7px 10px;
  border: 2px solid transparent;
  background-color: #fff;
  color: #0a0f22;
  font-size: 16px;
  font-weight: 500;
  outline: none;
  margin-top: 2em;
  cursor: pointer;
  transition: all 260ms ease-in-out;
}

button:hover {
  background-color: transparent;
  border: 2px solid #fff;
  color: #fff;
}
</style>
