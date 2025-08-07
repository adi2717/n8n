<script lang="ts" setup>
import { useLoadingService } from '@/composables/useLoadingService';
import { useTelemetry } from '@/composables/useTelemetry';
import { useToast } from '@/composables/useToast';
import { SOURCE_CONTROL_PULL_MODAL_KEY, VIEWS } from '@/constants';
import { sourceControlEventBus } from '@/event-bus/source-control';
import EnvFeatureFlag from '@/features/env-feature-flag/EnvFeatureFlag.vue';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import {
	getPullPriorityByStatus,
	getStatusText,
	getStatusTheme,
	notifyUserAboutPullWorkFolderOutcome,
} from '@/utils/sourceControlUtils';
import { type SourceControlledFile, SOURCE_CONTROL_FILE_TYPE } from '@n8n/api-types';
import { N8nBadge, N8nButton, N8nLink, N8nText } from '@n8n/design-system';
import { useI18n } from '@n8n/i18n';
import type { EventBus } from '@n8n/utils/event-bus';
import groupBy from 'lodash/groupBy';
import orderBy from 'lodash/orderBy';
import { computed, ref, onMounted } from 'vue';
import { RouterLink, useRoute, useRouter } from 'vue-router';
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller';
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css';
import Modal from './Modal.vue';

type SourceControlledFileType = SourceControlledFile['type'];

const props = defineProps<{
	data: { eventBus: EventBus; status?: SourceControlledFile[] };
}>();

const telemetry = useTelemetry();
const loadingService = useLoadingService();
const toast = useToast();
const i18n = useI18n();
const sourceControlStore = useSourceControlStore();
const route = useRoute();
const router = useRouter();

// Reactive status state - starts with props data or empty, then loads fresh data
const status = ref<SourceControlledFile[]>(props.data.status || []);
const isLoading = ref(false);

// Load fresh source control status when modal opens
async function loadSourceControlStatus() {
	if (isLoading.value) return;

	isLoading.value = true;
	loadingService.startLoading();
	loadingService.setLoadingText(i18n.baseText('settings.sourceControl.loading.checkingForChanges'));

	try {
		const freshStatus = await sourceControlStore.getAggregatedStatus();

		if (!freshStatus.length) {
			toast.showMessage({
				title: 'No changes to pull',
				message: 'Everything is up to date',
				type: 'info',
			});
			// Close modal since there's nothing to show
			close();
			return;
		}

		status.value = freshStatus;
	} catch (error) {
		toast.showError(error, i18n.baseText('error'));
		close();
	} finally {
		isLoading.value = false;
		loadingService.stopLoading();
		loadingService.setLoadingText(i18n.baseText('genericHelpers.loading'));
	}
}

const sortedFiles = computed(() =>
	orderBy(
		status.value,
		[({ status }) => getPullPriorityByStatus(status), ({ name }) => name.toLowerCase()],
		['desc', 'asc'],
	),
);

const groupedFilesByType = computed<
	Partial<Record<SourceControlledFileType, SourceControlledFile[]>>
>(() => groupBy(sortedFiles.value, 'type'));

type ItemsList = Array<
	{ type: 'render-title'; title: string; id: SourceControlledFileType } | SourceControlledFile
>;

const ITEM_TITLES: Record<Exclude<SourceControlledFileType, 'file'>, string> = {
	[SOURCE_CONTROL_FILE_TYPE.workflow]: 'Workflows',
	[SOURCE_CONTROL_FILE_TYPE.credential]: 'Credentials',
	[SOURCE_CONTROL_FILE_TYPE.variables]: 'Variables',
	[SOURCE_CONTROL_FILE_TYPE.tags]: 'Tags',
	[SOURCE_CONTROL_FILE_TYPE.folders]: 'Folders',
} as const;

const files = computed<ItemsList>(() =>
	[
		SOURCE_CONTROL_FILE_TYPE.workflow,
		SOURCE_CONTROL_FILE_TYPE.credential,
		SOURCE_CONTROL_FILE_TYPE.variables,
		SOURCE_CONTROL_FILE_TYPE.tags,
		SOURCE_CONTROL_FILE_TYPE.folders,
	].reduce<ItemsList>((acc, fileType) => {
		if (!groupedFilesByType.value[fileType]) {
			return acc;
		}

		acc.push({
			type: 'render-title',
			title: ITEM_TITLES[fileType],
			id: fileType,
		});

		acc.push(...groupedFilesByType.value[fileType]);
		return acc;
	}, []),
);

function close() {
	// Navigate back in history to maintain proper browser navigation
	// The global route watcher will handle closing the modal
	router.back();
}

async function pullWorkfolder() {
	loadingService.startLoading(i18n.baseText('settings.sourceControl.loading.pull'));
	close();

	try {
		const status = await sourceControlStore.pullWorkfolder(true);

		await notifyUserAboutPullWorkFolderOutcome(status, toast);

		sourceControlEventBus.emit('pull');
	} catch (error) {
		toast.showError(error, 'Error');
	} finally {
		loadingService.stopLoading();
	}
}

