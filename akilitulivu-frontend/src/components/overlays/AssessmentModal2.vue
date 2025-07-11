<template>
  <div class="modal-overlay">
    <transition name="modal-grow" mode="out-in">
      <div class="modal">
        <h2>{{ currentTitle }}</h2>

        <!-- Step 0: Choose Assessment Type -->
        <div v-if="step === -1" class="intro-form">
          <p class="guide">{{ $t('select_assessment_type') }}</p>
          <button @click="selectAssessment('phq9')">{{ $t('depression_assessment') }}</button>
          <button @click="selectAssessment('gad7')">{{ $t('anxiety_assessment') }}</button>
          <button @click="selectAssessment('sdq')">{{ $t('assess_child') }}</button>
        </div>

        <!-- Step 1: User Info -->
        <div v-else-if="step === 0" class="intro-form">

          {{ formData.userType === 'child' ? $t('select_age_group_child') : $t('select_age_group') }}
          <select v-model="formData.ageGroup">
            <option disabled value="">{{ $t('choose_option') }}</option>
            <option value="5-10">5-10</option>
            <option value="11-15">11-15</option>
            <option value="16-20">16-20</option>
            <option value="21-25">21-25</option>
            <option value="26-30">26-30</option>
            <option value="31-35">31-35</option>
            <option value="36-40">36-40</option>
            <option value="41+">40+</option>
          </select>

          {{ formData.userType === 'child' ? $t('select_sex_child') : $t('select_sex') }}
          <select v-model="formData.sex">
            <option disabled value="">{{ $t('choose_option') }}</option>
            <option value="male">{{ $t('male') }}</option>
            <option value="female">{{ $t('female') }}</option>
          </select>

          <button :disabled="!formComplete" @click="step = 1">{{ $t('start_assessment') }}</button>
        </div>

        <!-- Step 2: Assessment Questions -->
        <div v-else-if="step <= questions.length">
          <div class="progress-wrapper">
            <div class="progress-text">{{ step }} / {{ questions.length }}</div>
            <div class="progress-bar">
              <div class="progress-fill" :style="{ width: (step / questions.length * 100) + '%' }"></div>
            </div>
          </div>

          <transition name="fade">
            <div class="question" v-if="step <= questions.length">
              <p class="guide">{{ $t('assessment_guide') }}</p>
              <p>{{ questions[step - 1].text }}</p>
              <div class="options">
                <button v-for="(option, index) in questions[step - 1].options" :key="index" @click="select(option.score)">
                  {{ option.text }}
                </button>
              </div>
            </div>
          </transition>
        </div>

        <!-- Step 3: Results -->
        <div class="result" v-if="step > questions.length">
          <p class="score">{{ $t('your_score') }}: {{ totalScore }} <i><b>{{ $t('out_of') }}</b></i> {{ maxScore }}</p>

          <div v-if="loading" class="loading">
            ⏳ <span class="dots">{{ $t('janja_typing') }}</span>
          </div>

          <div v-else>
            <div v-if="aiMessageHTML" class="ai-response" v-html="aiMessageHTML"></div>

            <div v-if="redirectLink">
              <router-link :to="redirectLink">
                <button class="contact-btn">{{ $t('followup_support') }}</button>
              </router-link>
            </div>

            <div v-else>
              <p class="ai-response">{{ $t('use_akili') }}</p>
            </div>
          </div>
        </div>

        <!-- Close Button -->
        <div class="actions">
          <button :disabled="loading" @click="close">
            <span v-if="loading" class="spinner"></span>
            {{ $t('close') }}
          </button>
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { marked } from "marked";

