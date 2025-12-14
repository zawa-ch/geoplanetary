<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<div class="_gaps">
	<div :class="$style.header">
		<MkSelect v-model="type" :items="typeDef" :class="$style.typeSelect">
		</MkSelect>
		<button v-if="draggable" class="drag-handle _button" :class="$style.dragHandle">
			<i class="ti ti-menu-2"></i>
		</button>
		<button v-if="draggable" class="_button" :class="$style.remove" @click="removeSelf">
			<i class="ti ti-x"></i>
		</button>
	</div>

	<div v-if="type === 'and' || type === 'or'" class="_gaps">
		<Sortable v-model="v.values" tag="div" class="_gaps" itemKey="id" handle=".drag-handle" :group="{ name: 'roleFormula' }" :animation="150" :swapThreshold="0.5">
			<template #item="{element}">
				<div :class="$style.item">
					<!-- divが無いとエラーになる https://github.com/SortableJS/vue.draggable.next/issues/189 -->
					<RolesEditorFormula :modelValue="element" draggable @update:modelValue="updated => valuesItemUpdated(updated)" @remove="removeItem(element)"/>
				</div>
			</template>
		</Sortable>
		<MkButton rounded style="margin: 0 auto;" @click="addValue"><i class="ti ti-plus"></i> {{ i18n.ts.add }}</MkButton>
	</div>

	<div v-else-if="type === 'not'" :class="$style.item">
		<RolesEditorFormula v-model="v.value"/>
	</div>

	<MkInput v-else-if="type === 'createdLessThan' || type === 'createdMoreThan'" v-model="v.sec" type="number">
		<template #suffix>sec</template>
	</MkInput>

	<MkInput v-else-if="type === 'loggedInMoreThanOrEq' || type === 'loggedInLessThanOrEq'" v-model="v.day" type="number">
		<template #suffix>day</template>
	</MkInput>

	<MkInput v-else-if="['usernameEntropyMoreThanOrEq', 'usernameEntropyLessThanOrEq', 'usernameEntropyMeanMoreThanOrEq', 'usernameEntropyMeanLessThanOrEq'].includes(type)" v-model="v.value" :min="0" :step="0.1" type="number">
		<template #suffix>bit</template>
	</MkInput>

	<MkInput v-else-if="['followersLessThanOrEq', 'followersMoreThanOrEq', 'followingLessThanOrEq', 'followingMoreThanOrEq', 'notesLessThanOrEq', 'notesMoreThanOrEq', 'tagCountIs', 'tagCountMoreThanOrEq', 'tagCountLessThanOrEq', 'fieldCountIs', 'fieldCountMoreThanOrEq', 'fieldCountLessThan'].includes(type)" v-model="v.value" type="number">
	</MkInput>

	<MkInput v-else-if="['usernameMatchOf', 'nameMatchOf', 'hostMatchOf', 'hasTagMatchOf', 'emailMatchOf', 'hasFieldNameMatchOf', 'hasFieldValueMatchOf'].includes(type)" v-model="v.pattern" type="text">
		<template #caption>{{ i18n.ts._role.patternEditDescription }}</template>
	</MkInput>

	<MkSelect v-else-if="type === 'roleAssignedTo'" v-model="v.roleId" :items="assignedToDef">
	</MkSelect>

	<div v-else-if="['avatarLikelyBlurhash', 'bannerLikelyBlurhash'].includes(type)">
		<MkInput v-model="v.hash" type="text">
			<template #label>{{ i18n.ts._role.hash }}</template>
		</MkInput>
		<MkInput v-model="v.diff" type="number">
			<template #label>{{ i18n.ts._role.allowDifference }}</template>
		</MkInput>
	</div>
</div>
</template>

<script lang="ts" setup>
import { computed, defineAsyncComponent, ref, watch } from 'vue';
import type { GetMkSelectValueTypesFromDef, MkSelectItem } from '@/components/MkSelect.vue';
import { genId } from '@/utility/id.js';
import MkInput from '@/components/MkInput.vue';
import MkSelect from '@/components/MkSelect.vue';
import MkButton from '@/components/MkButton.vue';
import { i18n } from '@/i18n.js';
import { deepClone } from '@/utility/clone.js';
import { rolesCache } from '@/cache.js';

const Sortable = defineAsyncComponent(() => import('vuedraggable').then(x => x.default));

const emit = defineEmits<{
	(ev: 'update:modelValue', value: any): void;
	(ev: 'remove'): void;
}>();

const props = defineProps<{
	modelValue: any;
	draggable?: boolean;
}>();

const v = ref(deepClone(props.modelValue));

const roles = await rolesCache.fetch();

watch(() => props.modelValue, () => {
	if (JSON.stringify(props.modelValue) === JSON.stringify(v.value)) return;
	v.value = deepClone(props.modelValue);
}, { deep: true });

