<script setup lang="ts">
import { nextTick, ref } from 'vue';
import FileDragDrop from '@/components/FileDragDrop.vue';
import { utils, read, write } from 'xlsx';
import { z } from "zod";

interface ActivityInfo {
	caregiver: string
	date: string
	startTime: string
	endTime: string
	durationHours: number
	rawText: string
}

interface FullData extends ActivityInfo {
	employeeId: string | null
	hireDate: string
	requirementDate: string
	isQualified: boolean
	plan: string
	planLimit: number
	canUseLimit: number
	ytdUsed: number
	balance: number
	approvedHours: number | null
	rate: number | null
	totalPay: number | null
}

interface CareLog {
	"Caregiver Name": string
	"Official Clock In": string
	"Official Clock Out": string
	"Pay Rate Amount": number
	"Caregiver ID": string // same as Employee ID in PTOLog
	"is_used"?: boolean
	[key: string]: string | number | Date | boolean | undefined
}

const careLogSchema = z.object({
	"Caregiver Name": z.string(),
	"Official Clock In": z.string().or(z.date()),
	"Official Clock Out": z.string().or(z.date()),
	"Pay Rate Amount": z.number(),
	"Caregiver ID": z.string()
})

interface ActivityLog {
	Description: string
}

const activityLogSchema = z.object({
	Description: z.string()
})

interface PTOLog {
	"Employee Name": string
	"Employee ID": string
	"Accrue Thru": string
	"Year Ending": string
	"Plan Name": string
	"Accrual Rate": number
	"Carry Over": number
	"Accrued": number
	"Used": number
	"Balance": number
}

const ptoLogSchema = z.object({
	"Employee ID": z.string(),
	"Plan Name": z.string(),
	"Used": z.number(),
	"Balance": z.number()
})

interface HireLog {
	"Employee Name": string
	"Employee ID": string
	"Hire Date": string
	"Years Service": number
}

const hireLogSchema = z.object({
	"Employee Name": z.string(),
	"Employee ID": z.string(),
	"Hire Date": z.string().or(z.date()),
	"Years Service": z.number()
})

type ErrorLog = {
	message: string;
	type: string;
}

const files = ref<File[]>([]);
const errors = ref<ErrorLog[]>([]);
const activityData = <ActivityInfo[]>[];
let careLogs = <CareLog[]>[];
let ptoData = <PTOLog[]>[];
let hireData = <HireLog[]>[];
let fullData: FullData[] = [];
const loading = ref(false);
const hasErrors = ref(false);
const gotData = ref(false);
const SICK_POLICY = {
	"SICK POLICY": 40,
	"LA SICK POLICY": 48,
	"SICK LIVE IN": 72
};

async function getData() {
	loading.value = true;
	if (files.value.length === 0) {
		loading.value = false;
		return;
	}
	// now process the files in order
	for (let i = 0; i < files.value.length; i++) {
		// else if (files.value[i].type === 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' || files.value[i].type === 'application/vnd.ms-excel') {
 		if (files.value[i]!.name.endsWith('.xlsx') || files.value[i]!.name.endsWith('.xls')) {
			let type = '';
			if (files.value[i]!.name.startsWith('care_logs')) {
				type = 'care_logs';
			} else if (files.value[i]!.name.startsWith('all_activity')) {
				type = 'sick_activity';
			} else if (files.value[i]!.name.startsWith('pto')) {
				type = 'pto';
			} else if (files.value[i]!.name.startsWith('hire_date')) {
				type = 'hire_date';
			}
			await processExcel(files.value[i]!, type);
		}
	}
	processData();
	gotData.value = true;
	
	loading.value = false;
}

