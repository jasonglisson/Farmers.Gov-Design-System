<template>
  <div class="container">
    <div
      :id="stepperId"
      class="row simple-stepper pb-12"
      :lang="data.lang_code"
      ref="thisStepper"
    >
      <div class="medium-12">
        <h2 v-if="data.stepper_header">{{ data.stepper_header }}</h2>
        <p v-if="data.stepper_description">{{ data.stepper_description }}</p>
        <div
          v-for="(step, stepIndex) in data.questions"
          v-if="stepIsVisible(stepIndex) || stepIndex === 0"
          :id="`stepper-row-${stepIndex + 1}`"
          :class="elementClasses('input-row', answeredClass(step, stepIndex))"
        >
          <div class="question-number mr-2">
            <div class="number-container">
              <div class="number">{{ stepNumber(stepIndex) }}</div>
            </div>
          </div>
          <div class="question-and-answer pl-4 full-width">
            <h3 class="question mt-6">
              {{ step.question }}
            </h3>
            <div
              v-html="processLinksInContent(step.description)"
              class="stepper-description"
              @click="handleLinkAlert"
            />
            <fieldset
              class="m-0 p-0 no-border"
              v-if="step.type === 'radio'"
              @click="showPrintLink()"
            >
              <legend class="sr-only">{{ step.question }}</legend>
              <RadioButton
                v-for="(option, index) in step.options"
                :key="index"
                :stepIndex="stepIndex"
                :id="formOptionId(step.question, option.value)"
                :name="formOptionName(stepIndex, option)"
                :value="option.value"
                :text="option.text"
                :option="option"
                :checked="optionIsChecked(option.value, stepIndex)"
              />
            </fieldset>
            <Result
              v-if="getResult(stepIndex) && getResult(stepIndex).result"
              :data="getResult(stepIndex).result"
              :printLink="printLink"
              :selectedValue="selectedValue"
            />
          </div>
        </div>
      </div>

      <iframe
        id="print-frame"
        aria-hidden="true"
        title="print_frame"
        name="print_frame"
        tabindex="-1"
        :srcdoc="printCss"
        style="height: 0; width: 0; visiblity: hidden; border: none"
      ></iframe>
    </div>
  </div>
</template>

<script>
import Result from "./Result.vue";
import RadioButton from "./form/RadioButton.vue";
import smoothscroll from "smoothscroll-polyfill";

export default {
  name: "App",
  props: ["data", "options"],
  data() {
    return {
      stepperId: !!this.data.anchor_link_id
        ? this.data.anchor_link_id
        : this.data.element_id,
      visibleSteps: [],
      selected: [],
      results: [],
      printLink: false,
      printCss: "",
      printPopUpPage: this.options.print.printPopupPage,
      selectedValue: "",
      touchDevice: "ontouchstart" in document.documentElement,
    };
  },
  components: {
    Result: Result,
    RadioButton: RadioButton,
  },
  mounted() {
    this.resetStepper();
    let cssMarkup = [];
    if (!!this.options && !!this.options.print) {
      this.printLink = this.options.print.showPrintLink;
      const cssFiles = this.options.print.printCssFiles;
      cssFiles.forEach((file) => {
        cssMarkup.push(
          `<link href="${file}" rel="stylesheet" type="text/css">`
        );
      });
    }
    this.printCss = cssMarkup.join("\r\n");
  },
  methods: {
    stepNumber(stepIndex) {
      return (
        this.visibleSteps.findIndex((row) => row.stepIndex === stepIndex) + 1
      );
    },
    resetStepper() {
      this.visibleSteps = [{ stepIndex: 0 }];
      this.results = [];
      this.selectedValue = "";
    },
    scrollToFirstStep() {
      this.scrollToStep(0);
    },
    stepIsVisible(stepIndex) {
      return this.visibleSteps.filter((row) => row.stepIndex === stepIndex)
        .length;
    },
    optionIsChecked(value, stepIndex) {
      return Boolean(
        this.visibleSteps.findIndex(
          (row) => row.stepIndex === stepIndex && row.answer === value
        ) > -1
      );
    },
    processAnswer(option, stepIndex) {
      this.selectedValue = "";
      // remove questions if their index is higher than the one currently selected.
      this.visibleSteps = this.visibleSteps.filter(
        (row) => row.stepIndex <= stepIndex
      );
      this.results = [];

      if (!!option.value) {
        this.selectedValue = option.value;
        this.setAnswer(stepIndex, option.value);
        if (!option.go_to) this.scrollToStep(stepIndex + 1);
      }
      if (!!option.go_to) {
        let nextStep = option.go_to - 1;
        this.setAnswer(nextStep);
        this.scrollToStep(nextStep);
      }
      if (!!option.result && !option.go_to) {
        this.results = this.results.filter(
          (row) => row.stepIndex !== stepIndex
        );
        this.results.push({ stepIndex: stepIndex, result: option.result });
      }
    },
    setAnswer(index, value) {
      this.results = [];
      this.visibleSteps = this.visibleSteps.filter(
        (row) => row.stepIndex !== index
      );
      this.visibleSteps.push({ stepIndex: index, answer: value });
    },
    getResult(index) {
      return !!this.results
        ? this.results.filter((row) => row.stepIndex === index)[0]
        : [];
    },
    showPrintLink() {
      if (
        !!this.options &&
        !!this.options.print.showPrintLink &&
        this.visibleSteps.length ===
          this.visibleSteps.filter((row) => row.answer).length
      ) {
        this.printLink = true;
      } else {
        this.printLink = false;
      }
    },
    formOptionId(question, value) {
      return this.createFormElementId(
        `${this.webFriendlyName(question).substring(
          0,
          30
        )}-${this.webFriendlyName(value)}`
      );
    },
    formOptionName(stepIndex) {
      return `${this.stepperId}-input-${stepIndex + 1}`;
    },
    answeredClass(step, stepIndex) {
      return this.visibleSteps.filter((row) => row.stepIndex > stepIndex).length
        ? "answered"
        : "";
    },
    scrollToStep(index) {
      if (index !== this.data.questions.length) {
        setTimeout(() => {
          const element = document.querySelector(
            `#${this.stepperId} #stepper-row-${index + 1}`
          );
          if (!!element) element.scrollIntoView({ behavior: "smooth" });
        }, 200);
      }
    },
    printStepper() {
      const content = this.$refs.thisStepper.outerHTML;
      const css = this.printCss;
      if (
        (this.is_safari() && !this.touchDevice) ||
        (this.is_chrome() && this.is_ios())
      ) {
        this.printPopUp(content, css, this.printPopUpPage);
        return;
      }
      this.printFromIframe(content, css);
    },
  },
};
</script>
