<template>
    <div class="video-container">
        <video
            ref="video"
            width="720"
            height="560"
            autoplay
            muted
            playsinline
        ></video>
        <canvas ref="canvas" width="720" height="560"></canvas>
        <div class="controls">
            <button @click="startVideo">Start Video</button>
            <button @click="pauseVideo">Pause Video</button>
            <button @click="stopVideo">Stop Video</button>
        </div>
    </div>
</template>

<script setup>
import * as faceapi from "face-api.js";
import { ref } from "vue";

const video = ref(null);
const canvas = ref(null);
const ctx = ref(null);

const setupCamera = async () => {
    const stream = await navigator.mediaDevices.getUserMedia({
        video: { width: 720, height: 560 },
        audio: false,
    });
    video.value.srcObject = stream;
    await new Promise((resolve) => {
        video.value.onloadedmetadata = () => {
            resolve();
            video.value.play();
        };
    });
};

const loadModels = async () => {
    await faceapi.nets.ssdMobilenetv1.loadFromUri("/models");
    await faceapi.nets.faceLandmark68Net.loadFromUri("/models");
};

const distanceBetweenPoints = (point1, point2) => {
    const dx = point1.x - point2.x;
    const dy = point1.y - point2.y;
    return Math.sqrt(dx * dx + dy * dy);
};

const calculateAngle = (p1, p2, p3) => {
    const angle =
        Math.atan2(p3.y - p2.y, p3.x - p2.x) -
        Math.atan2(p1.y - p2.y, p1.x - p2.x);
    return angle * (180 / Math.PI);
};

const calculateFeatures = (landmarks) => {
    const leftCheek = landmarks.positions[0];
    const rightCheek = landmarks.positions[16];
    const chin = landmarks.positions[8];
    const foreheadTop = landmarks.positions[19];
    const leftJaw = landmarks.positions[4];
    const rightJaw = landmarks.positions[12];

    const faceWidth = distanceBetweenPoints(leftCheek, rightCheek);
    const jawWidth = distanceBetweenPoints(leftJaw, rightJaw);
    const faceLength = distanceBetweenPoints(foreheadTop, chin);
    const foreheadWidth = distanceBetweenPoints(
        landmarks.positions[19],
        landmarks.positions[24]
    );
    const cheekboneWidth = distanceBetweenPoints(
        landmarks.positions[1],
        landmarks.positions[15]
    );

    const jawAngle = calculateAngle(leftJaw, chin, rightJaw);

    return {
        faceWidth,
        jawWidth,
        faceLength,
        foreheadWidth,
        cheekboneWidth,
        jawAngle,
        faceWidthToLength: faceWidth / faceLength,
        jawWidthToCheekboneWidth: jawWidth / cheekboneWidth,
    };
};

const classifyFaceShape = (features) => {
    const { faceWidthToLength, jawWidthToCheekboneWidth, jawAngle } = features;

    if (faceWidthToLength > 0.8 && faceWidthToLength < 1.0) {
        if (jawAngle > 100) {
            return "Square";
        } else if (jawWidthToCheekboneWidth < 0.8) {
            return "Heart";
        } else {
            return "Round";
        }
    } else if (faceWidthToLength < 0.8) {
        return "Oval";
    } else if (faceWidthToLength > 1.0) {
        return "Diamond";
    }
    return "Unknown";
};

const onPlay = async () => {
    if (video.value.paused || video.value.ended) return;

    console.log("detections");
    const detections = await faceapi
        .detectAllFaces(video.value)
        .withFaceLandmarks();

    ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height);
    detections.forEach((detection) => {
        const landmarks = detection.landmarks;
        const features = calculateFeatures(landmarks);
        const faceShape = classifyFaceShape(features);

        ctx.value.strokeStyle = "red";
        ctx.value.lineWidth = 2;
        ctx.value.strokeRect(
            detection.detection.box.x,
            detection.detection.box.y,
            detection.detection.box.width,
            detection.detection.box.height
        );
        ctx.value.fillText(
            faceShape,
            detection.detection.box.x,
            detection.detection.box.y + detection.detection.box.height + 10
        );
    });

    requestAnimationFrame(onPlay);
};

const pauseVideo = () => {
    video.value.pause();
    cancelAnimationFrame(animationFrameId);
};
const stopVideo = () => {
    video.value.pause();
    video.value.srcObject.getTracks().forEach((track) => track.stop());
    ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height);
    cancelAnimationFrame(animationFrameId);
};
const startVideo = async () => {
    await setupCamera();
    video.value.play();
};

const load = async () => {
    console.log("models loaded");
    await loadModels();
    console.log("models loaded");

    ctx.value = canvas.value.getContext("2d");
    canvas.fillStyle = "#ff00ff";

    video.value.addEventListener("play", onPlay);
};
load();
</script>

<style>
.video-container {
    position: relative;
    width: 720px;
    height: 560px;
}
.controls {
    margin-top: 10px;
}
#video {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* Ensure canvas does not interfere with video controls */
}
</style>