function parseActivityContent(content: string): ActivityInfo | null {
	try {
		if (content.length < 1) {
			errors.value.push({
				message: `Content length to short nothing to parse`,
				type: 'error'
			})
			hasErrors.value = true;
			endOfErrors();
		}
		// Extract date: "on 09/09/2025"
		const dateMatch = content.match(/on (\d{2}\/\d{2}\/\d{4})/i)
		const date = dateMatch ? dateMatch[1] : ''

		// Extract start time: "from 11:00 PM PDT"
		const startMatch = content.match(/from (\d{1,2}:\d{2} [AP]M)/i)
		const startTime = startMatch ? startMatch[1] : ''

		// Extract end time: "to 07:00 AM PDT"
		const endMatch = content.match(/to (\d{1,2}:\d{2} [AP]M)/i)
		const endTime = endMatch ? endMatch[1] : ''

		// Extract caregiver: "assigned to caregiver Karen Smith"
		const caregiverMatch = content.match(/assigned to caregiver ([^*)\n]+?)(?:\s*\*|\s*\))/i)
		const caregiver = caregiverMatch ? caregiverMatch[1]!.trim() : ''
		if (!caregiver) {
			errors.value.push({
				message: `Unable to parse the caregivers name from all_activity in "${content}", check for any special characters or missing name`,
				type: 'error'
			})
			hasErrors.value = true;
			endOfErrors();
		}
		if (date && startTime && endTime && caregiver) {
			// Calculate duration
			const durationHours = calculateDuration(date, startTime, endTime)
			return {
				date,
				startTime,
				endTime,
				caregiver,
				durationHours,
				rawText: content
			}
		}
	} catch (error) {
		console.error('Error parsing shift:', error)
	}

	return null
}

function calculateDuration(date: string, startTime: string, endTime: string): number {
	try {
		// Parse the date (MM/DD/YYYY)
		const [month, day, year] = date.split('/').map(Number)
		if (!month || !day || !year) throw new Error('Invalid date format')
		// Parse start time
		const startDate = parseDateTime(year, month, day, startTime)

		// Parse end time
		let endDate = parseDateTime(year, month, day, endTime)

		// If end time is earlier than start time, it's the next day
		if (endDate < startDate) {
			endDate = new Date(endDate.getTime() + 24 * 60 * 60 * 1000)
		}

		// Calculate difference in hours
		const diffMs = endDate.getTime() - startDate.getTime()
		const diffHours = diffMs / (1000 * 60 * 60)

		return Math.round(diffHours * 100) / 100 // Round to 2 decimal places
	} catch (error) {
		console.error('Error calculating duration:', error)
		return 0
	}
}

function parseDateTime(year: number, month: number, day: number, timeStr: string): Date {
	const timeMatch = timeStr.match(/(\d{1,2}):(\d{2})\s*(AM|PM)/i)

	if (!timeMatch) {
		throw new Error(`Invalid time format: ${timeStr}`)
	}

	let hours = parseInt(timeMatch[1]!)
	const minutes = parseInt(timeMatch[2]!)
	const isPM = timeMatch[3]!.toUpperCase() === 'PM'

	// Convert to 24-hour format
	if (isPM && hours !== 12) {
		hours += 12
	} else if (!isPM && hours === 12) {
		hours = 0
	}

	return new Date(year, month - 1, day, hours, minutes)
}
/**
 * 
 * @param file File
 * @param type activity, care_logs, pto, hire_date, sick_file
 */
