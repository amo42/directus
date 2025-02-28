<template>
	<v-notice v-if="!relationInfo" type="warning">{{ t('relationship_not_setup') }}</v-notice>
	<div v-else class="many-to-many">
		<template v-if="loading">
			<v-skeleton-loader
				v-for="n in clamp(totalItemCount - (page - 1) * limit, 1, limit)"
				:key="n"
				:type="totalItemCount > 4 ? 'block-list-item-dense' : 'block-list-item'"
			/>
		</template>

		<v-notice v-else-if="displayItems.length === 0">{{ t('no_items') }}</v-notice>

		<v-list v-else>
			<draggable
				:force-fallback="true"
				:model-value="displayItems"
				item-key="id"
				handle=".drag-handle"
				:disabled="!allowDrag"
				@update:model-value="sortItems($event)"
			>
				<template #item="{ element }">
					<v-list-item
						:class="{ deleted: element.$type === 'deleted' }"
						:dense="totalItemCount > 4"
						block
						clickable
						@click="editItem(element)"
					>
						<v-icon v-if="allowDrag" name="drag_handle" class="drag-handle" left @click.stop="() => {}" />
						<render-template
							:collection="relationInfo.junctionCollection.collection"
							:item="element"
							:template="templateWithDefaults"
						/>
						<div class="spacer" />
						<v-icon
							v-if="!disabled"
							:name="getDeselectIcon(element)"
							class="deselect"
							@click.stop="deleteItem(element)"
						/>
						<v-menu show-arrow placement="bottom-end">
							<template #activator="{ toggle }">
								<v-icon name="more_vert" clickable @click.stop="toggle" />
							</template>

							<v-list>
								<v-list-item clickable :href="getUrl(element)">
									<v-list-item-icon><v-icon name="launch" /></v-list-item-icon>
									<v-list-item-content>{{ t('open_file_in_tab') }}</v-list-item-content>
								</v-list-item>
								<v-list-item clickable :href="getUrl(element, true)">
									<v-list-item-icon><v-icon name="download" /></v-list-item-icon>
									<v-list-item-content>{{ t('download_file') }}</v-list-item-content>
								</v-list-item>
							</v-list>
						</v-menu>
					</v-list-item>
				</template>
			</draggable>
		</v-list>

		<div class="actions">
			<v-button v-if="enableCreate && createAllowed" :disabled="disabled" @click="showUpload = true">
				{{ t('upload_file') }}
			</v-button>
			<v-button v-if="enableSelect && selectAllowed" :disabled="disabled" @click="selectModalActive = true">
				{{ t('add_existing') }}
			</v-button>
			<v-pagination v-if="pageCount > 1" v-model="page" :length="pageCount" :total-visible="5" />
		</div>

		<drawer-item
			v-model:active="editModalActive"
			:disabled="disabled"
			:collection="relationInfo.junctionCollection.collection"
			:primary-key="currentlyEditing || '+'"
			:related-primary-key="relatedPrimaryKey || '+'"
			:junction-field="relationInfo.junctionField.field"
			:edits="editsAtStart"
			:circular-field="relationInfo.reverseJunctionField.field"
			@input="stageEdits"
		>
			<template #actions>
				<v-button
					v-if="currentlyEditing !== '+' && relationInfo.relatedCollection.collection === 'directus_files'"
					secondary
					rounded
					icon
					download
					:href="downloadUrl"
				>
					<v-icon name="download" />
				</v-button>
			</template>
		</drawer-item>

		<drawer-collection
			v-if="!disabled"
			v-model:active="selectModalActive"
			:collection="relationInfo.relatedCollection.collection"
			:selection="selectedPrimaryKeys"
			:filter="customFilter"
			multiple
			@input="onSelect"
		/>

		<v-dialog v-if="!disabled" v-model="showUpload">
			<v-card>
				<v-card-title>{{ t('upload_file') }}</v-card-title>
				<v-card-text>
					<v-upload multiple from-url :folder="folder" @input="onUpload" />
				</v-card-text>
				<v-card-actions>
					<v-button @click="showUpload = false">{{ t('done') }}</v-button>
				</v-card-actions>
			</v-card>
		</v-dialog>
	</div>
</template>

