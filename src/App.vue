    <script setup lang="ts">
    import { useDevicesList, useUserMedia } from '@vueuse/core'
    import { reactive, shallowRef,  watchEffect, nextTick, onMounted, ref } from 'vue'

    const currentCamera = shallowRef<string>()
    const { videoInputs: cameras } = useDevicesList({
        requestPermissions: true,
        onUpdated() {
            if (!cameras.value.find(i => i.deviceId === currentCamera.value))
                currentCamera.value = cameras.value[0]?.deviceId
        },
    })

    const video = shallowRef<HTMLVideoElement | null>(null)
    const canvas = shallowRef<HTMLCanvasElement | null>(null)
    const photoList = shallowRef<string[]>([])
    const facingMode = shallowRef<'user' | 'environment'>('user')
    const { stream, enabled } = useUserMedia({
        constraints: reactive({
            video: {
                deviceId: currentCamera,
                advanced: [{ width: 1280 }, { height: 720 }],
                aspectRatio: 1.7777777778,
                facingMode,
            },
        }),
    })
    const timer = ref(3);
    const isShowTimer = ref(false);
    const isBlinking = ref(false);

    watchEffect(() => {
        if (video.value) video.value.srcObject = stream.value!
    })

    async function capturePhoto() {
        if (photoList.value.length === 4) {
            clearPhotos()
        }
        const firstShoot = photoList.value.length === 0
        const timeOut = firstShoot ? 3000 : 0
        if (firstShoot) {
            timerCountDown()
        }
        setTimeout(async() => {
            if (video.value && canvas.value) {
                const context = canvas.value.getContext('2d')
                if (context) {
                    canvas.value.width = video.value.videoWidth
                    canvas.value.height = video.value.videoHeight
                    context.drawImage(video.value, 0, 0, canvas.value.width, canvas.value.height)
                    const photo = canvas.value.toDataURL('image/png')
                    isBlinking.value = true
                    setTimeout(() => isBlinking.value = false, 200)
                    photoList.value.push(photo)
                }
            }

            await nextTick().then(()=> {
                enabled.value = false
                enabled.value = true
            }).finally(()=> {
                if(photoList.value.length < 4){
                    timerCountDown()
                }
            })
            if(photoList.value.length < 4){
                setTimeout(() => {
                    capturePhoto()
                }, 3000)
            }
        }, timeOut);
    }

    function switchCamera() {
        facingMode.value = facingMode.value === 'user' ? 'environment' : 'user'
    }

    function clearPhotos() {
        photoList.value = []
    }

    function timerCountDown(){
        isShowTimer.value = true;
        let count = timer.value;
        const interval = setInterval(() => {
            timer.value--
            if(timer.value === 0){
                playAudio('./shutter-camera.mp3')
                isShowTimer.value = false;
                clearInterval(interval)
                timer.value = count
            }
        }, 1000);
    }

    function playAudio(audioUrl: string) {
        const audio = new Audio(audioUrl);
        audio.play().catch(error => {
            console.error('Error playing audio:', error);
        });
    }

    onMounted(() => {
        setTimeout(() => {
            enabled.value = true
        }, 1000);
    })
    </script>

    <template>
        <div class="w-full h-screen bg-gradient-to-l from-blue-100 to-blue-300 flex items-center justify-between px-64 relative">
            <!-- Video Camera -->
            <div class="w-2/3 h-fit bg-white relative rounded-2xl overflow-hidden">
                <div v-if="isBlinking" class="blink-effect"></div>
                <div class="w-full h-full relative">
                    <span v-if="isShowTimer" class="absolute top-1/2 left-1/2 z-50">
                        <span class="text-6xl font-semibold text-white">{{ timer }}</span>
                    </span>
                    <video
                        ref="video"
                        muted
                        autoplay
                        class="w-full h-full object-cover"
                        :class="{ 'scale-x-[-1]': facingMode === 'user' }"
                    />
                </div>
                <div class="w-full flex items-center justify-center py-3 bg-white ro">
                    <div class="w-16 h-16 pointer rounded-full border border-white p-1 flex items-center justify-center group cursor-pointer">
                        <button v-if="enabled" @click="capturePhoto" class="w-12 h-12 rounded-full bg-red-500 group-hover:bg-red-600 transition-colors cursor-pointer"></button>
                    </div>
                </div>
                <div class="absolute top-2 right-2 flex items-center gap-x-3">
                    <!-- <button @click="switchCamera" class="bg-purple-500 text-white px-4 py-2 rounded">Flip Camera</button> -->
                    <!-- <button @click="clearPhotos" class="bg-yellow-500 text-white px-4 py-2 rounded">Try Again</button> -->
                </div>
            </div>

            <!-- Captured Photos -->
            <div class="w-80 grid grid-cols-1 bg-white p-4 gap-y-3 pb-16">
                <div
                v-for="index in 4"
                :key="index"
                class="border border-gray-300 rounded shadow-md h-[20vh] flex items-center justify-center transition-all duration-500"
                :class="{ 'bg-gray-100': !photoList[index - 1], 'scale-x-[-1]': facingMode === 'user' }"
                >
                <img
                    v-if="photoList[index - 1]"
                    :src="photoList[index - 1]"
                    alt="Captured"
                    class="w-full h-full object-cover rounded"
                />
                </div>
            </div>

            <canvas ref="canvas" class="hidden"></canvas>
        </div>
    </template>

    <style>
    /* Efek Blink */
    .blink-effect {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: white;
        opacity: 1;
        z-index: 99;
        animation: blink 0.2s ease-in-out;
    }

    @keyframes blink {
        0% { opacity: 1; }
        100% { opacity: 0; }
    }

    /* Animasi Slide-in untuk Foto */
    .slide-enter-active {
        transition: transform 0.5s ease-out, opacity 0.5s ease-out;
    }
    .slide-enter-from {
        transform: translateX(100%);
        opacity: 0;
    }
    .slide-enter-to {
        transform: translateX(0);
        opacity: 1;
    }
    </style>