async function processExcel(file: File, type: string): Promise<void> {
	return new Promise((resolve, reject) => {
		const reader = new FileReader();
		
		reader.onload = (e: ProgressEvent<FileReader>) => {
			try {
				e.preventDefault();
				const data = new Uint8Array(e.target!.result as ArrayBuffer);
				const workbook = read(data, { type: 'array', cellDates: true, dateNF: 'mm/dd/yyyy;@' });
				const firstSheetName = workbook.SheetNames[0];
				const worksheet = workbook.Sheets[firstSheetName!];
				const cellRefs = Object.keys(worksheet!).filter(k => !k.startsWith('!'));
    
				if (cellRefs.length > 0) {
					const cellAddresses = cellRefs.map(ref => utils.decode_cell(ref));
					const rows = cellAddresses.map(addr => addr.r);
					const cols = cellAddresses.map(addr => addr.c);
					
					const correctedRange = {
						s: { r: Math.min(...rows), c: Math.min(...cols) },
						e: { r: Math.max(...rows), c: Math.max(...cols) }
					};
					
					worksheet!['!ref'] = utils.encode_range(correctedRange);
				}
				const jsonData = utils.sheet_to_json(worksheet!, { raw: true });
				switch(type) {
					case 'care_logs':
						try {
							careLogSchema.parse(jsonData[0])	
						} catch (error) {
							if (error instanceof z.ZodError) {
								let message = '';
								for (let i = 0; i < error.issues.length; i++) {
									message += `Column: ${error.issues[i]!.path} - missing or invalid data\n`;
								}
								errors.value.push({
									message: `Care Log Schema invalid: \n${message}`,
									type: 'error',
								});
								return;
							}
						}
						const processedData = (jsonData as CareLog[]).map((row: CareLog) => {
							// Convert any Date objects to ISO strings
							Object.keys(row).forEach((key: string) => {
								if (row[key] instanceof Date) {
									// format to MM/DD/YYYY
									const month = (row[key].getMonth() + 1).toString().padStart(2, '0');
									const day = row[key].getDate().toString().padStart(2, '0');
									const year = row[key].getFullYear();
									row[key] = `${month}/${day}/${year}`;
								}
							});
							
							return row;
						});

						careLogs = processedData as CareLog[];
						break;
					case 'sick_activity':
						try {
							activityLogSchema.parse(jsonData[0])	
						} catch (error) {
							if (error instanceof z.ZodError) {
								let message = '';
								for (let i = 0; i < error.issues.length; i++) {
									message += `Column: ${error.issues[i]!.path} - missing or invalid data\n`;
								}
								errors.value.push({
									message: `All Activity Schema invalid: \n${message}`,
									type: 'error',
								});
								return;
							}
						}
						for (let i = 0; i < jsonData.length; i++) {
							const parsedDescription = parseActivityContent((jsonData[i] as ActivityLog)['Description'] || '');
							const data: ActivityInfo = {
								date: parsedDescription?.date || '',
								startTime: parsedDescription?.startTime || '',
								endTime: parsedDescription?.endTime || '',
								caregiver: parsedDescription?.caregiver || '',
								durationHours: parsedDescription?.durationHours || 0,
								rawText: (jsonData[i] as ActivityLog)['Description'] || ''
							}
							activityData.push(data);
						}
						break;
					case 'pto':
						try {
							ptoLogSchema.parse(jsonData[0])	
						} catch (error) {
							if (error instanceof z.ZodError) {
								let message = '';
								for (let i = 0; i < error.issues.length; i++) {
									message += "Column: " + error.issues[i]!.path + ' - missing or invalid data\n';
								}
								errors.value.push({
									message: `PTO Log Schema invalid: \n${message}`,
									type: 'error',
								});
								return;
							}
								return;
						}
						ptoData = jsonData as PTOLog[];
						break;
					case 'hire_date':
						try {
							hireLogSchema.parse(jsonData[0])	
						} catch (error) {
							if (error instanceof z.ZodError) {
								let message = '';
								for (let i = 0; i < error.issues.length; i++) {
									message += "Column: " + error.issues[i]!.path + ' - missing or invalid data\n';
								}
								errors.value.push({
									message: `Hire Date Schema invalid: \n${message}`,
									type: 'error',
								});
								return;
							}
						}
						hireData = jsonData as HireLog[];
						break;
					default:
						// do nothing
						console.error('Unknown file type:', type);
						break;
				}
				resolve();
			} catch (error) {
				console.error('Error processing Excel:', error);
				reject(error);
			}
		};
		
		reader.onerror = () => {
			reject(new Error('Failed to read Excel file'));
		};
		
		reader.readAsArrayBuffer(file);
	});
}