export default {
  emits: ["close"],
  data() {
    return {
      step: -1,
      selectedTest: null,
      totalScore: 0,
      loading: false,
      aiMessage: "",
      aiMessageHTML: "",
      redirectLink: null,
      scores: [],
      questions: [],
      formData: {
        userType: "",
        ageGroup: "",
        sex: ""
      }
    };
  },
  computed: {
    formComplete() {
      return this.formData.userType && this.formData.ageGroup && this.formData.sex;
    },
    maxScore() {
      return this.questions.length * 3;
    },
    currentTitle() {
      switch (this.selectedTest) {
        case 'phq9': return this.$t('PHQ9_title');
        case 'gad7': return this.$t('GAD7_title');
        case 'sdq': return this.$t('Child_title');
        default: return this.$t('choose_assessment');
      }
    }
  },
  methods: {
    opts() {
      return [
        { text: this.$t('PHQ-Option_1'), score: 0 },
        { text: this.$t('PHQ-Option_2'), score: 1 },
        { text: this.$t('PHQ-Option_3'), score: 2 },
        { text: this.$t('PHQ-Option_4'), score: 3 },
      ];
    },
    selectAssessment(type) {
      this.selectedTest = type;
      this.questions = Array.from({ length: type === 'phq9' ? 9 : 7 }, (_, i) => ({
        text: this.$t(`${type.toUpperCase()}_${i + 1}`),
        options: this.opts()
      }));
      if (type === "phq9" || type === "gad7") {
        this.formData.userType = 'self';
      } else {
        this.formData.userType = 'child';
      }
      this.step = 0;
    },
    async select(score) {
      this.scores.push(score);
      this.totalScore += score;
      this.step++;

      if (this.step > this.questions.length) {
        await this.getAIResponse();
      }
    },
    async getAIResponse() {
      this.loading = true;
      try {
        const lang_sample = this.questions[0].text;
        const res = await fetch(`http://localhost:8000/api/assessment/${this.selectedTest}/`, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            scores: this.scores,
            lang_text: lang_sample,
            user_type: this.formData.userType,
            age_group: this.formData.ageGroup,
            sex: this.formData.sex,
          })
        });

        const data = await res.json();
        this.aiMessage = data.response || "*Could not retrieve AI insight.*";
        this.aiMessageHTML = marked.parse(this.aiMessage);
        this.redirectLink = data.redirect_link || null;

      } catch (err) {
        this.aiMessage = "*Network issue. Try again later.*";
        this.aiMessageHTML = marked.parse(this.aiMessage);
      } finally {
        this.loading = false;
      }
    },
    close() {
      if (!this.loading) this.$emit("close");
    }
  }
};
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal {
  background: linear-gradient(to bottom right, #ffffff, #ffe6e6);
  border-radius: 20px;
  padding: 30px;
  width: 90%;
  max-width: 500px;
  box-shadow: 0 10px 20px rgba(0,0,0,0.15);
  animation: scaleIn 0.4s ease;
}
h2 {
  margin-top: 0;
  color: var(--primary-red);
  text-align: center;
}
b {
  color: black;
  font-size: 14px;
}
.progress-wrapper {
  margin-bottom: 1.5rem;
}
.progress-text {
  text-align: center;
  font-weight: bold;
}
.progress-bar {
  background-color: #f1c6c6;
  border-radius: 10px;
  height: 12px;
  overflow: hidden;
}
.progress-fill {
  height: 100%;
  background-color: var(--primary-red);
  transition: width 0.3s ease;
}
.question p {
  font-size: 17px;
  font-weight: 600;
  margin-bottom: 1rem;
}
.question .guide {
  margin: 3px 0;
  padding: 3px 6px;
  border-left: 4px solid var(--primary-red);
  background: #fff0f0;
  font-style: italic;
  font-size: 14px;
  color: #555;
  border-radius: 8px;
}
.options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.options button {
  background-color: #fff0f0;
  border: 1px solid var(--primary-red);
  padding: 10px 15px;
  border-radius: 12px;
  cursor: pointer;
}
.options button:hover {
  background-color: var(--primary-red);
  color: white;
}
.result p {
  font-size: 1rem;
  text-align: center;
}
.score {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--primary-red);
  margin-bottom: 1rem;
}
.recomendations {
  margin: 6px auto;
  padding: 10px 15px;
  font-size: 15px;
  border-radius: 10px;
  width: 90%;
  text-align: center;
}
.good {
  background-color: #d6f5d6;
  border-left: 5px solid green;
}
.average {
  background-color: #fff8e1;
  border-left: 5px solid orange;
}
.bad {
  background-color: #ffe6e6;
  border-left: 5px solid red;
}
.loading {
  text-align: center;
  font-style: italic;
  color: #555;
  margin-top: 15px;
  font-size: 15px;
}
.dots::after {
  content: '...';
  animation: dotFlash 1.5s steps(3, end) infinite;
}
@keyframes dotFlash {
  0% { content: ''; }
  33% { content: '.'; }
  66% { content: '..'; }
  100% { content: '...'; }
}
.spinner {
  display: inline-block;
  width: 16px;
  height: 16px;
  margin-right: 6px;
  border: 2px solid rgba(255, 255, 255, 0.4);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  vertical-align: middle;
}
@keyframes spin {
  to { transform: rotate(360deg); }
}
.ai-response {
  text-align: center;
  margin-top: 15px;
  padding: 10px;
  border-left: 4px solid #aaa;
  
  background: #f9f9f9;
  border-radius: 8px;
}
.actions {
  text-align: center;
  margin-top: 20px;
}
.actions button {
  padding: 10px 20px;
  background-color: var(--primary-red);
  color: white;
  border: none;
  border-radius: 10px;
  cursor: pointer;
}
.modal-grow-enter-active,
.modal-grow-leave-active {
  transition: all 0.9s ease;
}
.modal-grow-enter-from,
.modal-grow-leave-to {
  opacity: 0;
  transform: scale(0.98);
}
.actions button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
@keyframes scaleIn {
  from { transform: scale(0.9); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}
.intro-form {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-bottom: 20px;
}

.intro-form select {
  padding: 10px;
  border-radius: 10px;
  border: 1px solid #ccc;
  font-size: 16px;
  background: #fff;
  outline-color: var(--primary-red);
}

.intro-form button {
  margin-top: 15px;
  padding: 12px;
  background-color: var(--primary-red);
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s ease;
}

.intro-form button:hover {
  background-color: #d9534f;
}

.intro-form button:disabled {
  background-color: #f4bfbf;
  cursor: not-allowed;
  opacity: 0.7;
}

.contact-btn {
  margin-top: 15px;
  padding: 12px 20px;
  background-color: #2c3e50;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s ease;
}

.contact-btn:hover {
  background-color: #1a252f;
}

</style>
