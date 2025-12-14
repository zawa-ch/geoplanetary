<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<div class="_gaps">
	<div :class="$style.header">
		<MkSelect v-model="type" :class="$style.typeSelect" :items="formulaTypeDef(child)"/>
		<button v-if="draggable" class="drag-handle _button" :class="$style.dragHandle">
			<i class="ti ti-menu-2"></i>
		</button>
		<button v-if="draggable" class="_button" :class="$style.remove" @click="removeSelf">
			<i class="ti ti-x"></i>
		</button>
	</div>

	<div v-if="type === 'and' || type === 'or'" class="_gaps">
		<Sortable
			v-model="values" tag="div" class="_gaps" itemKey="key" handle=".drag-handle"
			:group="{ name: 'prohibitedNoteFormula' }" :animation="150" :swapThreshold="0.5"
		>
			<template #item="{ element }">
				<div :class="$style.item">
					<!-- divが無いとエラーになる https://github.com/SortableJS/vue.draggable.next/issues/189 -->
					<ProhibitedNoteFormula
						:modelValue="element.value" child draggable
						@update:modelValue="updated => valuesItemUpdated(element.key, updated)" @remove="removeItem(element)"
					/>
				</div>
			</template>
		</Sortable>
		<MkButton rounded style="margin: 0 auto;" @click="addValue">
			<i class="ti ti-plus"></i> {{ i18n.ts.add }}
		</MkButton>
	</div>

	<div v-else-if="type === 'not'" :class="$style.item">
		<ProhibitedNoteFormula v-model="subformula" child/>
	</div>

	<MkSelect
		v-else-if="type === 'roleAssignedTo'" v-model="roleId"
		:items="roles.map((i) => { return { value: i.id, label: i.name } })"
	/>

	<MkInput v-else-if="type === 'hasFileMD5Is'" v-model="md5hash" type="text"/>

	<MkInput
		v-else-if="['fileTotalSizeMoreThanOrEq', 'fileTotalSizeLessThan', 'hasFileSizeMoreThanOrEq', 'hasFileSizeLessThan'].includes(type)"
		v-model="size" type="number"
	>
		<template #suffix>byte</template>
	</MkInput>

	<MkInput
		v-else-if="['mentionCountIs', 'mentionCountMoreThanOrEq', 'mentionCountLessThan', 'fileCountIs', 'fileCountMoreThanOrEq', 'fileCountLessThan', 'hashtagCountIs', 'hashtagCountMoreThanOrEq', 'hashtagCountLessThan'].includes(type)"
		v-model="count" type="number"
	/>

	<MkTextarea v-else-if="['textMatchOf', 'hasHashtagMatchOf'].includes(type)" v-model="pattern" type="text">
		<template #caption>{{ i18n.ts._prohibitedNote.patternEditDescription }}</template>
	</MkTextarea>

	<div v-else-if="type === 'hasLikelyBlurhash'">
		<MkInput v-model="blurhash" type="text">
			<template #label>{{ i18n.ts._prohibitedNote.hash }}</template>
		</MkInput>
		<MkInput v-model="blurhashdiff" type="number">
			<template #label>{{ i18n.ts._prohibitedNote.allowDifference }}</template>
		</MkInput>
	</div>

	<div :class="$style.caption">
		<slot name="caption"></slot>
	</div>
</div>
</template>

<script lang="ts" setup>
import * as Misskey from 'misskey-js';
import { computed, defineAsyncComponent, ref, watch } from 'vue';
import MkInput from '@/components/MkInput.vue';
import MkTextarea from '@/components/MkTextarea.vue';
import MkSelect from '@/components/MkSelect.vue';
import MkButton from '@/components/MkButton.vue';
import { i18n } from '@/i18n.js';
import { deepClone } from '@/utility/clone.js';
import { rolesCache } from '@/cache.js';
import { genId } from '@/utility/id';

const Sortable = defineAsyncComponent(() => import('vuedraggable').then(x => x.default));

const roles = await rolesCache.fetch();

const emit = defineEmits<{
	(ev: 'update:modelValue', value: Misskey.entities.ProhibitedNoteFormulaValue | null): void;
	(ev: 'remove'): void;
}>();

const props = defineProps<{
	modelValue: Misskey.entities.ProhibitedNoteFormulaValue | null;
	draggable?: boolean;
	child?: boolean;
}>();