<script setup lang="ts">
import { useRelationM2M } from '@/composables/use-relation-m2m';
import { useRelationMultiple, RelationQueryMultiple, DisplayItem } from '@/composables/use-relation-multiple';
import { computed, ref, toRefs } from 'vue';
import { useI18n } from 'vue-i18n';
import DrawerItem from '@/views/private/components/drawer-item.vue';
import DrawerCollection from '@/views/private/components/drawer-collection.vue';
import Draggable from 'vuedraggable';
import { adjustFieldsForDisplays } from '@/utils/adjust-fields-for-displays';
import { get, clamp, isEmpty } from 'lodash';
import { usePermissionsStore } from '@/stores/permissions';
import { useUserStore } from '@/stores/user';
import { addTokenToURL } from '@/api';
import { getRootPath } from '@/utils/get-root-path';
import { getFieldsFromTemplate } from '@directus/shared/utils';
import { Filter } from '@directus/shared/types';

const props = withDefaults(
	defineProps<{
		value?: (number | string | Record<string, any>)[] | Record<string, any>;
		primaryKey: string | number;
		collection: string;
		field: string;
		template?: string | null;
		disabled?: boolean;
		enableCreate?: boolean;
		enableSelect?: boolean;
		folder?: string;
		limit?: number;
	}>(),
	{
		value: () => [],
		template: () => null,
		disabled: false,
		enableCreate: true,
		enableSelect: true,
		folder: undefined,
		limit: 15,
	}
);

const emit = defineEmits(['input']);
const { t } = useI18n();
const { collection, field, primaryKey, limit } = toRefs(props);
const { relationInfo } = useRelationM2M(collection, field);

const value = computed({
	get: () => props.value,
	set: (val) => {
		emit('input', val);
	},
});

const templateWithDefaults = computed(() => {
	if (!relationInfo.value) return null;

	if (props.template) return props.template;
	if (relationInfo.value.junctionCollection.meta?.display_template)
		return relationInfo.value.junctionCollection.meta?.display_template;

	let relatedDisplayTemplate = relationInfo.value.relatedCollection.meta?.display_template;
	if (relatedDisplayTemplate) {
		const regex = /({{.*?}})/g;
		const parts = relatedDisplayTemplate.split(regex).filter((p) => p);

		for (const part of parts) {
			if (part.startsWith('{{') === false) continue;
			const key = part.replace(/{{/g, '').replace(/}}/g, '').trim();
			const newPart = `{{${relationInfo.value.relation.field}.${key}}}`;

			relatedDisplayTemplate = relatedDisplayTemplate.replace(part, newPart);
		}

		return relatedDisplayTemplate;
	}

	return `{{${relationInfo.value.relation.field}.${relationInfo.value.relatedPrimaryKeyField.field}}}`;
});

const fields = computed(() =>
	adjustFieldsForDisplays(
		getFieldsFromTemplate(templateWithDefaults.value),
		relationInfo.value?.junctionCollection.collection ?? ''
	)
);

const page = ref(1);

const query = computed<RelationQueryMultiple>(() => ({
	fields: fields.value,
	limit: limit.value,
	page: page.value,
}));

const {
	update,
	remove,
	select,
	displayItems,
	totalItemCount,
	loading,
	selected,
	isItemSelected,
	localDelete,
	getItemEdits,
} = useRelationMultiple(value, query, relationInfo, primaryKey);

const pageCount = computed(() => Math.ceil(totalItemCount.value / limit.value));

const allowDrag = computed(
	() => totalItemCount.value <= limit.value && relationInfo.value?.sortField !== undefined && !props.disabled
);

function getDeselectIcon(item: DisplayItem) {
	if (item.$type === 'deleted') return 'settings_backup_restore';
	if (localDelete(item)) return 'delete';
	return 'close';
}

function sortItems(items: DisplayItem[]) {
	const sortField = relationInfo.value?.sortField;
	if (!sortField) return;

	const sortedItems = items.map((item, index) => ({
		...item,
		[sortField]: index,
	}));
	update(...sortedItems);
}

const selectedPrimaryKeys = computed(() => {
	if (!relationInfo.value) return [];
	const junctionField = relationInfo.value.junctionField.field;
	const relationPkField = relationInfo.value.relatedPrimaryKeyField.field;

	return selected.value.map((item) => item[junctionField][relationPkField]);
});

const editModalActive = ref(false);
const currentlyEditing = ref<string | number | null>(null);
const relatedPrimaryKey = ref<string | number | null>(null);
const selectModalActive = ref(false);
const editsAtStart = ref<Record<string, any>>({});

function editItem(item: DisplayItem) {
	if (!relationInfo.value) return;

	const relationPkField = relationInfo.value.relatedPrimaryKeyField.field;
	const junctionField = relationInfo.value.junctionField.field;
	const junctionPkField = relationInfo.value.junctionPrimaryKeyField.field;

	editsAtStart.value = getItemEdits(item);

	editModalActive.value = true;

	if (item?.$type === 'created' && !isItemSelected(item)) {
		currentlyEditing.value = null;
		relatedPrimaryKey.value = null;
	} else {
		currentlyEditing.value = get(item, [junctionPkField], null);
		relatedPrimaryKey.value = get(item, [junctionField, relationPkField], null);
	}
}

