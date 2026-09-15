<template>
    <ion-card class="camera-card">
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
                        <span class="rec-text">LIVE ({{ currentFacingMode === 'user' ? 'FRONT' : 'BACK' }})</span>
                    </div>
                    <div class="hud-crosshair">
                        <div class="cross-x"></div>
                        <div class="cross-y"></div>
                    </div>
                </div>

                <!-- Floating Flip Camera Button over the Viewfinder -->
                <ion-button
                    fill="clear"
                    class="flip-btn"
                    @click="toggleCameraFacing"
                >
                    <ion-icon slot="icon-only" :icon="cameraReverseIcon" />
                </ion-button>
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
                color="success"
                @click="takePicture"
            >
                <ion-icon slot="start" :icon="cameraIcon" />
                Capture Photo
            </ion-button>

            <ion-button
                v-if="isCameraOpen"
                expand="block"
                fill="outline"
                color="danger"
                @click="closeCamera"
            >
                Cancel & Return
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
    IonIcon,
    IonText,
} from "@ionic/vue";

import { camera as cameraIcon, cameraReverse as cameraReverseIcon } from "ionicons/icons";
import { ref, nextTick, onUnmounted } from "vue";
import { Camera } from "@capacitor/camera";

const emit = defineEmits<{
    (event: "photoCaptured", photo: string): void;
    (event: "cameraStatusChange", isOpen: boolean): void;
}>();

const errorMessage = ref("");
const isCameraOpen = ref(false);
const currentFacingMode = ref<"user" | "environment">("user");

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
        // If an existing stream exists, kill its active tracks before requesting a new one
        if (stream) {
            stream.getTracks().forEach((track) => track.stop());
        }

        stream = await navigator.mediaDevices.getUserMedia({
            video: {
                facingMode: currentFacingMode.value,
                width: { ideal: 1280 },
                height: { ideal: 720 },
            },
            audio: false,
        });

        isCameraOpen.value = true;
        emit("cameraStatusChange", true);

        await nextTick();

        if (videoElement.value) {
            videoElement.value.srcObject = stream;
            await videoElement.value.play();
        }
    } catch (error) {
        console.error("Camera error:", error);
        errorMessage.value = "Unable to access camera hardware.";
    }
};

// Toggle handler to cycle back and front camera setups
const toggleCameraFacing = async () => {
    currentFacingMode.value = currentFacingMode.value === "user" ? "environment" : "user";
    // Instantly reopen device capture channels with the updated parameters
    if (isCameraOpen.value) {
        await openCamera();
    }
};

const takePicture = () => {
    if (!videoElement.value || !canvasElement.value) {
        return;
    }

    const video = videoElement.value;
    const canvas = canvasElement.value;

    if (video.videoWidth === 0 || video.videoHeight === 0) {
        errorMessage.value = "Camera feed stream not synchronized yet.";
        return;
    }

    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    const context = canvas.getContext("2d");
    if (!context) {
        errorMessage.value = "Unable to render canvas frames.";
        return;
    }

    context.drawImage(video, 0, 0, canvas.width, canvas.height);
    const photo = canvas.toDataURL("image/jpeg", 0.9);
    
    // Emit payload up to homepage
    emit("photoCaptured", photo);
    
    // Close up camera hardware pipeline automatically
    closeCamera();
};

const closeCamera = () => {
    if (stream) {
        stream.getTracks().forEach((track) => track.stop());
        stream = null;
    }
    isCameraOpen.value = false;
    emit("cameraStatusChange", false);
};

onUnmounted(() => {
    closeCamera();
});
</script>

<style scoped>
.camera-card {
    margin: 0;
    box-shadow: none;
    background: transparent;
}

.camera-viewport {
    position: relative;
    width: 100%;
    border-radius: 16px;
    overflow: hidden;
    margin-bottom: 16px;
    background: #000;
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
}

.camera-preview {
    display: block;
    width: 100%;
    height: auto;
    min-height: 380px;
    object-fit: cover;
}

.hidden-canvas {
    display: none;
}

/* Flip Camera Button Overlay styling */
.flip-btn {
    position: absolute;
    bottom: 16px;
    right: 16px;
    --background: rgba(0, 0, 0, 0.6);
    --color: #ffffff;
    --border-radius: 50%;
    width: 48px;
    height: 48px;
    backdrop-filter: blur(4px);
    z-index: 10;
}

/* HUD Overlay Styles */
.hud-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    box-sizing: border-box;
}

.hud-corners {
    position: absolute;
    top: 16px;
    left: 16px;
    right: 16px;
    bottom: 16px;
    border: 1.5px solid rgba(255, 255, 255, 0.25);
    border-radius: 8px;
}

.hud-rec-indicator {
    position: absolute;
    top: 24px;
    left: 24px;
    display: flex;
    align-items: center;
    background: rgba(0, 0, 0, 0.55);
    padding: 4px 10px;
    border-radius: 20px;
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
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 1px;
}

.hud-crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 24px;
    height: 24px;
    transform: translate(-50%, -50%);
    opacity: 0.5;
}

.cross-x, .cross-y {
    position: absolute;
    background: #fff;
}

.cross-x {
    top: 11px;
    left: 0;
    width: 24px;
    height: 2px;
}

.cross-y {
    left: 11px;
    top: 0;
    width: 2px;
    height: 24px;
}

@keyframes blink {
    to { visibility: hidden; }
}
</style>
