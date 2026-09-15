<template>
  <div class="gallery-container">
    <h2 v-if="photos.length > 0" class="gallery-title">Captured Images</h2>
    
    <!-- Empty State Graphic -->
    <div v-if="photos.length === 0" class="empty-gallery">
      <ion-icon :icon="imagesIcon" class="empty-icon" />
      <p>No photos captured yet. Turn on the camera to start your gallery collection.</p>
    </div>

    <!-- Responsive Layout Display Grid -->
    <ion-grid v-else>
      <ion-row>
        <ion-col 
          size="6" 
          size-md="4"
          size-lg="3"
          v-for="(photo, index) in photos" 
          :key="index"
        >
          <div class="photo-wrapper">
            <img :src="photo" alt="Gallery Capture Item" class="gallery-img" />
          </div>
        </ion-col>
      </ion-row>
    </ion-grid>
  </div>
</template>

<script setup lang="ts">
import { IonGrid, IonRow, IonCol, IonIcon } from "@ionic/vue";
import { images as imagesIcon } from "ionicons/icons";

defineProps<{
  photos: string[];
}>();
</script>

<style scoped>
.gallery-container {
  margin-top: 16px;
}

.gallery-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: var(--ion-color-dark);
  margin-left: 8px;
  margin-bottom: 12px;
}

.empty-gallery {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  text-align: center;
  color: var(--ion-color-step-400, #8c8c8c);
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 12px;
  opacity: 0.5;
}

.empty-gallery p {
  font-size: 0.95rem;
  max-width: 260px;
  line-height: 1.4;
}

/* Image Item Layout Frames */
.photo-wrapper {
  position: relative;
  width: 100%;
  aspect-ratio: 1 / 1;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
  background-color: var(--ion-color-step-100, #f4f5f8);
  transition: transform 0.2s ease;
}

.photo-wrapper:active {
  transform: scale(0.96);
}

.gallery-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
</style>
