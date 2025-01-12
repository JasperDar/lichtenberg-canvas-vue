<script setup lang="ts">
import { onMounted, reactive, type Ref, ref } from "vue";
import { useWindowSize, type Fn, useRafFn } from "@vueuse/core";

const props = defineProps({
    margin: {
        type: Number,
        default: 0.25,
    },
    lineColor: {
        type: String,
        default: '#888888',
    },
    lineOpacity: {
        type: Number,
        default: 0.25,
    },
    lineWidth: {
        type: Number,
        default: 1,
    },
    lineLength: {
        type: Number,
        default: 6,
    },
    targetDepth: {
        type: Number,
        default: 100,
    },
    forkAmount: {
        type: Number,
        default: 2,
    },
    forkAngle: {
        type: Number,
        default: 30,
    },
    fps: {
        type: Number,
        default: 40,
    },
    inset: {
        type: Number,
        default: 40,
    },
    bounds: {
        type: Number,
        default: 40,
    },
});

const canvas = ref<HTMLCanvasElement|null>(null);

const segments = ref<Fn[]>([]);

const lastAnimationFrame = ref<DOMHighResTimeStamp>(performance.now());

const size = reactive<{
    width: Ref<number, number>;
    height: Ref<number, number>;
}>(useWindowSize());

onMounted(async () => {
    start();
});

const start = () : void => {
    setCanvasDimensions();
    setCanvasStyling();
    startAnimation();
};

const reset = () : void => {
    setCanvasStyling();
    startAnimation();
};

defineExpose({
    reset,
});

const setCanvasDimensions = () : void => {
    const devicePixelRatio = window.devicePixelRatio || 1;

    canvas.value!.style.width = `${size.width}px`;
    canvas.value!.style.height = `${size.height}px`;
    canvas.value!.width = (devicePixelRatio * size.width);
    canvas.value!.height = (devicePixelRatio * size.height);

    const ctx = canvas.value!.getContext('2d');

    ctx!.scale(devicePixelRatio, devicePixelRatio);
};

const setCanvasStyling = () : void => {
    const ctx = canvas.value!.getContext('2d');

    ctx!.clearRect(0, 0, canvas.value!.width, canvas.value!.height);
    ctx!.lineWidth = props.lineWidth;
    ctx!.strokeStyle = getFullHexCode();
};

const getFullHexCode = () => {
    let alpha = Math.round(props.lineOpacity * 255).toString(16).padStart(2, '0');

    if (props.lineColor.startsWith('--')) {
        const styles = getComputedStyle(document.documentElement);
        const color = styles.getPropertyValue(props.lineColor).trim();

        return color + alpha;
    }

    return props.lineColor + alpha;
};

const processSegment = (startX: number, startY: number, angle: number, depth: { value: number; } = {value: 0}) : void => {
    // Draw segment line
    const [ endX, endY ] = calculateEndCoordinates(startX, startY, angle);

    const ctx = canvas.value!.getContext('2d');

    ctx!.beginPath();
    ctx!.moveTo(startX, startY);
    ctx!.lineTo(endX, endY);
    ctx!.stroke();

    depth.value += 1;

    generateNewSegments(endX, endY, angle, depth);
};

const calculateEndCoordinates = (startX: number, startY: number, angle: number) : number[] => {
    let lineLength = calculateLineLength();

    const dX = lineLength * Math.cos(angle);
    const dY = lineLength * Math.sin(angle);

    const endX = startX + dX;
    const endY = startY + dY;

    return [ endX, endY ];
};

const calculateLineLength = () : number => {
    return Math.random() * props.lineLength;
};

const generateNewSegments = (startX: number, startY: number, angle: number, depth: { value: number; }) : void => {
    // Don't generate new segments if current segment is out of bounds
    if (startX < -props.bounds || startX > size.width + props.bounds) {
        return;
    }

    if (startY < -props.bounds || startY > size.height + props.bounds) {
        return;
    }

    // Calculate min and max angle
    const minAngle = angle - (Math.random() * degreesToRadians(props.forkAngle / 2));
    const maxAngle = angle + (Math.random() * degreesToRadians(props.forkAngle / 2));

    // Calculate rate based on depth and amount of branches
    const branchChancePercentage = calculateBranchChancePercentage(depth);

    for (let i = 0; i < props.forkAmount; i++) {
        if (Math.random() > branchChancePercentage) {
            continue;
        }

        const newAngle = Math.random() * (maxAngle - minAngle) + minAngle;

        segments.value.push(() => processSegment(startX, startY, newAngle, depth));
    }

};

const degreesToRadians = (degrees: number) : number => {
    return degrees * Math.PI / 180;
};

const calculateBranchChancePercentage = (depth: { value: number; }) : number => {
    // Target depth reached, we can fade out generation
    // Amount of branches generated * percentage chance should be 0.5 or less
    if (depth.value > props.targetDepth) {
        return Math.max(0.5, (1 / props.forkAmount));
    }

    // Target depth not yet reached, we want to generate branches more liberally
    // Amount of branches generated * percentage chance should be 0.5 or greater
    return 0.5 + (0.5 / props.forkAmount);
};

let controls = useRafFn(() => {
    if ((performance.now() - lastAnimationFrame.value) < getFrameInterval()) {
        return;
    }

    let segmentsToRender:Fn[] = segments.value;
    segments.value = [];

    lastAnimationFrame.value = performance.now();

    if (! segmentsToRender.length) {
        controls.pause();
    }

    // Sender all segments for this frame
    segmentsToRender.forEach((processSegmentFunction: Fn) => {
        processSegmentFunction();
    });
}, {immediate: false});

const getFrameInterval = () : number => {
    return 1000 / props.fps;
};

const applyMarginToSize = (size: number) : number => {
    let margin = props.margin;

    if (margin < 0) {
        margin = 0;
    }

    if (margin > 0.5) {
        margin = 0.5;
    }

    const offLimitStartingArea = 2 * margin;
    const availableStartingArea = 1 - offLimitStartingArea;

    const multiplier = Math.random() * availableStartingArea + margin;

    return size * multiplier;
};

const startAnimation = () : void => {
    controls.pause();

    segments.value = [
        () => processSegment(applyMarginToSize(size.width), -props.inset, degreesToRadians(90)),
        () => processSegment(applyMarginToSize(size.width), size.height + props.inset, degreesToRadians(270)),
    ];

    // On bigger screens, we also render on the sides
    if (size.width > 768) {
        segments.value.push(() => processSegment(-props.inset, applyMarginToSize(size.height), 0));
        segments.value.push(() => processSegment(size.width + props.inset, applyMarginToSize(size.height), degreesToRadians(180)));
    }

    controls.resume();
};

</script>

<template>
    <div class="lichtenberg-canvas">
        <canvas ref="canvas" />
    </div>
</template>

<style scoped>
.lichtenberg-canvas {
    pointer-events: none;
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    top: 0;
    mask-image: radial-gradient(circle, transparent, black);
    --webkit-mask-image: radial-gradient(circle, transparent, black);

    @media print {
        visibility: hidden;
    }
}
</style>