function processData() {
	fullData = [];
	
	// check that we have some data on each of the required arrays
	if (activityData.length === 0) {
		errors.value.push({
			message: 'No activity data found, please check the file name starts with all_activity and is the correct format',
			type: "error"
		});
		hasErrors.value = true;
		endOfErrors();
		return;
	}
	if (careLogs.length === 0) {
		errors.value.push({
			message: 'No care log data found, please check the file name starts with care_logs and is the correct format',
			type: 'error'
		});
		hasErrors.value = true;
		endOfErrors();
		return;
	}
	if (ptoData.length === 0) {
		errors.value.push({
			message: 'No PTO data found, please check the file name starts with pto and is the correct format',
			type: 'error'
		});
		hasErrors.value = true;
		endOfErrors();
		return;
	}
	if (hireData.length === 0) {
		errors.value.push({
			message: 'No hire date data found, please check the file name starts with hire_date and is the correct format',
			type: 'error'
		});
		hasErrors.value = true;
		endOfErrors();
		return;
	}
	// loop over the sick activity data and match dat from other files to build a full record
	for (let i = 0; i < activityData.length; i++) {
		const activity = activityData[i] as ActivityInfo;
		if (!activity.caregiver) {
			console.warn('no caregiver name', activity.rawText);
			continue;
		}
		// merge the activity record to the full record data
		const fullEntry: FullData = {
			...activity,
			rate: null,
			employeeId: null,
			hireDate: '',
			plan: '',
			planLimit: 0,
			ytdUsed: 0,
			canUseLimit: 0,
			balance: 0,
			requirementDate: '',
			isQualified: false,
			approvedHours: 0,
			totalPay: 0
		};
		// loop through the care logs and update the rate and employee id by matching the activity caregiver name
		for (let j = 0; j < careLogs.length; j++) {
			const care = careLogs[j];
			const careGiverName = care!["Caregiver Name"];
			// need to clean up the care-log caregiver name if they have an appended * in the name
			const careName = careGiverName.split(' *');
			const cname = careName.length > 1 ? careName[0] : careGiverName;
			if (!careLogs[j]?.is_used && activity.caregiver === cname && (
				(care!["Official Clock In"] && activity.date === care!["Official Clock In"].split('T')[0]) || 
				(care!["Official Clock Out"] && activity.date === care!["Official Clock Out"].split('T')[0]))
			) {
				fullEntry.rate = care!["Pay Rate Amount"];
				fullEntry.employeeId = care!["Caregiver ID"];
				careLogs[j]!['is_used'] = true;
				break;
			}
		}
		// check for employee rate or id is defined
		if (fullEntry.rate === null || fullEntry.employeeId === null) {
			errors.value.push({
				message: `No matching caregiver ${fullEntry.caregiver} in care_logs by name for activity: ${activity.rawText}. check all_activity and care_logs files.`,
				type: 'warn'
			});
			hasErrors.value = true;
			endOfErrors();
			continue;
		}
		// loop through the Hire Data to find the hire date and get the full name to match against the PTO data
		for (let k = 0; k < hireData.length; k++) {
			const hire = hireData[k];
			if (hire!["Employee ID"] && fullEntry.employeeId === hire!["Employee ID"]) {
				fullEntry.hireDate = hire!["Hire Date"];
				// calculate the requirement date based on the hire date + 90 days
				const hireDate = new Date(fullEntry.hireDate);
				const requirementDate = new Date(hireDate.getTime() + (90 * 24 * 60 * 60 * 1000));
				const month = (requirementDate.getMonth() + 1).toString().padStart(2, '0');
				const day = requirementDate.getDate().toString().padStart(2, '0');
				const year = requirementDate.getFullYear();
				fullEntry.requirementDate = `${month}/${day}/${year}`;
				// check if the activity date is after the requirement date
				const activityDate = new Date(fullEntry.date);
				fullEntry.isQualified = activityDate >= requirementDate;
				break;
			}
		}
		// check that there is a hire date
		if (!fullEntry.hireDate) {
			errors.value.push({
				message: `No hire date found for caregiver ${fullEntry.caregiver}, employee ID's didn't match: employee ID: ${fullEntry.employeeId}. check hire_date and care_logs files`,
				type: 'warn'
			});
			hasErrors.value = true;
			endOfErrors();
			continue;
		}
		// check if the hire date qualifies
		if (!fullEntry.isQualified) {
			const hireDateTimeAdd90 = new Date(fullEntry.hireDate).getTime() + (90 * 24 * 60 * 60 * 1000);
			const activityDateTime = new Date(fullEntry.date).getTime();
			const remainingDays = Math.ceil((hireDateTimeAdd90 - activityDateTime) / (1000 * 60 * 60 * 24));
			errors.value.push({
				message: `Caregiver ${fullEntry.caregiver} hire date is not past 90 days from the activity date no need to calculate PTO; Hire Date ${fullEntry.hireDate} | remaining days ${remainingDays}`,
				type: 'warn'
			});
			hasErrors.value = true;
			endOfErrors();
			continue;
		}
		
		// loop through the PTO data to find the plan, accrual rate, carry over, pay period accrued, pay period used, balance
		for (let l = 0; l < ptoData.length; l++) {
			const pto = ptoData[l];
			// ensure the employee name is fully uppercase and has no comma
			if (pto!['Employee ID'] && fullEntry.employeeId === pto!['Employee ID']) {
				fullEntry.plan = pto!["Plan Name"];
				fullEntry.planLimit = SICK_POLICY[fullEntry.plan as keyof typeof SICK_POLICY] || 0;
				fullEntry.ytdUsed = pto!["Used"]; // YTD used
				fullEntry.balance = pto!["Balance"];
				break;
			}
		}
		// if no plan found then log an error
		if (!fullEntry.plan) {
			errors.value.push({
				message: `No PTO plan found for caregiver ${fullEntry.caregiver}, employee ID's didn't match: employee ID: ${fullEntry.employeeId}. check pto and care_log files`,
				type: 'error'
			});
			hasErrors.value = true;
			endOfErrors();
			continue;
		}

		// calculate the approved hours based on the limit of the plan and the balance what ever is lower unless they are not qualified then 0
		if (fullEntry.isQualified) {
			// need to check if this caregiver already another activityData in the same pay period and if so, update the ytdUsed and balance before calculating approved hours
			// loop in reverse order to get the most recent and if found break
			for (let m = fullData.length - 1; m >= 0; m--) {
				if (fullData[m]!.employeeId === fullEntry.employeeId) {
					// also if fullData[i].date and starttime and end time are the same as fullEntry then skip
					if (fullData[m]!.date === fullEntry.date && fullData[m]!.startTime === fullEntry.startTime && fullData[m]!.endTime === fullEntry.endTime) {
						continue;
					}
					// update the ytdUsed and balance on the nextActivity
					fullEntry.ytdUsed = fullEntry.ytdUsed + (fullData[m]!.approvedHours || 0);
					fullEntry.balance = fullEntry.balance - (fullData[m]!.approvedHours || 0);
					continue;
				}
			}
			// calculate the approved hours based on the plan limit, the ytd used and the balance
			const availableHours = fullEntry.planLimit - fullEntry.ytdUsed;
			fullEntry.canUseLimit = availableHours > 0 ? availableHours : 0;
			const availableBalance = Math.min(fullEntry.balance, fullEntry.canUseLimit);
			fullEntry.approvedHours = Math.min(fullEntry.durationHours, availableBalance);
		}
		
		// calculate the total pay based on the approved hours and the rate
		if (fullEntry.approvedHours !== null && fullEntry.rate !== null) {
			fullEntry.totalPay = Math.round(fullEntry.approvedHours * fullEntry.rate * 100) / 100;
		}
		if(fullEntry.rate && fullEntry.approvedHours) {
			fullData.push(fullEntry);
		}
	}
}

