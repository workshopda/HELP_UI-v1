<script lang="ts">
	import { config, models, settings, user } from '$lib/stores';
	import { createEventDispatcher, onMount, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import { updateUserInfo } from '$lib/apis/users';
	import { getUserPosition } from '$lib/utils';
	const dispatch = createEventDispatcher();

	const i18n = getContext('i18n');

	export let saveSettings: Function;

	let backgroundImageUrl = null;
	let inputFiles = null;
	let filesInputElement;

	// Addons
	let titleAutoGenerate = true;
	let autoFollowUps = true;
	let autoTags = true;

	let responseAutoCopy = false;
	let widescreenMode = false;
	let splitLargeChunks = false;
	let scrollOnBranchChange = true;
	let userLocation = false;

	// Interface
	let defaultModelId = '';
	let showUsername = false;

	let notificationSound = true;
	let notificationSoundAlways = false;

	let highContrastMode = false;

	let detectArtifacts = true;

	let richTextInput = true;
	let promptAutocomplete = false;

	let largeTextAsFile = false;

	let landingPageMode = '';
	let chatBubble = true;
	let chatDirection: 'LTR' | 'RTL' | 'auto' = 'auto';
	let ctrlEnterToSend = false;
	let copyFormatted = false;

	let collapseCodeBlocks = false;
	let expandDetails = false;

	let imageCompression = false;
	let imageCompressionSize = {
		width: '',
		height: ''
	};

	// chat export
	let stylizedPdfExport = true;

	// Admin - Show Update Available Toast
	let showUpdateToast = true;
	let showChangelog = true;

	let showEmojiInCall = false;
	let voiceInterruption = false;
	let hapticFeedback = false;

	let webSearch = null;

	let iframeSandboxAllowSameOrigin = false;
	let iframeSandboxAllowForms = false;

	const toggleExpandDetails = () => {
		expandDetails = !expandDetails;
		saveSettings({ expandDetails });
	};

	const toggleCollapseCodeBlocks = () => {
		collapseCodeBlocks = !collapseCodeBlocks;
		saveSettings({ collapseCodeBlocks });
	};

	const toggleSplitLargeChunks = async () => {
		splitLargeChunks = !splitLargeChunks;
		saveSettings({ splitLargeChunks: splitLargeChunks });
	};

	const toggleHighContrastMode = async () => {
		highContrastMode = !highContrastMode;
		saveSettings({ highContrastMode: highContrastMode });
	};

	const togglePromptAutocomplete = async () => {
		promptAutocomplete = !promptAutocomplete;
		saveSettings({ promptAutocomplete: promptAutocomplete });
	};

	const togglesScrollOnBranchChange = async () => {
		scrollOnBranchChange = !scrollOnBranchChange;
		saveSettings({ scrollOnBranchChange: scrollOnBranchChange });
	};

	const toggleWidescreenMode = async () => {
		widescreenMode = !widescreenMode;
		saveSettings({ widescreenMode: widescreenMode });
	};

	const toggleChatBubble = async () => {
		chatBubble = !chatBubble;
		saveSettings({ chatBubble: chatBubble });
	};

	const toggleLandingPageMode = async () => {
		landingPageMode = landingPageMode === '' ? 'chat' : '';
		saveSettings({ landingPageMode: landingPageMode });
	};

	const toggleShowUpdateToast = async () => {
		showUpdateToast = !showUpdateToast;
		saveSettings({ showUpdateToast: showUpdateToast });
	};

	const toggleNotificationSound = async () => {
		notificationSound = !notificationSound;
		saveSettings({ notificationSound: notificationSound });
	};

	const toggleNotificationSoundAlways = async () => {
		notificationSoundAlways = !notificationSoundAlways;
		saveSettings({ notificationSoundAlways: notificationSoundAlways });
	};

	const toggleShowChangelog = async () => {
		showChangelog = !showChangelog;
		saveSettings({ showChangelog: showChangelog });
	};

	const toggleShowUsername = async () => {
		showUsername = !showUsername;
		saveSettings({ showUsername: showUsername });
	};

	const toggleEmojiInCall = async () => {
		showEmojiInCall = !showEmojiInCall;
		saveSettings({ showEmojiInCall: showEmojiInCall });
	};

	const toggleVoiceInterruption = async () => {
		voiceInterruption = !voiceInterruption;
		saveSettings({ voiceInterruption: voiceInterruption });
	};

	const toggleImageCompression = async () => {
		imageCompression = !imageCompression;
		saveSettings({ imageCompression });
	};

	const toggleHapticFeedback = async () => {
		hapticFeedback = !hapticFeedback;
		saveSettings({ hapticFeedback: hapticFeedback });
	};

	const toggleStylizedPdfExport = async () => {
		stylizedPdfExport = !stylizedPdfExport;
		saveSettings({ stylizedPdfExport: stylizedPdfExport });
	};

	const toggleUserLocation = async () => {
		userLocation = !userLocation;

		if (userLocation) {
			const position = await getUserPosition().catch((error) => {
				toast.error(error.message);
				return null;
			});

			if (position) {
				await updateUserInfo(localStorage.token, { location: position });
				toast.success($i18n.t('User location successfully retrieved.'));
			} else {
				userLocation = false;
			}
		}

		saveSettings({ userLocation });
	};

	const toggleTitleAutoGenerate = async () => {
		titleAutoGenerate = !titleAutoGenerate;
		saveSettings({
			title: {
				...$settings.title,
				auto: titleAutoGenerate
			}
		});
	};

	const toggleAutoFollowUps = async () => {
		autoFollowUps = !autoFollowUps;
		saveSettings({ autoFollowUps });
	};

	const toggleAutoTags = async () => {
		autoTags = !autoTags;
		saveSettings({ autoTags });
	};

	const toggleDetectArtifacts = async () => {
		detectArtifacts = !detectArtifacts;
		saveSettings({ detectArtifacts });
	};

	const toggleRichTextInput = async () => {
		richTextInput = !richTextInput;
		saveSettings({ richTextInput });
	};

	const toggleLargeTextAsFile = async () => {
		largeTextAsFile = !largeTextAsFile;
		saveSettings({ largeTextAsFile });
	};

	const toggleResponseAutoCopy = async () => {
		const permission = await navigator.clipboard
			.readText()
			.then(() => {
				return 'granted';
			})
			.catch(() => {
				return '';
			});

		console.log(permission);

		if (permission === 'granted') {
			responseAutoCopy = !responseAutoCopy;
			saveSettings({ responseAutoCopy: responseAutoCopy });
		} else {
			toast.error(
				$i18n.t(
					'Clipboard write permission denied. Please check your browser settings to grant the necessary access.'
				)
			);
		}
	};

	const toggleCopyFormatted = async () => {
		copyFormatted = !copyFormatted;
		saveSettings({ copyFormatted });
	};

	const toggleChangeChatDirection = async () => {
		if (chatDirection === 'auto') {
			chatDirection = 'LTR';
		} else if (chatDirection === 'LTR') {
			chatDirection = 'RTL';
		} else if (chatDirection === 'RTL') {
			chatDirection = 'auto';
		}
		saveSettings({ chatDirection });
	};

	const togglectrlEnterToSend = async () => {
		ctrlEnterToSend = !ctrlEnterToSend;
		saveSettings({ ctrlEnterToSend });
	};

	const updateInterfaceHandler = async () => {
		saveSettings({
			models: [defaultModelId],
			imageCompressionSize: imageCompressionSize
		});
	};

	const toggleWebSearch = async () => {
		webSearch = webSearch === null ? 'always' : null;
		saveSettings({ webSearch: webSearch });
	};

	const toggleIframeSandboxAllowSameOrigin = async () => {
		iframeSandboxAllowSameOrigin = !iframeSandboxAllowSameOrigin;
		saveSettings({ iframeSandboxAllowSameOrigin });
	};

	const toggleIframeSandboxAllowForms = async () => {
		iframeSandboxAllowForms = !iframeSandboxAllowForms;
		saveSettings({ iframeSandboxAllowForms });
	};

	onMount(async () => {
		titleAutoGenerate = $settings?.title?.auto ?? true;
		autoTags = $settings?.autoTags ?? true;
		autoFollowUps = $settings?.autoFollowUps ?? true;

		highContrastMode = $settings?.highContrastMode ?? false;

		detectArtifacts = $settings?.detectArtifacts ?? true;
		responseAutoCopy = $settings?.responseAutoCopy ?? false;

		showUsername = $settings?.showUsername ?? false;
		showUpdateToast = $settings?.showUpdateToast ?? true;
		showChangelog = $settings?.showChangelog ?? true;

		showEmojiInCall = $settings?.showEmojiInCall ?? false;
		voiceInterruption = $settings?.voiceInterruption ?? false;

		richTextInput = $settings?.richTextInput ?? true;
		promptAutocomplete = $settings?.promptAutocomplete ?? false;
		largeTextAsFile = $settings?.largeTextAsFile ?? false;
		copyFormatted = $settings?.copyFormatted ?? false;

		collapseCodeBlocks = $settings?.collapseCodeBlocks ?? false;
		expandDetails = $settings?.expandDetails ?? false;

		landingPageMode = $settings?.landingPageMode ?? '';
		chatBubble = $settings?.chatBubble ?? true;
		widescreenMode = $settings?.widescreenMode ?? false;
		splitLargeChunks = $settings?.splitLargeChunks ?? false;
		scrollOnBranchChange = $settings?.scrollOnBranchChange ?? true;
		chatDirection = $settings?.chatDirection ?? 'auto';
		userLocation = $settings?.userLocation ?? false;

		notificationSound = $settings?.notificationSound ?? true;
		notificationSoundAlways = $settings?.notificationSoundAlways ?? false;

		iframeSandboxAllowSameOrigin = $settings?.iframeSandboxAllowSameOrigin ?? false;
		iframeSandboxAllowForms = $settings?.iframeSandboxAllowForms ?? false;

		stylizedPdfExport = $settings?.stylizedPdfExport ?? true;

		hapticFeedback = $settings?.hapticFeedback ?? false;
		ctrlEnterToSend = $settings?.ctrlEnterToSend ?? false;

		imageCompression = $settings?.imageCompression ?? false;
		imageCompressionSize = $settings?.imageCompressionSize ?? { width: '', height: '' };

		defaultModelId = $settings?.models?.at(0) ?? '';
		if ($config?.default_models) {
			defaultModelId = $config.default_models.split(',')[0];
		}

		backgroundImageUrl = $settings?.backgroundImageUrl ?? null;
		webSearch = $settings?.webSearch ?? null;
	});
</script>

