<template>
    <ion-card>
        <ion-card-header>
            <ion-card-title>Camera</ion-card-title>
        </ion-card-header>

        <ion-card-content>
            <video
                v-if="isCameraOpen"
                ref="videoElement"
                autoplay
                playsinline
                muted
                class="camera-preview"
            ></video>

            <canvas
                ref="canvasElement"
                class="hidden-canvas"
            ></canvas>

            <ion-button
                v-if="!isCameraOpen"
                expand="block"
                @click="openCamera"
            >
                <ion-icon slot="start" :icon="cameraIcon" />
                Open Camera
            </ion-button>

            <ion-button
                v-if="isCameraOpen"
                expand="block"
                @click="takePicture"
            >
                <ion-icon slot="start" :icon="cameraIcon" />
                Take Picture
            </ion-button>

            <ion-button
                v-if="isCameraOpen"
                expand="block"
                fill="outline"
                @click="closeCamera"
            >
                Close Camera
            </ion-button>

            <ion-text v-if="errorMessage" color="danger">
                <p>{{ errorMessage }}</p>
            </ion-text>
        </ion-card-content>
    </ion-card>
</template>

<script setup lang="ts">
import {
    IonButton,
    IonCard,
    IonCardContent,
    IonCardHeader,
    IonCardTitle,
    IonIcon,
    IonText,
} from "@ionic/vue";

import { camera as cameraIcon } from "ionicons/icons";
import { ref, nextTick, onUnmounted } from "vue";

const emit = defineEmits<{
    (event: "photoCaptured", photo: string): void;
}>();

const errorMessage = ref("");
const isCameraOpen = ref(false);

const videoElement = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);

let stream: MediaStream | null = null;

const openCamera = async () => {
    errorMessage.value = "";

    try {
        stream = await navigator.mediaDevices.getUserMedia({
            video: {
                facingMode: "user",
                width: { ideal: 1280 },
                height: { ideal: 720 },
            },
            audio: false,
        });

        isCameraOpen.value = true;

        // Wait for Vue to create the <video> element
        await nextTick();

        if (videoElement.value) {
            videoElement.value.srcObject = stream;

            await videoElement.value.play();
        }
    } catch (error) {
        console.error("Camera error:", error);

        errorMessage.value =
            "Unable to access camera. Please allow camera permission.";
    }
};

const takePicture = () => {
    if (!videoElement.value || !canvasElement.value) {
        return;
    }

    const video = videoElement.value;
    const canvas = canvasElement.value;

    if (video.videoWidth === 0 || video.videoHeight === 0) {
        errorMessage.value = "Camera is not ready yet.";
        return;
    }

    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    const context = canvas.getContext("2d");

    if (!context) {
        errorMessage.value = "Unable to capture photo.";
        return;
    }

    context.drawImage(
        video,
        0,
        0,
        canvas.width,
        canvas.height
    );

    const photo = canvas.toDataURL("image/jpeg", 0.9);

    emit("photoCaptured", photo);
};

const closeCamera = () => {
    if (stream) {
        stream.getTracks().forEach((track) => {
            track.stop();
        });

        stream = null;
    }

    isCameraOpen.value = false;
};

onUnmounted(() => {
    closeCamera();
});
</script>

<style scoped>
.camera-preview {
    display: block;
    width: 100%;
    height: auto;
    min-height: 300px;
    background: black;
    border-radius: 10px;
    object-fit: cover;
}

.hidden-canvas {
    display: none;
}
</style>