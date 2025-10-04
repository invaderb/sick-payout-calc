<script setup lang="ts">
import { ref, watch } from 'vue';

interface Props {
	open?: boolean;
	title?: string;
}

const props = withDefaults(defineProps<Props>(), {
	open: false,
	title: 'Dialog'
});

const emit = defineEmits<{
	close: []
	confirm: []
}>();

const dialogRef = ref<HTMLDialogElement | null>(null);

// Sync the dialog open state with the prop
watch(() => props.open, (newVal) => {
	if (newVal) {
		dialogRef.value?.showModal();
	} else {
		dialogRef.value?.close();
	}
});

function handleClose() {
	emit('close');
}

// function handleConfirm() {
// 	emit('confirm');
// 	emit('close');
// }
</script>

<template>
	<dialog ref="dialogRef" @close="handleClose">
		<article>
			<header>
				<button aria-label="Close" rel="prev" @click="handleClose"></button>
				<h3>{{ title }}</h3>
			</header>
			<main>
				<slot></slot>
			</main>
			<footer>
				<button class="secondary" @click="handleClose">Close</button>
				<!-- <button @click="handleConfirm">Confirm</button> -->
			</footer>
		</article>
	</dialog>
</template>