<template>
    <Modal :show="modelValue" @close="closeModal" max-width="full">
        <div class="p-4 md:p-6 h-full flex flex-col">
            <div class="flex items-start justify-between mb-4 flex-shrink-0">
                <div class="flex-1 pr-4">
                    <h2 class="text-lg font-medium text-gray-900 truncate">
                        {{ file?.name }}
                    </h2>
                    <!-- Description Section -->
                    <div class="mt-3">
                        <div v-if="!isEditingDescription" class="flex items-start gap-2">
                            <div class="flex-1">
                                <p v-if="file?.description" class="text-sm text-gray-600 whitespace-pre-wrap">
                                    {{ file.description }}
                                </p>
                                <p v-else class="text-sm text-gray-400 italic">
                                    No description
                                </p>
                            </div>
                            <button
                                v-if="canEditDescription"
                                @click="startEditingDescription"
                                class="text-gray-400 hover:text-gray-600 focus:outline-none flex-shrink-0 mt-1"
                                title="Edit description"
                            >
                                <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
                                </svg>
                            </button>
                        </div>
                        <div v-else class="space-y-2">
                            <textarea
                                v-model="descriptionEdit"
                                rows="3"
                                class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 text-sm"
                                placeholder="Add a description..."
                                maxlength="5000"
                            ></textarea>
                            <div class="flex items-center justify-between">
                                <span class="text-xs text-gray-500">{{ descriptionEdit.length }}/5000</span>
                                <div class="flex gap-2">
                                    <button
                                        @click="cancelEditingDescription"
                                        class="px-3 py-1 text-sm text-gray-700 bg-gray-100 hover:bg-gray-200 rounded-md focus:outline-none"
                                    >
                                        Cancel
                                    </button>
                                    <button
                                        @click="saveDescription"
                                        :disabled="savingDescription"
                                        class="px-3 py-1 text-sm text-white bg-indigo-600 hover:bg-indigo-700 rounded-md focus:outline-none disabled:opacity-50 disabled:cursor-not-allowed"
                                    >
                                        {{ savingDescription ? 'Saving...' : 'Save' }}
                                    </button>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
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
import { ref, watch, computed } from "vue";
import { isImage, isVideo } from "@/Helper/file-helper.js";
import { httpGet, httpPatch } from "@/Helper/http-helper.js";
import { usePage } from "@inertiajs/vue3";

// Refs
const loading = ref(false);
const error = ref(null);
const previewUrl = ref(null);
const isEditingDescription = ref(false);
const descriptionEdit = ref('');
const savingDescription = ref(false);

// Props & Emit
const props = defineProps({
    modelValue: Boolean,
    file: Object
});

const emit = defineEmits(['update:modelValue', 'fileUpdated']);

const page = usePage();

// Computed
const canEditDescription = computed(() => {
    if (!props.file) return false;
    // Check if user owns the file (created_by matches current user id)
    return props.file.created_by === page.props.auth?.user?.id;
});

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

function startEditingDescription() {
    descriptionEdit.value = props.file?.description || '';
    isEditingDescription.value = true;
}

function cancelEditingDescription() {
    isEditingDescription.value = false;
    descriptionEdit.value = '';
}

async function saveDescription() {
    if (!props.file) return;

    savingDescription.value = true;
    try {
        const response = await httpPatch(route('file.updateDescription'), {
            id: props.file.id,
            description: descriptionEdit.value.trim() || null
        });

        // Emit event to parent component to update the file in the list
        emit('fileUpdated', response);

        isEditingDescription.value = false;
    } catch (err) {
        error.value = err.error?.message || 'Failed to save description';
        console.error('Error saving description:', err);
    } finally {
        savingDescription.value = false;
    }
}

// Watch for file changes
watch(() => props.modelValue, (newValue) => {
    if (newValue && props.file) {
        loadPreview();
        // Reset editing state when modal opens
        isEditingDescription.value = false;
        descriptionEdit.value = '';
    }
});

watch(() => props.file, () => {
    if (props.modelValue && props.file) {
        loadPreview();
        // Reset editing state when file changes
        isEditingDescription.value = false;
        descriptionEdit.value = '';
    }
});

</script>

<style scoped>

</style>