const formulaTypeDef = (child: boolean) => {
	return (!child ? [
		{ value: 'disable', label: i18n.ts._prohibitedNote.disable },
	] : []).concat([
		{ value: 'false', label: i18n.ts._prohibitedNote._condition.false_value },
		{ value: 'true', label: i18n.ts._prohibitedNote._condition.true_value },
		{ value: 'and', label: i18n.ts._prohibitedNote._condition.and },
		{ value: 'or', label: i18n.ts._prohibitedNote._condition.or },
		{ value: 'not', label: i18n.ts._prohibitedNote._condition.not },
		{ value: 'roleAssignedTo', label: i18n.ts._prohibitedNote._condition.roleAssignedTo },
		{ value: 'hasText', label: i18n.ts._prohibitedNote._condition.hasText },
		{ value: 'textMatchOf', label: i18n.ts._prohibitedNote._condition.textMatchOf },
		{ value: 'hasMentions', label: i18n.ts._prohibitedNote._condition.hasMentions },
		{ value: 'mentionCountIs', label: i18n.ts._prohibitedNote._condition.mentionCountIs },
		{ value: 'mentionCountMoreThanOrEq', label: i18n.ts._prohibitedNote._condition.mentionCountMoreThanOrEq },
		{ value: 'mentionCountLessThan', label: i18n.ts._prohibitedNote._condition.mentionCountLessThan },
		{ value: 'isReply', label: i18n.ts._prohibitedNote._condition.isReply },
		{ value: 'isQuoted', label: i18n.ts._prohibitedNote._condition.isQuoted },
		{ value: 'hasFiles', label: i18n.ts._prohibitedNote._condition.hasFiles },
		{ value: 'fileCountIs', label: i18n.ts._prohibitedNote._condition.fileCountIs },
		{ value: 'fileCountMoreThanOrEq', label: i18n.ts._prohibitedNote._condition.fileCountMoreThanOrEq },
		{ value: 'fileCountLessThan', label: i18n.ts._prohibitedNote._condition.fileCountLessThan },
		{ value: 'fileTotalSizeMoreThanOrEq', label: i18n.ts._prohibitedNote._condition.fileTotalSizeMoreThanOrEq },
		{ value: 'fileTotalSizeLessThan', label: i18n.ts._prohibitedNote._condition.fileTotalSizeLessThan },
		{ value: 'hasFileSizeMoreThanOrEq', label: i18n.ts._prohibitedNote._condition.hasFileSizeMoreThanOrEq },
		{ value: 'hasFileSizeLessThan', label: i18n.ts._prohibitedNote._condition.hasFileSizeLessThan },
		{ value: 'hasFileMD5Is', label: i18n.ts._prohibitedNote._condition.hasFileMD5Is },
		{ value: 'hasBrowserInsafe', label: i18n.ts._prohibitedNote._condition.hasBrowserInsafe },
		{ value: 'hasPictures', label: i18n.ts._prohibitedNote._condition.hasPictures },
		{ value: 'hasLikelyBlurhash', label: i18n.ts._prohibitedNote._condition.hasLikelyBlurhash },
		{ value: 'hasHashtags', label: i18n.ts._prohibitedNote._condition.hasHashtags },
		{ value: 'hashtagCountIs', label: i18n.ts._prohibitedNote._condition.hashtagCountIs },
		{ value: 'hashtagCountMoreThanOrEq', label: i18n.ts._prohibitedNote._condition.hashtagCountMoreThanOrEq },
		{ value: 'hashtagCountLessThan', label: i18n.ts._prohibitedNote._condition.hashtagCountLessThan },
		{ value: 'hasHashtagMatchOf', label: i18n.ts._prohibitedNote._condition.hasHashtagMatchOf },
	]);
};

function cloneModelValue() {
	return deepClone(props.modelValue);
}

