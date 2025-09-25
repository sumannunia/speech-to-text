<template>
  <v-card class="mx-auto" max-width="500" min-width="500">
    <v-card-title>Voice Transcription</v-card-title>
    <v-card-text>
      <v-container>
        <v-row justify="center">
          <v-col cols="auto">
            <v-btn
              icon
              size="x-large"
              :color="isListening ? 'red' : 'primary'"
              @click="toggleSpeechRecognition"
              :disabled="!isApiSupported"
            >
              <v-icon size="large">{{
                isListening ? 'mdi-microphone-off' : 'mdi-microphone'
              }}</v-icon>
            </v-btn>
          </v-col>
        </v-row>
        <v-row justify="center" class="mt-2">
          <v-col cols="auto">
            <p v-if="status" class="text-subtitle-1 text-center">
              {{ status }}
            </p>
          </v-col>
        </v-row>

        <v-row>
          <v-col>
            <v-textarea
              v-model="transcribedText"
              label="Transcription"
              variant="outlined"
              rows="5"
              auto-grow
              clearable
              @click:clear="clearTranscription"
            ></v-textarea>
          </v-col>
        </v-row>
      </v-container>

      <v-snackbar
        v-model="snackbar.show"
        :color="snackbar.color"
        :timeout="5000"
        top
      >
        {{ snackbar.text }}
        <template v-slot:actions>
          <v-btn text @click="snackbar.show = false"> Close </v-btn>
        </template>
      </v-snackbar>
    </v-card-text>
  </v-card>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const isListening = ref(false);
const isApiSupported = ref(true);
const transcribedText = ref('');
const status = ref('');
let recognition = null;
let finalTranscript = ''; // Stores the final transcript to build upon

// New state for the snackbar
const snackbar = ref({
  show: false,
  text: '',
  color: 'error',
});

const initSpeechRecognition = () => {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    isApiSupported.value = false;
    snackbar.value = {
      show: true,
      text: 'Speech Recognition is not supported by your browser.',
      color: 'error',
    };
    return;
  }

  recognition = new SpeechRecognition();
  recognition.lang = 'en-US';
  recognition.interimResults = true;
  recognition.continuous = true;

  recognition.onstart = () => {
    isListening.value = true;
    status.value = 'Listening...';
  };

  recognition.onresult = (event) => {
    let interimTranscript = '';
    for (let i = event.resultIndex; i < event.results.length; ++i) {
      if (event.results[i].isFinal) {
        finalTranscript += event.results[i][0].transcript;
      } else {
        interimTranscript += event.results[i][0].transcript;
      }
    }
    transcribedText.value = finalTranscript + interimTranscript;
  };

  recognition.onend = () => {
    isListening.value = false;
    status.value = 'Click the microphone to start transcribing.';
  };

  recognition.onerror = (event) => {
    console.error('Speech recognition error:', event.error);
    isListening.value = false;
    // Show the error in the snackbar for better visibility
    snackbar.value = {
      show: true,
      text: getFriendlyErrorMessage(event.error),
      color: 'error',
    };
  };
};

const getFriendlyErrorMessage = (error) => {
  switch (error) {
    case 'not-allowed':
      return 'Microphone access denied. Please allow microphone access.';
    case 'network':
      return 'Network error: Transcription failed. Please check your connection.';
    case 'no-speech':
      return 'No speech was detected. Please try again.';
    default:
      return `An error occurred: ${error}`;
  }
};

const toggleSpeechRecognition = () => {
  if (!recognition) return;
  if (isListening.value) {
    recognition.stop();
  } else {
    // Add a space if we are appending to existing text
    if (transcribedText.value) {
      finalTranscript = transcribedText.value + ' ';
    }
    recognition.start();
  }
};

// Function to handle clearing the text
const clearTranscription = () => {
  transcribedText.value = '';
  finalTranscript = '';
};

onMounted(() => {
  initSpeechRecognition();
  if (isApiSupported.value) {
    status.value = 'Click the microphone to start transcribing.';
  }
});

onUnmounted(() => {
  if (recognition) {
    recognition.stop();
  }
});
</script>