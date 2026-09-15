<template>
    <ion-card>
        <ion-card-header>
            <ion-card-title>Camera</ion-card-title>
        </ion-card-header>

        <ion-card-content>
            <!-- Viewfinder container wraps the video stream and overlays -->
            <div v-if="isCameraOpen" class="camera-viewport">
                <video
                    ref="videoElement"
                    autoplay
                    playsinline
                    muted
                    class="camera-preview"
                ></video>
                
                <!-- Camera Look HUD Overlays -->
                <div class="hud-layer">
                    <div class="hud-corners"></div>
                    <div class="hud-rec-indicator">
                        <span class="red-dot"></span>
                        <span class="rec-text">LIVE</span>
                    </div>
                    <div class="hud-crosshair">
                        <div class="cross-x"></div>
                        <div class="cross-y"></div>
                    </div>
                </div>
            </div>

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
import { Camera } from "@capacitor/camera";

const emit = defineEmits<{
    (event: "photoCaptured", photo: string): void;
}>();

const errorMessage = ref("");
const isCameraOpen = ref(false);

const videoElement = ref<HTMLVideoElement | null>(null);
const canvasElement = ref<HTMLCanvasElement | null>(null);

let stream: MediaStream | null = null;

const requestNativePermissions = async (): Promise<boolean> => {
    try {
        const status = await Camera.checkPermissions();
        if (status.camera !== "granted") {
            const request = await Camera.requestPermissions({ permissions: ["camera"] });
            return request.camera === "granted";
        }
        return true;
    } catch (error) {
        console.warn("Capacitor plugin fallback:", error);
        return true;
    }
};

const openCamera = async () => {
    errorMessage.value = "";

    const hasPermission = await requestNativePermissions();
    if (!hasPermission) {
        errorMessage.value = "Camera permission denied by user.";
        return;
    }

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

        await nextTick();

        if (videoElement.value) {
            videoElement.value.srcObject = stream;
            await videoElement.value.play();
        }
    } catch (error) {
        console.error("Camera error:", error);
        errorMessage.value = "Unable to access camera. Please allow camera permission.";
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

    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    const photo = canvas.toDataURL("image/jpeg", 0.9);
    emit("photoCaptured", photo);
};

const closeCamera = () => {
    if (stream) {
        stream.getTracks().forEach((track) => track.stop());
        stream = null;
    }
    isCameraOpen.value = false;
};

onUnmounted(() => {
    closeCamera();
});
</script>

<style scoped>
/* Viewport positioning layout */
.camera-viewport {
    position: relative;
    width: 100%;
    border-radius: 10px;
    overflow: hidden;
    margin-bottom: 12px;
    background: #000;
}

.camera-preview {
    display: block;
    width: 100%;
    height: auto;
    min-height: 300px;
    object-fit: cover;
}

.hidden-canvas {
    display: none;
}

/* HUD Overlay Styles */
.hud-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* Allows click actions to bypass graphic layers */
    box-sizing: border-box;
    padding: 15px;
}

/* Border frame bracket framing */
.hud-corners {
    position: absolute;
    top: 10px;
    left: 10px;
    right: 10px;
    bottom: 10px;
    border: 2px solid rgba(255, 255, 255, 0.3);
}

/* Live Recording Dot Design */
.hud-rec-indicator {
    position: absolute;
    top: 20px;
    left: 20px;
    display: flex;
    align-items: center;
    background: rgba(0, 0, 0, 0.5);
    padding: 4px 8px;
    border-radius: 4px;
}

.red-dot {
    width: 8px;
    height: 8px;
    background-color: #ff3b30;
    border-radius: 50%;
    margin-right: 6px;
    animation: blink 1s infinite steps(2, start);
}

.rec-text {
    color: #fff;
    font-size: 11px;
    font-weight: bold;
    letter-spacing: 1px;
}

/* Center Capture Target Crosshair */
.hud-crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 20px;
    height: 20px;
    transform: translate(-50%, -50%);
    opacity: 0.6;
}

.cross-x, .cross-y {
    position: absolute;
    background: #fff;
}

.cross-x {
    top: 9px;
    left: 0;
    width: 20px;
    height: 20px;
    border-top: 2px solid white;
}

.cross-y {
    left: 9px;
    top: 0;
    height: 20px;
    border-left: 2px solid white;
}

@keyframes blink {
    to { visibility: hidden; }
}
</style>
