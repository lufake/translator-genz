"<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Übersetzer</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-item>
        <ion-label>Translation Method</ion-label>
        <ion-select v-model="translationMethod">
          <ion-select-option value="deepl">DeepL API</ion-select-option>
          <ion-select-option value="mlkit">ML Kit</ion-select-option>
        </ion-select>
      </ion-item>
      
      <ion-item>
        <ion-label position="floating"><strong>Eingabesprache</strong></ion-label>
        <ion-select v-model="sourceLanguage">
          <ion-select-option value="de">Deutsch</ion-select-option>
          <ion-select-option value="en">Englisch</ion-select-option>
          <ion-select-option value="fr">Französisch</ion-select-option>
          <ion-select-option value="it">Italienisch</ion-select-option>
          <ion-select-option value="es">Spanisch</ion-select-option>
        </ion-select>
      </ion-item>

      <ion-item>
        <ion-label position="floating"><strong>Ausgabesprache</strong></ion-label>
        <ion-select v-model="targetLanguage">
          <ion-select-option value="de">Deutsch</ion-select-option>
          <ion-select-option value="en">Englisch</ion-select-option>
          <ion-select-option value="fr">Französisch</ion-select-option>
          <ion-select-option value="it">Italienisch</ion-select-option>
          <ion-select-option value="es">Spanisch</ion-select-option>
        </ion-select>
      </ion-item>

      <ion-item>
        <ion-textarea :rows="6" v-model="sourceText" placeholder="Gib hier den zu übersetzenden Text ein..."></ion-textarea>
      </ion-item>

      <ion-button expand="block" @click="translateText">Übersetzen</ion-button>
      <ion-button expand="block" @click="swapLanguages">Sprachen tauschen</ion-button>

      <ion-card>
        <ion-card-header>
          Übersetzung
        </ion-card-header>
        <ion-card-content>
          {{ translatedText }}
        </ion-card-content>
      </ion-card>

      <ion-button expand="block" @click="copyText">In Zwischenablage kopieren</ion-button>
      <ion-button expand="block" @click="speakText">Vorlesen</ion-button>

      <ion-toast :is-open="showToast" :message="toastMessage" :duration="3000"></ion-toast>

      <ion-loading
        :is-open="loading"
        message="Downloading language model..."
      ></ion-loading>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar, IonItem, IonLabel, IonSelect, IonSelectOption, IonTextarea, IonButton, IonCard, IonCardHeader, IonCardContent, IonToast, IonLoading, alertController } from '@ionic/vue';
import { ref, onMounted } from 'vue';
import { Network } from '@capacitor/network';
import { Clipboard } from '@capacitor/clipboard';
import { TextToSpeech } from '@capacitor-community/text-to-speech';
import { Translation, Language } from '@capacitor-mlkit/translation';
import axios from 'axios';

const sourceLanguage = ref(Language.English); // Using enum from ML Kit
const targetLanguage = ref(Language.German); // Using enum from ML Kit
const sourceText = ref('');
const translatedText = ref('');
const showToast = ref(false);
const toastMessage = ref("");
const translationMethod = ref('deepl');
const loading = ref(false);
const deepLAPIKey = '44c5f6a9-448b-47c1-b128-155b3ade2fbd:fx'

async function translateText() {
  if (translationMethod.value === 'deepl') {
    const params = new URLSearchParams();
    params.append('auth_key', deepLAPIKey);
    params.append('text', sourceText.value);
    params.append('source_lang', sourceLanguage.value);
    params.append('target_lang', targetLanguage.value);

    try {
      const response = await axios.post('https://api-free.deepl.com/v2/translate', params);
      translatedText.value = response.data.translations[0].text;
    } catch (error) {
      console.error('Translation error:', error);
      translatedText.value = 'Übersetzungsfehler';
    }
  } else {
    const networkStatus = await Network.getStatus();
    if (networkStatus.connectionType === 'wifi' || await confirmDownloadOnMobileData()) {
      const modelStatus = await Translation.getDownloadedModels();
      if (!modelStatus.languages.includes(sourceLanguage.value)) {
        await Translation.downloadModel({ language: sourceLanguage.value });
      }
      try {
        const result = await Translation.translate({
          text: sourceText.value,
          sourceLanguage: sourceLanguage.value,
          targetLanguage: targetLanguage.value
        });
        translatedText.value = result.text;
      } catch (error) {
        console.error('ML Kit Translation error:', error);
        translatedText.value = 'Übersetzungsfehler';
      }
    } else {
      console.log('Download cancelled or not on WiFi without user consent.');
    }
  }
}

async function confirmDownloadOnMobileData() {
  const alert = await alertController.create({
    header: 'Download via Mobile Data',
    message: 'Downloading over mobile data may incur charges. Continue?',
    buttons: [
      {
        text: 'Cancel',
        role: 'cancel',
        handler: () => {
          return false;
        }
      }, {
        text: 'Continue',
        handler: () => {
          return true;
        }
      }
    ]
  });

  await alert.present();
  const result = await alert.onDidDismiss();
  return result.role === 'continue';
}

// Utility function to swap source and target languages
function swapLanguages() {
  [sourceLanguage.value, targetLanguage.value] = [targetLanguage.value, sourceLanguage.value];
  [sourceText.value, translatedText.value] = [translatedText.value, sourceText.value];
}

// Function to copy translated text to the clipboard
async function copyText() {
  try {
    await Clipboard.write({
      string: translatedText.value
    });
    toastMessage.value = "Text in die Zwischenablage kopiert";
    showToast.value = true;
    setTimeout(() => showToast.value = false, 3000);
  } catch (err) {
    console.error('Failed to copy text', err);
  }
}

// Function to speak the translated text
async function speakText() {
  try {
    await TextToSpeech.speak({
      text: translatedText.value,
      lang: targetLanguage.value
    });
  } catch (err) {
    console.error('Failed to speak text', err);
  }
}

//This hook preloads the English model if ML Kit is chosen as the translation method
onMounted(async () => {
  if (translationMethod.value === 'mlkit') {
    loading.value = true;
    await Translation.downloadModel({ language: Language.English });
    loading.value = false;
  }
});

</script>