function stageEdits(item: Record<string, any>) {
	if (isEmpty(item)) return;

	update(item);
}

function deleteItem(item: DisplayItem) {
	if (
		page.value === Math.ceil(totalItemCount.value / limit.value) &&
		page.value !== Math.ceil((totalItemCount.value - 1) / limit.value)
	) {
		page.value = Math.max(1, page.value - 1);
	}

	remove(item);
}

const showUpload = ref(false);

function onUpload(files: Record<string, any>[]) {
	showUpload.value = false;
	if (files.length === 0 || !relationInfo.value) return;

	const fileIds = files.map((file) => file.id);

	select(fileIds);
}

function onSelect(selected: string[]) {
	select(selected.filter((id) => selectedPrimaryKeys.value.includes(id) === false));
}

const downloadUrl = computed(() => {
	if (relatedPrimaryKey.value === null || relationInfo.value?.relatedCollection.collection !== 'directus_files') return;
	return addTokenToURL(getRootPath() + `assets/${relatedPrimaryKey.value}?download`);
});

function getUrl(junctionRow: Record<string, any>, addDownload?: boolean) {
	const junctionField = relationInfo.value?.junctionField.field;
	if (!junctionField) return;

	const key = junctionRow[junctionField]?.id ?? junctionRow[junctionField] ?? null;
	if (!key) return null;

	if (addDownload) {
		return addTokenToURL(getRootPath() + `assets/${key}?download`);
	}

	return addTokenToURL(getRootPath() + `assets/${key}`);
}

const customFilter = computed(() => {
	const filter: Filter = {
		_and: [],
	};

	if (props.folder) {
		filter._and.push({
			folder: {
				id: { _eq: props.folder },
			},
		});
	}

	if (!relationInfo.value) return filter;

	const reverseRelation = `$FOLLOW(${relationInfo.value.junctionCollection.collection},${relationInfo.value.junctionField.field})`;

	const selectFilter: Filter = {
		[reverseRelation]: {
			_none: {
				[relationInfo.value.reverseJunctionField.field]: {
					_eq: props.primaryKey,
				},
			},
		},
	};

	if (selectedPrimaryKeys.value.length > 0)
		filter._and.push({
			[relationInfo.value.relatedPrimaryKeyField.field]: {
				_nin: selectedPrimaryKeys.value,
			},
		});

	if (props.primaryKey !== '+') filter._and.push(selectFilter);

	return filter;
});

const userStore = useUserStore();
const permissionsStore = usePermissionsStore();

const createAllowed = computed(() => {
	const admin = userStore.currentUser?.role.admin_access === true;
	if (admin) return true;

	const hasJunctionPermissions = !!permissionsStore.permissions.find(
		(permission) =>
			permission.action === 'create' && permission.collection === relationInfo.value?.junctionCollection.collection
	);

	const hasRelatedPermissions = !!permissionsStore.permissions.find(
		(permission) =>
			permission.action === 'create' && permission.collection === relationInfo.value?.relatedCollection.collection
	);

	return hasJunctionPermissions && hasRelatedPermissions;
});

const selectAllowed = computed(() => {
	const admin = userStore.currentUser?.role.admin_access === true;
	if (admin) return true;

	const hasJunctionPermissions = !!permissionsStore.permissions.find(
		(permission) =>
			permission.action === 'create' && permission.collection === relationInfo.value?.junctionCollection.collection
	);

	return hasJunctionPermissions;
});
</script>

<style lang="scss" scoped>
.v-list {
	--v-list-padding: 0 0 4px;

	.v-list-item.deleted {
		--v-list-item-border-color: var(--danger-25);
		--v-list-item-border-color-hover: var(--danger-50);
		--v-list-item-background-color: var(--danger-10);
		--v-list-item-background-color-hover: var(--danger-25);

		::v-deep(.v-icon) {
			color: var(--danger-75);
		}
	}
}

.actions {
	margin-top: 8px;
	display: flex;
	gap: 8px;

	.v-pagination {
		margin-left: auto;

		::v-deep(.v-button) {
			display: inline-flex;
		}
	}
}

.deselect {
	--v-icon-color: var(--foreground-subdued);
	margin-right: 4px;
	transition: color var(--fast) var(--transition);

	&:hover {
		--v-icon-color: var(--danger);
	}
}

.render-template {
	height: 100%;
}
</style>
