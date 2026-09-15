<template>
  <ion-page>
    <ion-header>
      <ion-toolbar> 
        <ion-title>{{ isCameraActive ? 'Capture Image' : 'My Photo Gallery' }}</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content class="ion-padding">
      
      <!-- Camera management block segment -->
      <CameraComponent 
        @photoCaptured="addPhoto" 
        @cameraStatusChange="updateCameraStatus"
      />
      
      <!-- Section Toggle: Only render the gallery component when camera visibility is false -->
      <PhotoGalleryComponent 
        v-if="!isCameraActive" 
        :photos="photos" 
      />

    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
} from "@ionic/vue";
import { ref } from "vue";
import CameraComponent from "@/components/CameraComponent.vue";
import PhotoGalleryComponent from "@/components/PhotoGalleryComponent.vue";

const photos = ref<string[]>([]);
const isCameraActive = ref(false);

const addPhoto = (photo: string) => {
  photos.value.unshift(photo);
};

// Track section toggle changes from camera component child triggers
const updateCameraStatus = (isOpen: boolean) => {
  isCameraActive.value = isOpen;
};
</script>

<style scoped>
/* Scoped styles remain clean since structure handles conditional layout displays */
</style>