/**
 * Save data to the sick_file.xlsx
 */
function saveData() {
	const headers = [
		'caregiver',
		'employeeId',
		'rawText',
		'startTime',
		'endTime',
		'durationHours',
		'date',
		'hireDate',
		'requirementDate',
		'isQualified',
		'plan',
		'planLimit',
		'ytdUsed',
		'canUseLimit',
		'balance',
		'approvedHours',
		'rate',
		'totalPay'
	];
	const totalPaySum = fullData.reduce((sum, row) => sum + (row.totalPay || 0), 0);

	// Add a total row to your data
	const dataWithTotal = [
		...fullData,
		{
			caregiver: 'TOTAL',
			employeeId: '',
			rawText: '',
			startTime: '',
			endTime: '',
			durationHours: '',
			date: '',
			hireDate: '',
			requirementDate: '',
			isQualified: '',
			plan: '',
			planLimit: '',
			ytdUsed: '',
			balance: '',
			approvedHours: '',
			rate: '',
			totalPay: totalPaySum
		}
	];
	// sheet js create a new workbook with the full data and download it as sick_file.xlsx
	const wb = utils.book_new();
	const ws = utils.json_to_sheet(dataWithTotal, {header: headers});
	utils.book_append_sheet(wb, ws, 'Sick Data');
	const wbout = write(wb, { bookType: 'xlsx', type: 'array' });
	const blob = new Blob([wbout], { type: 'application/octet-stream' });
	const url = URL.createObjectURL(blob);
	const a = document.createElement('a');
	a.href = url;
	a.download = 'sick_file.xlsx';
	document.getElementById('downloadFile')!.appendChild(a);
	a.click();
	// remove the element
	URL.revokeObjectURL(url);
	errors.value.push({
		message: 'Data saved to sick_file.xlsx',
		type: 'success'
	});
	endOfErrors();
	a.remove();
}

