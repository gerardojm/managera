<template>
    <Modal :show="modelValue" @close="closeModal" max-width="full">
        <div class="p-4 md:p-6 h-full flex flex-col">
            <div class="flex items-center justify-between mb-4 flex-shrink-0">
                <h2 class="text-lg font-medium text-gray-900 truncate pr-4">
                    {{ file?.name }}
                </h2>
                <button
                    @click="closeModal"
                    class="text-gray-400 hover:text-gray-500 focus:outline-none flex-shrink-0"
                >
                    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                </button>
            </div>
            <div class="flex-1 flex items-center justify-center bg-gray-100 rounded-lg overflow-hidden min-h-0">
                <div v-if="loading" class="flex items-center justify-center p-8">
                    <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900"></div>
                </div>
                <div v-else-if="error" class="p-8 text-center text-red-600">
                    <p>Failed to load preview</p>
                    <p class="text-sm mt-2">{{ error }}</p>
                </div>
                <img
                    v-else-if="file && isImage(file)"
                    :src="previewUrl"
                    :alt="file.name"
                    class="max-w-full max-h-full w-auto h-auto object-contain"
                    @error="handleImageError"
                />
                <video
                    v-else-if="file && isVideo(file)"
                    :src="previewUrl"
                    controls
                    class="max-w-full max-h-full w-auto h-auto object-contain"
                    @error="handleVideoError"
                >
                    Your browser does not support the video tag.
                </video>
            </div>
        </div>
    </Modal>
</template>

<script setup>
// Imports
import Modal from "@/Components/Modal.vue";
import { ref, watch } from "vue";
import { isImage, isVideo } from "@/Helper/file-helper.js";
import { httpGet } from "@/Helper/http-helper.js";

// Refs
const loading = ref(false);
const error = ref(null);
const previewUrl = ref(null);

// Props & Emit
const props = defineProps({
    modelValue: Boolean,
    file: Object
});

const emit = defineEmits(['update:modelValue']);

// Methods
function closeModal() {
    emit('update:modelValue', false);
    previewUrl.value = null;
    error.value = null;
    loading.value = false;
}

async function loadPreview() {
    if (!props.file || (!isImage(props.file) && !isVideo(props.file))) {
        return;
    }

    loading.value = true;
    error.value = null;

    try {
        const params = new URLSearchParams();
        params.append('ids[]', props.file.id);
        
        const response = await httpGet(route('file.preview') + '?' + params.toString());
        
        if (response.url) {
            previewUrl.value = response.url;
        } else {
            error.value = response.message || 'Failed to load preview';
        }
    } catch (err) {
        error.value = err.message || 'Failed to load preview';
    } finally {
        loading.value = false;
    }
}

function handleImageError() {
    error.value = 'Failed to load image';
}

function handleVideoError() {
    error.value = 'Failed to load video';
}

// Watch for file changes
watch(() => props.modelValue, (newValue) => {
    if (newValue && props.file) {
        loadPreview();
    }
});

watch(() => props.file, () => {
    if (props.modelValue && props.file) {
        loadPreview();
    }
});

</script>

<style scoped>

</style>

