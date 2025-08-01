<template>
	<div ref="objectEditor" class="flex flex-col gap-2" @paste="pasteObj">
		<div v-for="(value, key, index) in obj" :key="index" class="flex gap-2">
			<BuilderInput
				placeholder="Property"
				:modelValue="key"
				@update:modelValue="(val: string) => replaceKey(key, val)" />
			<BuilderInput
				placeholder="Value"
				:modelValue="value"
				@update:modelValue="(val: string) => updateObjectValue(key, val)" />
			<BuilderButton
				class="flex-shrink-0 text-xs"
				variant="subtle"
				icon="x"
				@click="deleteObjectKey(key as string)"></BuilderButton>
		</div>
		<BuilderButton variant="subtle" label="Add" @click="addObjectKey"></BuilderButton>
		<p class="rounded-sm bg-surface-gray-1 p-2 text-xs text-ink-gray-7" v-show="description">
			<span v-html="description"></span>
		</p>
	</div>
</template>
<script setup lang="ts">
import { mapToObject, replaceMapKey } from "@/utils/helpers";
import { nextTick, ref } from "vue";

const props = defineProps<{
	obj: Record<string, string>;
	description?: string;
}>();

const emit = defineEmits({
	"update:obj": (obj: Record<string, string>) => true,
});

const addObjectKey = async () => {
	const map = new Map(Object.entries(props.obj));
	map.set("", "");
	emit("update:obj", mapToObject(map));
	await nextTick();
	const inputs = objectEditor.value?.querySelectorAll("input");
	if (inputs) {
		const lastInput = inputs[inputs.length - 2];
		lastInput.focus();
	}
};

const updateObjectValue = (key: string, value: string) => {
	const map = new Map(Object.entries(props.obj));
	map.set(key, value);
	emit("update:obj", mapToObject(map));
};

const replaceKey = (oldKey: string, newKey: string) => {
	const map = new Map(Object.entries(props.obj));
	emit("update:obj", mapToObject(replaceMapKey(map, oldKey, newKey)));
};

const deleteObjectKey = (key: string) => {
	const map = new Map(Object.entries(props.obj));
	map.delete(key);
	emit("update:obj", mapToObject(map));
};

const objectEditor = ref<HTMLElement | null>(null);

const pasteObj = (e: ClipboardEvent) => {
	const text = e.clipboardData?.getData("text/plain");
	if (text) {
		if (!text.includes(":")) return;
		const map = new Map(Object.entries(props.obj));
		try {
			const objString = text.match(/{[^{}]+}/)?.[0];
			if (!objString) throw new Error("Invalid object");
			const obj = new Function("return " + objString)();
			if (typeof obj === "object") {
				for (const [key, value] of Object.entries(obj)) {
					map.set(key, value as string);
				}
				map.delete("");
			}
		} catch (e) {
			const lines = text.split(";");
			for (const line of lines) {
				const [key, value] = line.split(":");
				if (!key || !value) continue;
				map.set(key.trim(), value.replace(";", "").trim());
			}
			map.delete("");
		}
		emit("update:obj", mapToObject(map));
	}
};
</script>