function reset() {
	files.value = [];
	errors.value = [];
	activityData.length = 0;
	careLogs = [];
	ptoData = [];
	hireData = [];
	fullData = [];
	loading.value = false;
	hasErrors.value = false;
	gotData.value = false;
}

async function endOfErrors() {
	await nextTick(); // Wait for DOM to update
	const element = document.getElementById('errors');
	element?.scrollTo(0, element.scrollHeight);
}
</script>

<template>
	<main>
		<section>
			<h4>Select Files</h4>
			<FileDragDrop v-model="files" />
			<div style="display: flex; gap: 8px; margin-top: 16px;">
				<button v-if="!gotData" @click="getData()" :disabled="files.length < 4 && !hasErrors">
					<svg height="24px" width="24px" viewBox="0 0 32 32">
						<g>
							<path style="fill:#fff;" d="M19.945,22l6.008-8L32,22h-4v2c0,3.309-2.691,6-6,6H10c-3.309,0-6-2.691-6-6v-2h4v2 c0,1.102,0.898,2,2,2h12c1.102,0,2-0.898,2-2v-2H19.945z"/>
							<path style="fill:#fff;" d="M12.055,10l-6.008,8L0,10h4V8c0-3.309,2.691-6,6-6h12c3.309,0,6,2.691,6,6v2h-4V8 c0-1.102-0.898-2-2-2H10C8.898,6,8,6.898,8,8v2H12.055z"/>
						</g>
					</svg>
					Load & Process Data
				</button>
				<button v-if="gotData" @click="saveData()">
					<svg width="28px" height="28px" viewBox="0 0 24 24" fill="none">
						<path d="M17 17H17.01M17.4 14H18C18.9319 14 19.3978 14 19.7654 14.1522C20.2554 14.3552 20.6448 14.7446 20.8478 15.2346C21 15.6022 21 16.0681 21 17C21 17.9319 21 18.3978 20.8478 18.7654C20.6448 19.2554 20.2554 19.6448 19.7654 19.8478C19.3978 20 18.9319 20 18 20H6C5.06812 20 4.60218 20 4.23463 19.8478C3.74458 19.6448 3.35523 19.2554 3.15224 18.7654C3 18.3978 3 17.9319 3 17C3 16.0681 3 15.6022 3.15224 15.2346C3.35523 14.7446 3.74458 14.3552 4.23463 14.1522C4.60218 14 5.06812 14 6 14H6.6M12 15V4M12 15L9 12M12 15L15 12" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
					</svg>
					Save Data to File
				</button>
				<button class="secondary" @click="reset()">
					reset
				</button>
			</div>
		</section>
		<aside>
			<h5>Errors / Logs</h5>
			<div id="errors" class="errors-wrapper">
				<p v-if="errors.length === 0">No errors</p>
				<ol style="padding: 0;">
					<li style="list-style: none; white-space: break-spaces;" v-for="(error, index) in errors" :key="index" class="log-message" :class="error.type">{{ error.message }}</li>
				</ol>
			</div>
		</aside>
	</main>
	<!-- download element -->
	<div id="downloadFile" style="display: none;"></div> 
</template>

<style>
main {
	display: flex;
	flex-direction: row;
	gap: 16px;
}
section {
	flex: 2;
}
aside {
	flex: 1;
}
.errors-wrapper {
	border: 1px solid #ccc;
	padding: 8px;
	display: block;
	border-radius: 4px;
	max-height: 600px;
	overflow-y: auto;.log-message.error {
	border: 1px solid red;
}
}
.log-message.error {
  border: 1px solid red;
}
.log-message.warn {
  border: 1px solid orange;
}
.log-message.success {
  border: 1px solid green;
}
</style>