watch(v, () => {
	emit('update:modelValue', v.value);
}, { deep: true });

const typeDef = [
	{ label: i18n.ts._role._condition.isLocal, value: 'isLocal' },
	{ label: i18n.ts._role._condition.isRemote, value: 'isRemote' },
	{ label: i18n.ts._role._condition.isFederated, value: 'isFederated' },
	{ label: i18n.ts._role._condition.isSubscribing, value: 'isSubscribing' },
	{ label: i18n.ts._role._condition.isPublishing, value: 'isPublishing' },
	{ label: i18n.ts._role._condition.isForeign, value: 'isForeign' },
	{ label: i18n.ts._role._condition.isSuspended, value: 'isSuspended' },
	{ label: i18n.ts._role._condition.isLocked, value: 'isLocked' },
	{ label: i18n.ts._role._condition.isBot, value: 'isBot' },
	{ label: i18n.ts._role._condition.isCat, value: 'isCat' },
	{ label: i18n.ts._role._condition.isExplorable, value: 'isExplorable' },
	{ label: i18n.ts._role._condition.isMfaEnabled, value: 'isMfaEnabled' },
	{ label: i18n.ts._role._condition.isSecurityKeyAvailable, value: 'isSecurityKeyAvailable' },
	{ label: i18n.ts._role._condition.isUsingPwlessLogin, value: 'isUsingPwlessLogin' },
	{ label: i18n.ts._role._condition.isNoCrawle, value: 'isNoCrawle' },
	{ label: i18n.ts._role._condition.isNoAI, value: 'isNoAI' },
	{ label: i18n.ts._role._condition.roleAssignedTo, value: 'roleAssignedTo' },
	{ label: i18n.ts._role._condition.usernameMatchOf, value: 'usernameMatchOf' },
	{ label: i18n.ts._role._condition.usernameEntropyMoreThanOrEq, value: 'usernameEntropyMoreThanOrEq' },
	{ label: i18n.ts._role._condition.usernameEntropyLessThanOrEq, value: 'usernameEntropyLessThanOrEq' },
	{ label: i18n.ts._role._condition.usernameEntropyMeanMoreThanOrEq, value: 'usernameEntropyMeanMoreThanOrEq' },
	{ label: i18n.ts._role._condition.usernameEntropyMeanLessThanOrEq, value: 'usernameEntropyMeanLessThanOrEq' },
	{ label: i18n.ts._role._condition.nameMatchOf, value: 'nameMatchOf' },
	{ label: i18n.ts._role._condition.hostMatchOf, value: 'hostMatchOf' },
	{ label: i18n.ts._role._condition.nameIsDefault, value: 'nameIsDefault' },
	{ label: i18n.ts._role._condition.emailVerified, value: 'emailVerified' },
	{ label: i18n.ts._role._condition.emailMatchOf, value: 'emailMatchOf' },
	{ label: i18n.ts._role._condition.createdLessThan, value: 'createdLessThan' },
	{ label: i18n.ts._role._condition.createdMoreThan, value: 'createdMoreThan' },
	{ label: i18n.ts._role._condition.loggedInLessThanOrEq, value: 'loggedInLessThanOrEq' },
	{ label: i18n.ts._role._condition.loggedInMoreThanOrEq, value: 'loggedInMoreThanOrEq' },
	{ label: i18n.ts._role._condition.followersLessThanOrEq, value: 'followersLessThanOrEq' },
	{ label: i18n.ts._role._condition.followersMoreThanOrEq, value: 'followersMoreThanOrEq' },
	{ label: i18n.ts._role._condition.followingLessThanOrEq, value: 'followingLessThanOrEq' },
	{ label: i18n.ts._role._condition.followingMoreThanOrEq, value: 'followingMoreThanOrEq' },
	{ label: i18n.ts._role._condition.notesLessThanOrEq, value: 'notesLessThanOrEq' },
	{ label: i18n.ts._role._condition.notesMoreThanOrEq, value: 'notesMoreThanOrEq' },
	{ label: i18n.ts._role._condition.avatarUnset, value: 'avatarUnset' },
	{ label: i18n.ts._role._condition.avatarLikelyBlurhash, value: 'avatarLikelyBlurhash' },
	{ label: i18n.ts._role._condition.bannerUnset, value: 'bannerUnset' },
	{ label: i18n.ts._role._condition.bannerLikelyBlurhash, value: 'bannerLikelyBlurhash' },
	{ label: i18n.ts._role._condition.hasTags, value: 'hasTags' },
	{ label: i18n.ts._role._condition.tagCountIs, value: 'tagCountIs' },
	{ label: i18n.ts._role._condition.tagCountMoreThanOrEq, value: 'tagCountMoreThanOrEq' },
	{ label: i18n.ts._role._condition.tagCountLessThanOrEq, value: 'tagCountLessThanOrEq' },
	{ label: i18n.ts._role._condition.hasTagMatchOf, value: 'hasTagMatchOf' },
	{ label: i18n.ts._role._condition.hasFields, value: 'hasFields' },
	{ label: i18n.ts._role._condition.fieldCountIs, value: 'fieldCountIs' },
	{ label: i18n.ts._role._condition.fieldCountMoreThanOrEq, value: 'fieldCountMoreThanOrEq' },
	{ label: i18n.ts._role._condition.fieldCountLessThanOrEq, value: 'fieldCountLessThanOrEq' },
	{ label: i18n.ts._role._condition.hasFieldNameMatchOf, value: 'hasFieldNameMatchOf' },
	{ label: i18n.ts._role._condition.hasFieldValueMatchOf, value: 'hasFieldValueMatchOf' },
	{ label: i18n.ts._role._condition.and, value: 'and' },
	{ label: i18n.ts._role._condition.or, value: 'or' },
	{ label: i18n.ts._role._condition.not, value: 'not' },
] as const satisfies MkSelectItem[];

