<script lang="ts">
    import { toast } from 'svelte-sonner';
    import { v4 as uuidv4 } from 'uuid';
    import { tick, getContext, onMount, onDestroy } from 'svelte';

    const i18n = getContext('i18n');

    import { config, mobile, settings, socket } from '$lib/stores';
    import { blobToFile, compressImage } from '$lib/utils';
    import Tooltip from '../common/Tooltip.svelte';
    import RichTextInput from '../common/RichTextInput.svelte';
    import VoiceRecording from '../chat/MessageInput/VoiceRecording.svelte';
    import InputMenu from './MessageInput/InputMenu.svelte';
    import { uploadFile } from '$lib/apis/files';
    import { WEBUI_API_BASE_URL } from '$lib/constants';
    import FileItem from '../common/FileItem.svelte';
    import Image from '../common/Image.svelte';
    import FilesOverlay from '../chat/MessageInput/FilesOverlay.svelte';

    export let placeholder = $i18n.t('Send a Message');
    export let transparentBackground = false;
    export let id = null;

    let draggedOver = false;
    let recording = false;
    let content = '';
    let files = [];
    let filesInputElement;
    let inputFiles;
    export let typingUsers = [];
    export let onSubmit: Function;
    export let onChange: Function;
    export let scrollEnd = true;
    export let scrollToBottom: Function = () => {};

    const screenCaptureHandler = async () => {
        try {
            const mediaStream = await navigator.mediaDevices.getDisplayMedia({
                video: { cursor: 'never' },
                audio: false
            });
            const video = document.createElement('video');
            video.srcObject = mediaStream;
            await video.play();

            const canvas = document.createElement('canvas');
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;

            const context = canvas.getContext('2d');
            context.drawImage(video, 0, 0, canvas.width, canvas.height);

            mediaStream.getTracks().forEach((track) => track.stop());
            window.focus();

            const imageUrl = canvas.toDataURL('image/png');
            files = [...files, { type: 'image', url: imageUrl }];

            video.srcObject = null;
        } catch (error) {
            console.error('Error capturing screen:', error);
        }
    };

    const inputFilesHandler = async (inputFiles) => {
        inputFiles.forEach((file) => {
            if (
                ($config?.file?.max_size ?? null) !== null &&
                file.size > ($config?.file?.max_size ?? 0) * 1024 * 1024
            ) {
                toast.error(
                    $i18n.t(`File size should not exceed {{maxSize}} MB.`, {
                        maxSize: $config?.file?.max_size
                    })
                );
                return;
            }

            if (['image/gif', 'image/webp', 'image/jpeg', 'image/png', 'image/avif'].includes(file.type)) {
                let reader = new FileReader();
                reader.onload = async (event) => {
                    let imageUrl = event.target.result;
                    if (
                        $settings?.imageCompression ||
                        $config?.file?.image_compression?.width ||
                        $config?.file?.image_compression?.height
                    ) {
                        let width = $settings?.imageCompressionSize?.width ?? null;
                        let height = $settings?.imageCompressionSize?.height ?? null;

                        if ($config?.file?.image_compression?.width && width > $config.file.image_compression.width)
                            width = $config.file.image_compression.width;
                        if ($config?.file?.image_compression?.height && height > $config.file.image_compression.height)
                            height = $config.file.image_compression.height;

                        if (width || height) {
                            imageUrl = await compressImage(imageUrl, width, height);
                        }
                    }
                    files = [...files, { type: 'image', url: `${imageUrl}` }];
                };
                reader.readAsDataURL(file);
            } else {
                uploadFileHandler(file);
            }
        });
    };

    const uploadFileHandler = async (file) => {
        const tempItemId = uuidv4();
        const fileItem = {
            type: 'file',
            file: '',
            id: null,
            url: '',
            name: file.name,
            collection_name: '',
            status: 'uploading',
            size: file.size,
            error: '',
            itemId: tempItemId
        };

        if (fileItem.size == 0) {
            toast.error($i18n.t('You cannot upload an empty file.'));
            return null;
        }

        files = [...files, fileItem];

        try {
            let metadata = null;
            if ((file.type.startsWith('audio/') || file.type.startsWith('video/')) && $settings?.audio?.stt?.language) {
                metadata = { language: $settings?.audio?.stt?.language };
            }

            const uploadedFile = await uploadFile(localStorage.token, file, metadata);

            if (uploadedFile) {
                fileItem.status = 'uploaded';
                fileItem.file = uploadedFile;
                fileItem.id = uploadedFile.id;
                fileItem.collection_name = uploadedFile?.meta?.collection_name || uploadedFile?.collection_name;
                fileItem.url = `${WEBUI_API_BASE_URL}/files/${uploadedFile.id}`;
                files = files;
            } else {
                files = files.filter((item) => item?.itemId !== tempItemId);
            }
        } catch (e) {
            toast.error(`${e}`);
            files = files.filter((item) => item?.itemId !== tempItemId);
        }
    };

    const handleKeyDown = (event: KeyboardEvent) => {
        if (event.key === 'Escape') {
            draggedOver = false;
        }
    };

    const onDragOver = (e) => {
        e.preventDefault();
        if (e.dataTransfer?.types?.includes('Files')) {
            draggedOver = true;
        } else {
            draggedOver = false;
        }
    };

    const onDragLeave = () => {
        draggedOver = false;
    };

    const onDrop = async (e) => {
        e.preventDefault();
        if (e.dataTransfer?.files) {
            const inputFiles = Array.from(e.dataTransfer.files);
            if (inputFiles.length > 0) inputFilesHandler(inputFiles);
        }
        draggedOver = false;
    };

    const submitHandler = async () => {
        if (content === '' && files.length === 0) return;

        onSubmit({
            content,
            data: { files }
        });

        content = '';
        files = [];
        await tick();
        const chatInputElement = document.getElementById(`chat-input-${id}`);
        chatInputElement?.focus();
    };

    $: if (content) onChange();

    onMount(async () => {
        window.setTimeout(() => {
            const chatInput = document.getElementById(`chat-input-${id}`);
            chatInput?.focus();
        }, 0);
        window.addEventListener('keydown', handleKeyDown);

        const dropzoneElement = document.getElementById('channel-container');
        dropzoneElement?.addEventListener('dragover', onDragOver);
        dropzoneElement?.addEventListener('drop', onDrop);
        dropzoneElement?.addEventListener('dragleave', onDragLeave);
    });

    onDestroy(() => {
        window.removeEventListener('keydown', handleKeyDown);
        const dropzoneElement = document.getElementById('channel-container');
        dropzoneElement?.removeEventListener('dragover', onDragOver);
        dropzoneElement?.removeEventListener('drop', onDrop);
        dropzoneElement?.removeEventListener('dragleave', onDragLeave);
    });
