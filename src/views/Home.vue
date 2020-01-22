<template>
  <div class="train">
    <template v-if="mode == 'train'">
      <h1>Take pictures that define your different moods in the dropdown</h1>
    </template>
    <template v-else>
      <h1>Take a picture to let us know how you feel about our service</h1>
    </template>
    <select id="use_case" v-on:change="changeOption()">
      <option value="train">Train</option>
      <option value="test">Test</option>
    </select>
    <Camera></Camera>
    <template v-if="mode == 'train'">
      <select id="emotion_options">
        <template v-for="(emotion, index) in emotions">
          <option :key="index" :value="index">{{ emotion }}</option>
        </template>
      </select>
      <button v-on:click="trainModel()">Train Model</button>
    </template>
    <template v-else>
      <button v-on:click="getEmotion()">Get Emotion</button>
    </template>

    <h1>{{ detected_e }}</h1>
  </div>
</template>

<script>
// @ is an alias to /src
import Camera from "@/components/Camera.vue";
import * as tf from "@tensorflow/tfjs";
import * as mobilenetModule from "@tensorflow-models/mobilenet";
import * as knnClassifier from "@tensorflow-models/knn-classifier";
import axios from "axios";

export default {
  name: "Home",
  components: {
    Camera
  },
  data: function() {
    return {
      emotions: ["angry", "neutral", "happy"],
      classifier: null,
      mobilenet: null,
      class: null,
      detected_e: null,
      mode: "train",
      trainedData: []
    };
  },
  mounted: function() {
    this.init();
  },
  methods: {
    async init() {
      // load the load mobilenet and create a KnnClassifier
      this.classifier = knnClassifier.create();

      const datasetJson = localStorage.getItem("my-data");

      if (datasetJson) {
        const datasetObj = JSON.parse(datasetJson);
        const dataset = this.fromDatasetObject(datasetObj);
        this.classifier.setClassifierDataset(dataset);
      }
      this.mobilenet = await mobilenetModule.load();
    },
    trainModel() {
      let selected = document.getElementById("emotion_options");
      this.class = selected.options[selected.selectedIndex].value;
      this.addExample();
    },
    async toDatasetObject(dataset) {
      const result = await Promise.all(
        Object.entries(dataset).map(async ([classId, value]) => {
          const data = await value.data();

          return {
            classId: Number(classId),
            data: Array.from(data),
            shape: value.shape
          };
        })
      );

      return result;
    },
    fromDatasetObject(datasetObject) {
      return Object.entries(datasetObject).reduce(
        (result, [indexString, { data, shape }]) => {

          console.log('🚧  indexString =>', indexString)

          const tensor = tf.tensor2d(data, shape);
          const index = Number(indexString);

          result[index] = tensor;

          return result;
        },
        {}
      );
    },
    async addExample() {
      const img = tf.browser.fromPixels(this.$children[0].webcam.webcamElement);
      const logits = this.mobilenet.infer(img, "conv_preds");
      this.classifier.addExample(logits, parseInt(this.class));

      const dataset = this.classifier.getClassifierDataset();
      const datasetObj = await this.toDatasetObject(dataset);
      const jsonStr = JSON.stringify(datasetObj);

      localStorage.setItem("my-data", jsonStr);
    },
    async getEmotion() {
      const img = tf.browser.fromPixels(this.$children[0].webcam.webcamElement);
      const logits = this.mobilenet.infer(img, "conv_preds");
      const pred = await this.classifier.predictClass(logits);
      console.log(pred);
      this.detected_e = this.emotions[pred.label];
      this.registerEmotion();
    },
    changeOption() {
      const selected = document.getElementById("use_case");
      this.mode = selected.options[selected.selectedIndex].value;
    },
    registerEmotion() {
      axios
        .post("http://localhost:3128/callback", {
          emotion: this.detected_e
        })
        .then(() => {
          alert("Thanks for letting us know how you feel");
        });
    }
  }
};
</script>
