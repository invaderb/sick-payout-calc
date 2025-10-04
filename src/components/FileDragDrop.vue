<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

const dropZone = ref<HTMLElement | null>(null)
const fileArray = defineModel<File[]>({
	default: () => [],
})
const fileSelect = ref<HTMLInputElement | null>(null)

function dropHandler(ev: DragEvent) {
	ev.preventDefault();

	[...ev.dataTransfer!.items].map((item) => {
		if (item.kind === 'file') {
			const file = item.getAsFile()
			fileArray.value.push(file!)
		}
		return null
	}).filter(Boolean)
}

function fileSelectEvent() {
	fileSelect.value?.click()
}

function fileSelected(event: Event) {
	const input = event.target as HTMLInputElement
	if (!input.files?.length) return

	[...input.files].map((file) => {
		fileArray.value.push(file);
	})
}

function removeFile(event: Event, file: File) {
	fileArray.value = fileArray.value.filter(f => f !== file);
	event?.stopPropagation()
}

function preventDefaults(e: DragEvent){
	e.preventDefault()
}

onMounted(() => {
	window.addEventListener('dragover', preventDefaults)
	window.addEventListener('drop', preventDefaults)
})

onBeforeUnmount(() => {
	window.removeEventListener('dragover', preventDefaults)
	window.removeEventListener('drop', preventDefaults)
})
</script>

<template>
	<input ref="fileSelect" id="file-select" type="file" name="files" @change="fileSelected" multiple="true" />
	<div ref="dropZone" id="drop-zone" @drop="dropHandler" @click="fileSelectEvent" :class="{'column': fileArray.length === 0}">
		<ul style="flex: 2;">
			<li v-for="(file) in fileArray" :key="file.name"

			>
				<span class="file-name">{{ file.name }}</span>
				
				<button class="button-stroked icon-button" @click="removeFile($event, file)" style="margin-left: 8px;">
					<svg width="24px" height="24px" viewBox="0 0 24 24" fill="none" stroke="red">
						<path d="M10 12V17" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M14 12V17" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M4 7H20" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M6 10V18C6 19.6569 7.34315 21 9 21H15C16.6569 21 18 19.6569 18 18V10" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
						<path d="M9 5C9 3.89543 9.89543 3 11 3H13C14.1046 3 15 3.89543 15 5V7H9V5Z" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
					</svg>
				</button>
			</li>
		</ul>
		<p style="text-align: center; flex: 1;">
			Click or Drag one or more files to this <i>drop zone</i>. <br />
			<svg width="48px" height="48px" viewBox="0 0 24 24" fill="none" stroke="currentcolor">
				<path d="M13.5 3H12H8C6.34315 3 5 4.34315 5 6V18C5 19.6569 6.34315 21 8 21H12M13.5 3L19 8.625M13.5 3V7.625C13.5 8.17728 13.9477 8.625 14.5 8.625H19M19 8.625V11.8125" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
				<path d="M17.5 21L17.5 15M17.5 15L20 17.5M17.5 15L15 17.5" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
			</svg>
		</p>
	</div>
</template>

<style>
#file-select {
	display: block;
	visibility: hidden;
	height: 0px;
	padding: 0px;
	margin: 0px;
}

#drop-zone {
	border: 2px dashed grey;
	width: 100%;
	min-height: 100px;
	padding: 16px;
	display: flex;
	justify-content: space-between;
	cursor: pointer;
}

.column {
	flex-direction: column;
	justify-content: center;
	align-items: center;
}

#drop-zone ul {
  list-style: none;
  padding: 0;
}

#drop-zone li {
  list-style: none;
}

.icon-button {
	padding: 8px 12px;
	margin: 0px;
	border-radius: 25px;
	color: var(--pico-mark-color);
}

.button-stroked {
	background-color: transparent;
	border-color: grey;
}
</style>