function openDiffModal(id: string) {
	telemetry.track('User clicks compare workflows', {
		workflow_id: id,
		context: 'source_control_pull',
	});

	// Only update route - modal will be opened by route watcher
	void router.push({
		query: {
			...route.query,
			diff: id,
			direction: 'pull',
		},
	});
}

// Load data when modal opens
onMounted(() => {
	// Only load fresh data if we don't have any initial data
	if (!props.data.status || props.data.status.length === 0) {
		void loadSourceControlStatus();
	}
});
</script>

<template>
	<Modal
		v-if="!isLoading"
		width="500px"
		:title="i18n.baseText('settings.sourceControl.modals.pull.title')"
		:event-bus="data.eventBus"
		:name="SOURCE_CONTROL_PULL_MODAL_KEY"
		:before-close="close"
	>
		<template #content>
			<N8nText tag="div" class="mb-xs">
				{{ i18n.baseText('settings.sourceControl.modals.pull.description') }}
				<br />
				<N8nLink :to="i18n.baseText('settings.sourceControl.docs.using.pushPull.url')">
					{{ i18n.baseText('settings.sourceControl.modals.push.description.learnMore') }}
				</N8nLink>
			</N8nText>
			<div :class="$style.container">
				<DynamicScroller
					ref="scroller"
					:items="files"
					:min-item-size="47"
					:class="$style.scroller"
					style="max-height: 440px"
				>
					<template #default="{ item, index, active }">
						<div
							v-if="item.type === 'render-title'"
							:class="$style.listHeader"
							data-test-id="pull-modal-item-header"
						>
							<N8nText bold>{{ item.title }}</N8nText>
						</div>
						<DynamicScrollerItem
							v-else
							:item="item"
							:active="active"
							:size-dependencies="[item.name]"
							:data-index="index"
						>
							<div :class="$style.listItem" data-test-id="pull-modal-item">
								<div :class="$style.itemName">
									<RouterLink
										v-if="item.type === 'credential'"
										target="_blank"
										:to="{ name: VIEWS.CREDENTIALS, params: { credentialId: item.id } }"
									>
										<N8nText>{{ item.name }}</N8nText>
									</RouterLink>
									<RouterLink
										v-else-if="item.type === 'workflow'"
										target="_blank"
										:to="{ name: VIEWS.WORKFLOW, params: { name: item.id } }"
									>
										<N8nText>{{ item.name }}</N8nText>
									</RouterLink>
									<N8nText v-else>{{ item.name }}</N8nText>
								</div>
								<div :class="$style.itemActions">
									<N8nBadge :theme="getStatusTheme(item.status)" :class="$style.listBadge">
										{{ getStatusText(item.status) }}
									</N8nBadge>
									<EnvFeatureFlag name="SOURCE_CONTROL_WORKFLOW_DIFF">
										<N8nIconButton
											v-if="item.type === SOURCE_CONTROL_FILE_TYPE.workflow"
											icon="file-diff"
											type="secondary"
											:class="$style.diffButton"
											@click="openDiffModal(item.id)"
										/>
									</EnvFeatureFlag>
								</div>
							</div>
						</DynamicScrollerItem>
					</template>
				</DynamicScroller>
			</div>
		</template>

		<template #footer>
			<div :class="$style.footer">
				<N8nButton type="tertiary" class="mr-2xs" @click="close">
					{{ i18n.baseText('settings.sourceControl.modals.pull.buttons.cancel') }}
				</N8nButton>
				<N8nButton type="primary" data-test-id="force-pull" @click="pullWorkfolder">
					{{ i18n.baseText('settings.sourceControl.modals.pull.buttons.save') }}
				</N8nButton>
			</div>
		</template>
	</Modal>
</template>

<style module lang="scss">
.container {
	overflow-wrap: break-word;
	padding-right: 8px;
}

.scroller {
	margin-right: -8px;
}

.filesList {
	list-style: inside;
	margin-top: var(--spacing-3xs);
	padding-left: var(--spacing-2xs);

	li {
		margin-top: var(--spacing-3xs);
	}
}

.listHeader {
	padding-top: 16px;
	padding-bottom: 12px;
	height: 47px;
	margin-right: 8px;
}

.listItem {
	display: flex;
	align-items: center;
	padding-bottom: 8px;
	margin-right: 8px;

	&::before {
		display: block;
		content: '';
		width: 5px;
		height: 5px;
		background-color: var(--color-foreground-xdark);
		border-radius: 100%;
		margin: 7px 8px 6px 2px;
		flex-shrink: 0;
	}
}

.itemName {
	flex: 1;
	min-width: 0;
	margin-right: 8px;

	a,
	span {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		display: block;
	}
}

.itemActions {
	display: flex;
	align-items: center;
	gap: 8px;
	flex-shrink: 0;
}

.listBadge {
	align-self: center;
	white-space: nowrap;
	height: 30px;
}

.diffButton {
	flex-shrink: 0;
}

.footer {
	display: flex;
	flex-direction: row;
	justify-content: flex-end;
}
</style>