const formula = ref(cloneModelValue());
const subValueKeys = ref<string[]>([]);
if (['and', 'or'].some(t => formula.value && (t === formula.value.type))) {
	(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values.forEach(() => { subValueKeys.value.push(genId()); });
}

watch(() => props.modelValue, () => {
	if (JSON.stringify(props.modelValue) === JSON.stringify(formula.value)) return;
	subValueKeys.value = [];
	(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values.forEach(() => { subValueKeys.value.push(genId()); });
	formula.value = cloneModelValue();
}, { deep: true });

watch(formula, () => {
	emit('update:modelValue', formula.value);
}, { deep: true });

const type = computed({
	get: () => formula.value?.type ?? 'disable',
	set: (t) => {
		switch (t) {
			case 'true':
			case 'false': {
				formula.value = { type: t };
				break;
			}
			case 'and':
			case 'or': {
				formula.value = { type: t, values: [] };
				break;
			}
			case 'not': {
				formula.value = { type: t, value: { type: 'false' } };
				break;
			}
			case 'hasText':
			case 'hasMentions':
			case 'isReply':
			case 'isQuoted':
			case 'hasFiles':
			case 'hasBrowserInsafe':
			case 'hasPictures':
			case 'hasHashtags': {
				formula.value = { type: t };
				break;
			}
			case 'textMatchOf':
			case 'hasHashtagMatchOf': {
				formula.value = { type: t, pattern: '' };
				break;
			}
			case 'mentionCountIs':
			case 'mentionCountMoreThanOrEq':
			case 'mentionCountLessThan':
			case 'fileCountIs':
			case 'fileCountMoreThanOrEq':
			case 'fileCountLessThan':
			case 'hashtagCountIs':
			case 'hashtagCountMoreThanOrEq':
			case 'hashtagCountLessThan': {
				formula.value = { type: t, value: 3 };
				break;
			}
			case 'fileTotalSizeMoreThanOrEq':
			case 'fileTotalSizeLessThan':
			case 'hasFileSizeMoreThanOrEq':
			case 'hasFileSizeLessThan': {
				formula.value = { type: t, size: 10240 };
				break;
			}
			case 'roleAssignedTo': {
				formula.value = { type: t, roleId: '' };
				break;
			}
			case 'hasFileMD5Is': {
				formula.value = { type: t, hash: '' };
				break;
			}
			case 'hasLikelyBlurhash': {
				formula.value = { type: t, hash: '', diff: 0 };
				break;
			}
			case 'disable': {
				formula.value = null;
			}
		}
	},
});

const values = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values.map((v, i) => { return { key: subValueKeys.value[i], value: v }; }),
	set: (nv) => {
		subValueKeys.value = nv.map(v => v.key);
		(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values = nv.map(v => v.value);
	},
});
const subformula = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaNot).value,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaNot).value = nv,
});
const roleId = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaAssignsRole).roleId,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaAssignsRole).roleId = nv,
});
const md5hash = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaMD5HashMatch).hash,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaMD5HashMatch).hash = nv,
});
const size = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaSizeComp).size,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaSizeComp).size = nv,
});
const count = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaCountComp).value,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaCountComp).value = nv,
});
const pattern = computed({
	get: () => {
		const p = (formula.value as Misskey.entities.ProhibitedNoteFormulaPatternMatch).pattern;
		return Array.isArray(p) ? p.join('\n') : p;
	},
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaPatternMatch).pattern = nv.split('\n'),
});
const blurhash = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaBlurhashLikely).hash,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaBlurhashLikely).hash = nv,
});
const blurhashdiff = computed({
	get: () => (formula.value as Misskey.entities.ProhibitedNoteFormulaBlurhashLikely).diff,
	set: (nv) => (formula.value as Misskey.entities.ProhibitedNoteFormulaBlurhashLikely).diff = nv,
});

function addValue() {
	(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values.push({ type: 'false' });
	subValueKeys.value.push(genId());
}

function valuesItemUpdated(key: string, value: Misskey.entities.ProhibitedNoteFormulaValue | null) {
	const wi = subValueKeys.value.findIndex(k => k === key);
	if (value) {
		(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values[wi] = value;
	} else {
		(formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values = (formula.value as Misskey.entities.ProhibitedNoteFormulaLogics).values.filter((_, i) => i !== wi);
		subValueKeys.value = subValueKeys.value.filter((_, i) => i !== wi);
	}
}

function removeItem(item) {
	values.value = values.value.filter((v) => v.key !== item.key);
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

.caption {
	font-size: 0.85em;
	padding: 8px 0 0 0;
	color: color(from var(--MI_THEME-fg) srgb r g b /0.75);

	&:empty {
		display: none;
	}
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