</script>

<FilesOverlay show={draggedOver} />

<input
    bind:this={filesInputElement}
    bind:files={inputFiles}
    type="file"
    hidden
    multiple
    on:change={() => {
        if (inputFiles && inputFiles.length > 0) {
            inputFilesHandler(Array.from(inputFiles));
        } else {
            toast.error($i18n.t(`File not found.`));
        }
        filesInputElement.value = '';
    }}
/>

<div class="bg-transparent">
    <div class="{($settings?.widescreenMode ?? null) ? 'max-w-full' : 'max-w-6xl'} px-2.5 mx-auto inset-x-0 relative">
        <div class="absolute top-0 left-0 right-0 mx-auto inset-x-0 bg-transparent flex justify-center">
            <div class="flex flex-col px-3 w-full">
                <div class="relative">
                    {#if !scrollEnd}
                        <div class="absolute -top-12 left-0 right-0 flex justify-center z-30 pointer-events-none">
                            <button
                                class="bg-white border border-gray-100 dark:border-none dark:bg-white/20 p-1.5 rounded-none pointer-events-auto"
                                on:click={() => {
                                    scrollEnd = true;
                                    scrollToBottom();
                                }}
                            >
                                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5">
                                    <path fill-rule="evenodd" d="M10 3a.75.75 0 01.75.75v10.638l3.96-4.158a.75.75 0 111.08 1.04l-5.25 5.5a.75.75 0 01-1.08 0l-5.25-5.5a.75.75 0 111.08-1.04l3.96 4.158V3.75A.75.75 0 0110 3z" clip-rule="evenodd" />
                                </svg>
                            </button>
                        </div>
                    {/if}
                </div>
                <div class="relative">
                    <div class="-mt-5">
                        {#if typingUsers.length > 0}
                            <div class="text-xs px-4 mb-1">
                                <span class="font-normal text-black dark:text-white">
                                    {typingUsers.map((user) => user.name).join(', ')}
                                </span>
                                {$i18n.t('is typing...')}
                            </div>
                        {/if}
                    </div>
                </div>
            </div>
        </div>
        <div class="">
            {#if recording}
                <VoiceRecording
                    bind:recording
                    onCancel={async () => {
                        recording = false;
                        await tick();
                        document.getElementById(`chat-input-${id}`)?.focus();
                    }}
                    onConfirm={async (data) => {
                        const { text, filename } = data;
                        content = `${content}${text} `;
                        recording = false;
                        await tick();
                        document.getElementById(`chat-input-${id}`)?.focus();
                    }}
                />
            {:else}
                <form
                    class="w-full flex gap-1.5"
                    on:submit|preventDefault={submitHandler}
                >
                    <div class="flex-1 flex flex-col relative w-full rounded-none border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 shadow-sm dark:text-gray-100" dir={$settings?.chatDirection ?? 'auto'}>
                        {#if files.length > 0}
                            <div class="mx-2 mt-2.5 -mb-1 flex flex-wrap gap-2">
                                {#each files as file, fileIdx}
                                    {#if file.type === 'image'}
                                        <div class="relative group">
                                            <div class="relative">
                                                <Image src={file.url} alt="input" imageClassName="h-16 w-16 rounded-none object-cover" />
                                            </div>
                                            <div class="absolute -top-1 -right-1">
                                                <button
                                                    class="bg-white text-black border border-white rounded-none group-hover:visible invisible transition"
                                                    type="button"
                                                    on:click={() => {
                                                        files.splice(fileIdx, 1);
                                                        files = files;
                                                    }}
                                                >
                                                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-4 h-4">
                                                        <path d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z" />
                                                    </svg>
                                                </button>
                                            </div>
                                        </div>
                                    {:else}
                                        <FileItem
                                            item={file}
                                            name={file.name}
                                            type={file.type}
                                            size={file?.size}
                                            loading={file.status === 'uploading'}
                                            dismissible={true}
                                            edit={true}
                                            on:dismiss={() => {
                                                files.splice(fileIdx, 1);
                                                files = files;
                                            }}
                                            on:click={() => console.log(file)}
                                        />
                                    {/if}
                                {/each}
                            </div>
                        {/if}
                        <div class="px-2.5">
                            <div class="scrollbar-hidden font-primary text-left bg-transparent dark:text-gray-100 outline-hidden w-full pt-3 px-1 rounded-none resize-none h-fit max-h-80 overflow-auto border-b border-gray-200 dark:border-gray-700">
                                <RichTextInput
                                    bind:value={content}
                                    id={`chat-input-${id}`}
                                    messageInput={true}
                                    shiftEnter={!$mobile || !('ontouchstart' in window || navigator.maxTouchPoints > 0 || navigator.msMaxTouchPoints > 0)}
                                    {placeholder}
                                    largeTextAsFile={$settings?.largeTextAsFile ?? false}
                                    on:keydown={(e) => {
                                        e = e.detail.event;
                                        const isCtrlPressed = e.ctrlKey || e.metaKey;
                                        if (!$mobile || !('ontouchstart' in window || navigator.maxTouchPoints > 0 || navigator.msMaxTouchPoints > 0)) {
                                            if (e.keyCode === 13 && !e.shiftKey) {
                                                e.preventDefault();
                                            }
                                            if (content !== '' && e.keyCode === 13 && !e.shiftKey) {
                                                submitHandler();
                                            }
                                        }
                                        if (e.key === 'Escape') console.info('Escape');
                                    }}
                                    on:paste={(e) => {
                                        e = e.detail.event;
                                        console.info(e);
                                    }}
                                />
                            </div>
                        </div>
                        <div class="flex justify-between mb-2.5 mt-1.5 mx-0.5">
                            <div class="ml-1 self-end flex space-x-1">
                                <InputMenu
                                    {screenCaptureHandler}
                                    uploadFilesHandler={() => filesInputElement.click()}
                                >
                                    <button
                                        class="bg-transparent hover:bg-gray-100 dark:hover:bg-gray-700 transition rounded-none p-1.5 outline-hidden focus:outline-hidden"
                                        type="button"
                                        aria-label="More"
                                    >
                                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="size-5">
                                            <path d="M10.75 4.75a.75.75 0 0 0-1.5 0v4.5h-4.5a.75.75 0 0 0 0 1.5h4.5v4.5a.75.75 0 0 0 1.5 0v-4.5h4.5a.75.75 0 0 0 0-1.5h-4.5v-4.5Z" />
                                        </svg>
                                    </button>
                                </InputMenu>
                            </div>
                            <div class="self-end flex space-x-1 mr-1">
                                {#if content === ''}
                                    <Tooltip content={$i18n.t('Record voice')}>
                                        <button
                                            id="voice-input-button"
                                            class="text-gray-600 dark:text-gray-300 hover:text-gray-700 dark:hover:text-gray-200 transition rounded-none p-1.5 mr-0.5 self-center"
                                            type="button"
                                            on:click={async () => {
                                                try {
                                                    let stream = await navigator.mediaDevices.getUserMedia({ audio: true }).catch(function (err) {
                                                        toast.error($i18n.t(`Permission denied when accessing microphone: {{error}}`, { error: err }));
                                                        return null;
                                                    });
                                                    if (stream) {
                                                        recording = true;
                                                        const tracks = stream.getTracks();
                                                        tracks.forEach((track) => track.stop());
                                                    }
                                                    stream = null;
                                                } catch {
                                                    toast.error($i18n.t('Permission denied when accessing microphone'));
                                                }
                                            }}
                                            aria-label="Voice Input"
                                        >
                                            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5 translate-y-[0.5px]">
                                                <path d="M7 4a3 3 0 016 0v6a3 3 0 11-6 0V4z" />
                                                <path d="M5.5 9.643a.75.75 0 00-1.5 0V10c0 3.06 2.29 5.585 5.25 5.954V17.5h-1.5a.75.75 0 000 1.5h4.5a.75.75 0 000-1.5h-1.5v-1.546A6.001 6.001 0 0016 10v-.357a.75.75 0 00-1.5 0V10a4.5 4.5 0 01-9 0v-.357z" />
                                            </svg>
                                        </button>
                                    </Tooltip>
                                {/if}
                                <div class="flex items-center">
                                    <Tooltip content={$i18n.t('Send Message')}>
                                        <button 
										class="inline-flex
											items-center
											justify-center
											relative
											shrink-0
											can-focus
											select-none
											disabled:pointer-events-none
											disabled:opacity-50
											disabled:shadow-none
											disabled:drop-shadow-none 
											bg-accent-main-000
											text-oncolor-100
											font-styrene
											font-medium
											transition-all
											duration-200
											ease-in-out
											hover:bg-accent-main-200 
											h-8 w-8 
											rounded-none <!-- Square corners -->
											hover:translate-y-[-1px] hover:shadow-sm <!-- Slight hover lift -->
											active:scale-95"
										type="button" 
										aria-label="Send message">
										<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 256 256">
											<path d="M208.49,120.49a12,12,0,0,1-17,0L140,69V216a12,12,0,0,1-24,0V69L64.49,120.49a12,12,0,0,1-17-17l72-72a12,12,0,0,1,17,0l72,72A12,12,0,0,1,208.49,120.49Z"></path>
										</svg>
									</button>
									</Tooltip>
                                </div>
                            </div>
                        </div>
                    </div>
                </form>
            {/if}
        </div>
    </div>
</div>