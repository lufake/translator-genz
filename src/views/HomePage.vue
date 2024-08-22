<template>
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
      <ion-grid>
        <ion-row class="ion-align-items-center">
          <ion-col size="5">
            <ion-item>
              <ion-select v-model="sourceLanguage" interface="popover" class="custom-language-select">
                <ion-select-option value="de">Deutsch</ion-select-option>
                <ion-select-option value="en">Englisch</ion-select-option>
                <ion-select-option value="fr">Französisch</ion-select-option>
                <ion-select-option value="it">Italienisch</ion-select-option>
                <ion-select-option value="es">Spanisch</ion-select-option>
              </ion-select>
            </ion-item>
          </ion-col>

          <ion-col size="2" class="ion-text-center">
            <ion-icon class="swap-icon" :icon="swapHorizontalOutline" @click="swapLanguages"></ion-icon>
          </ion-col>

          <ion-col size="5">
            <ion-item>
              <ion-select v-model="targetLanguage" interface="popover" class="custom-language-select">
                <ion-select-option value="de">Deutsch</ion-select-option>
                <ion-select-option value="en">Englisch</ion-select-option>
                <ion-select-option value="fr">Französisch</ion-select-option>
                <ion-select-option value="it">Italienisch</ion-select-option>
                <ion-select-option value="es">Spanisch</ion-select-option>
              </ion-select>
            </ion-item>
          </ion-col>
        </ion-row>
      </ion-grid>

      <ion-item>
        <ion-textarea :rows="6" v-model="sourceText" placeholder="Gib hier den zu übersetzenden Text ein..."></ion-textarea>
      </ion-item>

      <ion-button class="rounded-button" expand="block" @click="translateText">Übersetzen</ion-button>

      <ion-card>
        <ion-card-header>
          Übersetzung
        </ion-card-header>
        <ion-card-content>
          {{ translatedText }}
        </ion-card-content>
      </ion-card>

      <ion-grid>
        <ion-row class="ion-justify-content-center">
          <ion-col size="auto">
            <ion-button class="icon-button" @click="copyText">
              <ion-icon :icon="clipboard"></ion-icon>
            </ion-button>
          </ion-col>
          <ion-col size="auto">
            <ion-button class="icon-button" @click="speakText">
              <ion-icon :icon="volumeHigh"></ion-icon>
            </ion-button>
          </ion-col>
        </ion-row>
      </ion-grid>

      <ion-toast :is-open="showToast" :message="toastMessage" :duration="3000"></ion-toast>

      <ion-loading :is-open="loading" message="Downloading language model..."></ion-loading>
    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonCol, IonGrid, IonRow, IonHeader, IonPage, IonTitle, IonToolbar, IonItem, IonLabel, IonSelect, IonSelectOption, IonTextarea, IonButton, IonCard, IonCardHeader, IonCardContent, IonToast, IonLoading, alertController, IonIcon } from '@ionic/vue';
import { ref, onMounted } from 'vue';
import { Network } from '@capacitor/network';
import { Clipboard } from '@capacitor/clipboard';
import { TextToSpeech } from '@capacitor-community/text-to-speech';
import { Translation, Language } from '@capacitor-mlkit/translation';
import axios from 'axios';
import { clipboard, swapHorizontalOutline, volumeHigh } from 'ionicons/icons';

const sourceLanguage = ref(Language.English);
const targetLanguage = ref(Language.German);
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

function swapLanguages() {
  [sourceLanguage.value, targetLanguage.value] = [targetLanguage.value, sourceLanguage.value];
  [sourceText.value, translatedText.value] = [translatedText.value, sourceText.value];
}

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

onMounted(async () => {
  if (translationMethod.value === 'mlkit') {
    loading.value = true;
    await Translation.downloadModel({ language: Language.English });
    loading.value = false;
  }
});
</script>

<style scoped>
.rounded-button {
  --border-radius: 8px;
  --background: #4d4d4d8f;
  width: 50%;
  margin: 20px auto;
}

.swap-icon {
  font-size: 24px;
  color: #000;
  cursor: pointer;
}

.custom-language-select {
  --padding-start: 10px;
  --padding-end: 10px;
  --background: #e0e0e0;
  --border-radius: 8px;
  --placeholder-color: #6d6d6d;
  --text-align: center; /* Center text alignment */
}

.ion-justify-content-center {
  justify-content: center;
}

.icon-button {
  --border-radius: 8px;
  --background: #4d4d4d8f;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 50px;
  height: 50px;
  border-radius: 8px;
  --padding: 0;
}

.icon-button ion-icon {
  font-size: 24px;
  color: #fff;
}
</style>
