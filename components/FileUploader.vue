<script>
import cv from '@techstark/opencv-js/dist/opencv.js';

export default defineNuxtComponent({
  data() {
    return {
      image: null,
    };
  },
  methods: {
    async loadImage(event) {
      const canvas = this.$refs.canvas;
      const context = canvas.getContext('2d');
      const file = event.target.files[0];
      const image = await this.loadImageFromFile(file);
      canvas.width = image.width;
      canvas.height = image.height;
      context.drawImage(image, 0, 0, image.width, image.height);
      const src = cv.imread(canvas);
      const gray = new cv.Mat();
      cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY);

      // Apply adaptive thresholding
      const threshold = new cv.Mat();
      const blockSize = 11; // the size of the neighborhood for thresholding
      const C = 2; // a constant value to be subtracted from the mean or weighted mean
      cv.adaptiveThreshold(
        gray,
        threshold,
        255,
        cv.ADAPTIVE_THRESH_MEAN_C,
        cv.THRESH_BINARY,
        blockSize,
        C
      );

      const canny = new cv.Mat();
      cv.Canny(threshold, canny, 50, 100);
      const contours = new cv.MatVector();
      const hierarchy = new cv.Mat();
      cv.findContours(
        canny,
        contours,
        hierarchy,
        cv.RETR_TREE,
        cv.CHAIN_APPROX_SIMPLE
      );
      const largestContourIndex = this.getLargestContourIndex(contours);
      const approx = new cv.Mat();
      const epsilon =
        0.1 * cv.arcLength(contours.get(largestContourIndex), true);
      cv.approxPolyDP(contours.get(largestContourIndex), approx, epsilon, true);
      cv.drawContours(
        src,
        contours,
        largestContourIndex,
        [255, 0, 0, 255],
        2,
        cv.LINE_8,
        hierarchy,
        0
      );
      cv.imshow(canvas, src);
      src.delete();
      gray.delete();
      threshold.delete();
      canny.delete();
      contours.delete();
      hierarchy.delete();
      approx.delete();
    },
    async loadImageFromFile(file) {
      return new Promise((resolve) => {
        const image = new Image();
        image.onload = () => {
          resolve(image);
        };
        image.src = URL.createObjectURL(file);
      });
    },
    getLargestContourIndex(contours) {
      let largestContourIndex = 0;
      let largestContourArea = 0;
      for (let i = 0; i < contours.size(); i++) {
        const area = cv.contourArea(contours.get(i));
        if (area > largestContourArea) {
          largestContourIndex = i;
          largestContourArea = area;
        }
      }
      return largestContourIndex;
    },
  },
});
</script>

<template>
  <div class="container">
    <div class="image-container">
      <div class="image-wrapper">
        <img class="original-image" ref="originalImage" />
        <div class="overlay"></div>
      </div>
      <div class="image-wrapper">
        <img class="processed-image" ref="processedImage" />
        <div class="overlay"></div>
      </div>
    </div>
    <input type="file" @change="loadImage" />
  </div>
</template>

<style>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
}

.image-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  gap: 50px;
  margin-bottom: 50px;
}

.image-wrapper {
  position: relative;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border: 2px solid red;
  box-sizing: border-box;
  pointer-events: none;
}

.original-image {
  max-width: 500px;
  max-height: 500px;
}

.processed-image {
  max-width: 500px;
  max-height: 500px;
}
</style>