const type = computed<GetMkSelectValueTypesFromDef<typeof typeDef>>({
	get: () => v.value.type,
	set: (t) => {
		if (t === 'and') v.value.values = [];
		if (t === 'or') v.value.values = [];
		if (t === 'not') v.value.value = { id: genId(), type: 'isRemote' };
		if (t === 'roleAssignedTo') v.value.roleId = '';
		if (t === 'usernameMatchOf') v.value.pattern = '';
		if (t === 'usernameEntropyMoreThanOrEq') v.value.value = 47;
		if (t === 'usernameEntropyLessThanOrEq') v.value.value = 47;
		if (t === 'usernameEntropyMeanMoreThanOrEq') v.value.value = 4.7;
		if (t === 'usernameEntropyMeanLessThanOrEq') v.value.value = 4.7;
		if (t === 'nameMatchOf') v.value.pattern = '';
		if (t === 'hostMatchOf') v.value.pattern = '';
		if (t === 'emailMatchOf') v.value.pattern = '';
		if (t === 'createdLessThan') v.value.sec = 86400;
		if (t === 'createdMoreThan') v.value.sec = 86400;
		if (t === 'loggedInMoreThanOrEq') v.value.day = 10;
		if (t === 'loggedInLessThanOrEq') v.value.day = 10;
		if (t === 'followersLessThanOrEq') v.value.value = 10;
		if (t === 'followersMoreThanOrEq') v.value.value = 10;
		if (t === 'followingLessThanOrEq') v.value.value = 10;
		if (t === 'followingMoreThanOrEq') v.value.value = 10;
		if (t === 'notesLessThanOrEq') v.value.value = 10;
		if (t === 'notesMoreThanOrEq') v.value.value = 10;
		if (t === 'avatarLikelyBlurhash') v.value.hash = '';
		if (t === 'avatarLikelyBlurhash') v.value.diff = 0;
		if (t === 'bannerLikelyBlurhash') v.value.hash = '';
		if (t === 'bannerLikelyBlurhash') v.value.diff = 0;
		if (t === 'tagCountIs') v.value.value = 10;
		if (t === 'tagCountMoreThanOrEq') v.value.value = 10;
		if (t === 'tagCountLessThanOrEq') v.value.value = 10;
		if (t === 'hasTagMatchOf') v.value.pattern = '';
		if (t === 'fieldCountIs') v.value.value = 4;
		if (t === 'fieldCountMoreThanOrEq') v.value.value = 4;
		if (t === 'fieldCountLessThan') v.value.value = 4;
		if (t === 'hasFieldNameMatchOf') v.value.pattern = '';
		if (t === 'hasFieldValueMatchOf') v.value.pattern = '';
		v.value.type = t;
	},
});

const assignedToDef = computed(() => roles.filter(r => r.target === 'manual').map(r => ({ label: r.name, value: r.id })) satisfies MkSelectItem[]);

function addValue() {
	v.value.values.push({ id: genId(), type: 'isRemote' });
}

function valuesItemUpdated(item) {
	const i = v.value.values.findIndex(_item => _item.id === item.id);
	v.value.values[i] = item;
}

function removeItem(item) {
	v.value.values = v.value.values.filter(_item => _item.id !== item.id);
}

function removeSelf() {
	emit('remove');
}
</script>

<style lang="scss" module>
.header {
	display: flex;
}

.typeSelect {
	flex: 1;
}

.dragHandle {
	cursor: move;
	margin-left: 10px;
}

.remove {
	margin-left: 10px;
}

.item {
	border: solid 2px var(--MI_THEME-divider);
	border-radius: var(--MI-radius);
	padding: 12px;

	&:hover {
		border-color: var(--MI_THEME-accent);
	}
}
</style>
