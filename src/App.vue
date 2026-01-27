<script setup lang="ts">
import { RouterView } from 'vue-router'
import HelpDialog from './components/HelpDialog.vue'
import { ref } from 'vue';

const isOpen = ref(false);

function handleConfirm() {
	isOpen.value = false;
}

</script>

<template>
	<header>
		<nav>
			<ul>
				<li style="display: flex; align-items: center; gap: 8px;">
					<svg width="40px" height="40px" viewBox="0 0 24 24" fill="none" style="margin-bottom: 1rem;">
						<path
							d="M21 12C21 16.714 21 19.0711 19.682 20.5355C18.364 22 16.2426 22 12 22C7.75736 22 5.63604 22 4.31802 20.5355C3 19.0711 3 16.714 3 12C3 7.28595 3 4.92893 4.31802 3.46447C5.63604 2 7.75736 2 12 2C16.2426 2 18.364 2 19.682 3.46447C20.5583 4.43821 20.852 5.80655 20.9504 8"
							stroke="currentColor" stroke-width="1.5" stroke-linecap="round" />
						<path
							d="M7 8C7 7.53501 7 7.30252 7.05111 7.11177C7.18981 6.59413 7.59413 6.18981 8.11177 6.05111C8.30252 6 8.53501 6 9 6H15C15.465 6 15.6975 6 15.8882 6.05111C16.4059 6.18981 16.8102 6.59413 16.9489 7.11177C17 7.30252 17 7.53501 17 8C17 8.46499 17 8.69748 16.9489 8.88823C16.8102 9.40587 16.4059 9.81019 15.8882 9.94889C15.6975 10 15.465 10 15 10H9C8.53501 10 8.30252 10 8.11177 9.94889C7.59413 9.81019 7.18981 9.40587 7.05111 8.88823C7 8.69748 7 8.46499 7 8Z"
							stroke="currentColor" stroke-width="1.5" />
						<circle cx="8" cy="13" r="1" fill="currentColor" />
						<circle cx="8" cy="17" r="1" fill="currentColor" />
						<circle cx="12" cy="13" r="1" fill="currentColor" />
						<circle cx="12" cy="17" r="1" fill="currentColor" />
						<circle cx="16" cy="13" r="1" fill="currentColor" />
						<circle cx="16" cy="17" r="1" fill="currentColor" />
					</svg>
					<h1>Sick Payout Calculator</h1>
				</li>
			</ul>
			<ul>
				<!-- <li>
					<RouterLink to="/">Home</RouterLink>
				</li> -->
				<li>
					<!-- help -->
					<button class="icon-button button-stroked column"
						style="display: flex; flex-direction: row; gap: 8px" @click="isOpen = true">
						<svg fill="currentcolor" width="24px" height="24px" viewBox="0 0 32 32" preserveAspectRatio="xMidYMid">
							<path
								d="M16.000,32.000 C7.178,32.000 -0.000,24.822 -0.000,16.000 C-0.000,7.178 7.178,-0.000 16.000,-0.000 C24.822,-0.000 32.000,7.178 32.000,16.000 C32.000,24.822 24.822,32.000 16.000,32.000 ZM29.410,20.000 L22.917,20.000 C22.216,21.209 21.209,22.216 20.000,22.917 L20.000,29.410 C24.507,28.064 28.064,24.507 29.410,20.000 ZM16.000,10.000 C12.691,10.000 10.000,12.691 10.000,16.000 C10.000,19.309 12.691,22.000 16.000,22.000 C19.309,22.000 22.000,19.309 22.000,16.000 C22.000,12.691 19.309,10.000 16.000,10.000 ZM16.000,30.000 C16.681,30.000 17.345,29.935 18.000,29.840 L18.000,23.737 C17.359,23.902 16.692,24.000 16.000,24.000 C15.308,24.000 14.641,23.902 14.000,23.737 L14.000,29.840 C14.655,29.935 15.319,30.000 16.000,30.000 ZM12.000,29.410 L12.000,22.917 C10.791,22.216 9.784,21.209 9.083,20.000 L2.590,20.000 C3.936,24.507 7.493,28.064 12.000,29.410 ZM2.000,16.000 C2.000,16.681 2.065,17.345 2.160,18.000 L8.263,18.000 C8.097,17.359 8.000,16.692 8.000,16.000 C8.000,15.308 8.097,14.641 8.263,14.000 L2.160,14.000 C2.065,14.655 2.000,15.319 2.000,16.000 ZM2.590,12.000 L9.083,12.000 C9.784,10.791 10.791,9.784 12.000,9.082 L12.000,2.590 C7.493,3.936 3.936,7.493 2.590,12.000 ZM16.000,2.000 C15.319,2.000 14.655,2.065 14.000,2.160 L14.000,8.263 C14.641,8.097 15.308,8.000 16.000,8.000 C16.692,8.000 17.359,8.097 18.000,8.263 L18.000,2.160 C17.345,2.065 16.681,2.000 16.000,2.000 ZM20.000,2.590 L20.000,9.082 C21.209,9.784 22.216,10.791 22.918,12.000 L29.410,12.000 C28.064,7.493 24.507,3.936 20.000,2.590 ZM23.737,14.000 C23.903,14.641 24.000,15.308 24.000,16.000 C24.000,16.692 23.903,17.359 23.737,18.000 L29.840,18.000 C29.935,17.345 30.000,16.681 30.000,16.000 C30.000,15.319 29.935,14.655 29.840,14.000 L23.737,14.000 Z" />
						</svg>
						Help
					</button>
				</li>
			</ul>
			<!-- <RouterLink to="/about">About</RouterLink> -->
		</nav>
		<div>
		</div>
	</header>

	<RouterView />

	<HelpDialog :open="isOpen" title="Help" @close="isOpen = false" @confirm="handleConfirm">
		<section>
			<p>
				This tool helps calculate the payout for unused sick leave based on your company's PTO policies. <br /><br />
				To get started, upload your Excel file containing employee activity data by clicking or dragging the file into the designated drop zone on the page. <br /><br />
				Once selected click the "Load and Process" button to process the data and display any errors or logs in the sidebar. If there are no errors, you can proceed to export the calculated sick leave payout data to a new Excel file by clicking the "Save Data to File" button. <br /><br />
				Make sure to review the exported data for accuracy before using it for payroll processing. If you encounter any issues or have questions, feel free to reach out for support.
			</p>
			<div>
				Requirements: <br />
				• There are 4 required files that must be provided in the Excel format
				<ul>
					<li>1. Employee Sick Activity File - This file should contain a list of employee sick activities.</li>
					<li>2. PTO File - This file should detail the various PTO plans available, including accrual rates and carry-over values etc.</li>
					<li>3. Care Logs File - This file should contain the caregivers rates over the last pay period.</li>
					<li>4. Employee Hire Date File - This file should provide the current balance of PTO for each employee.</li>
				</ul>
				• Each filename must start with the following text (case insensitive):
				<ul>
					<li>"all_activity" for the Employee Sick Activity File</li>
					<li>"pto" for the PTO File</li>
					<li>"care_logs" for the Care Logs File</li>
					<li>"hire_date" for the Employee Hire Date File</li>
				</ul>
			</div>
		</section>
	</HelpDialog>
</template>
