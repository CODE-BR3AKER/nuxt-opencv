<template>
  <div class="container">
    <div class="image-container">
      <div class="image-wrapper">
        <h1>Original</h1>
        <img class="original-image" ref="originalImage" />
        <div class="overlay"></div>
      </div>
      <div class="image-wrapper">
        <h1>Scanned</h1>
        <canvas class="processed-image" ref="canvas"></canvas>
        <div class="overlay"></div>
      </div>
    </div>
    <input type="file" @change="loadImage" />
  </div>
</template>

<script>
export default defineNuxtComponent({
  head() {
    return {
      script: [
        {
          src: 'https://docs.opencv.org/4.x/opencv.js',
        },
      ],
    };
  },
  data() {
    return {
      imageLoaded: false,
      originalImage: null,
      processedImage: '',
      cvLoaded: false, // new variable to track OpenCV library load status
    };
  },
  methods: {
    loadImage(event) {
      const file = event.target.files[0];
      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = () => {
        this.originalImage = this.$refs.originalImage;
        this.originalImage.src = reader.result;
        this.originalImage.onload = () => {
          this.$refs.canvas.width = this.originalImage.width;
          this.$refs.canvas.height = this.originalImage.height;
          const context = this.$refs.canvas.getContext('2d');
          context.drawImage(this.originalImage, 0, 0);
          this.imageLoaded = true;
          this.processImage();
        };
      };
    },
    processImage() {
      if (!this.imageLoaded) return;

      const canvas = this.$refs.canvas;
      const context = canvas.getContext('2d');
      context.drawImage(this.$refs.originalImage, 0, 0);
      const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
      // Convert image to grayscale
      const gray = new cv.Mat(canvas.height, canvas.width, cv.CV_8UC4);
      const data = new Uint8Array(imageData.data.buffer);
      gray.data.set(data);

      cv.cvtColor(gray, gray, cv.COLOR_RGBA2GRAY);

      // Apply Gaussian blur
      const blurred = new cv.Mat();
      cv.GaussianBlur(gray, blurred, new cv.Size(5, 5), 0);

      // Apply Canny edge detection
      const edges = new cv.Mat();
      cv.Canny(blurred, edges, 75, 200);

      // Find contours in the image
      const contours = new cv.MatVector();
      const hierarchy = new cv.Mat();
      cv.findContours(
        edges,
        contours,
        hierarchy,
        cv.RETR_LIST,
        cv.CHAIN_APPROX_SIMPLE
      );

      // Find the largest contour that meets certain criteria (minimum area)
      let largestContour = null;
      let largestContourIdx = -1;
      for (let i = 0; i < contours.size(); i++) {
        const contour = contours.get(i);
        const area = cv.contourArea(contour);
        if (area > 5000) {
          if (largestContour == null || area > cv.contourArea(largestContour)) {
            largestContour = contour;
            largestContourIdx = i;
          }
        }
      }

      if (largestContourIdx >= 0 && largestContourIdx < contours.size()) {
        // Get the corners of the paper by applying a perspective transform
        largestContour = contours.get(largestContourIdx);
        const rect = cv.boundingRect(largestContour);
        const src = new cv.Mat(4, 1, cv.CV_32FC2);
        const dst = new cv.Mat(4, 1, cv.CV_32FC2);
        src.data32F[0] = rect.x;
        src.data32F[1] = rect.y;
        src.data32F[2] = rect.x + rect.width;
        src.data32F[3] = rect.y;
        src.data32F[4] = rect.x;
        src.data32F[5] = rect.y + rect.height;
        src.data32F[6] = rect.x + rect.width;
        src.data32F[7] = rect.y + rect.height;
        dst.data32F[0] = 0;
        dst.data32F[1] = 0;
        dst.data32F[2] = this.scannedImageWidth;
        dst.data32F[3] = 0;
        dst.data32F[4] = 0;
        dst.data32F[5] = this.scannedImageHeight;
        dst.data32F[6] = this.scannedImageWidth;
        dst.data32F[7] = this.scannedImageHeight;

        const M = cv.getPerspectiveTransform(src, dst);
        const warped = new cv.Mat();
        cv.warpPerspective(
          gray,
          warped,
          M,
          new cv.Size(this.scannedImageWidth, this.scannedImageHeight)
        );

        // Set the scanned image data
        const scannedData = new Uint8ClampedArray(
          warped.data.byteLength * (4 / 1)
        );
        const scannedImageData = new ImageData(
          scannedData,
          this.scannedImageWidth,
          this.scannedImageHeight
        );
        const scannedDataU8 = new Uint8Array(scannedData.buffer);
        for (let i = 0; i < this.scannedImageHeight; i++) {
          for (let j = 0; j < this.scannedImageWidth; j++) {
            const index = i * this.scannedImageWidth + j;
            const value = warped.ucharAt(i, j);
            scannedDataU8[index * 4] = value;
            scannedDataU8[index * 4 + 1] = value;
            scannedDataU8[index * 4 + 2] = value;
            scannedDataU8[index * 4 + 3] = 255;
          }
        }
        this.scannedImageData = scannedImageData;
      }
      gray.delete();
      blurred.delete();
      edges.delete();
      contours.delete();
      hierarchy.delete();
    },
  },
});
</script>
<style>
.processed-image,
.original-image {
  width: 300px;
}
</style>
