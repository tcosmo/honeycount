<script lang="ts">
	import dayjs from 'dayjs';
	import { onMount } from 'svelte';
	import '../app.css';
	import DateInput from '../lib/DateInput.svelte';

	const TABLE_DATE_FORMAT = 'DD/MM';

	const moneyFormatter = new Intl.NumberFormat('ie-IE', {
		style: 'currency',
		currency: 'EUR',
		minimumFractionDigits: 0,
		maximumFractionDigits: 0
	});

	function formatMoney(money: number): string {
		return moneyFormatter.format(money);
	}

	function arraySum(array: number[]): number {
		return array.reduce((partialSum, a) => partialSum + a, 0);
	}

	interface State {
		currentPeriodStartDate: Date;
		salary: Record<string, number>;
		fixedExpensesTotal: Record<string, number>;
		savingTarget: Record<string, number>;
		runningExpenses: [Date, number][];
	}

	const DEFAULT_SALARY = 0;
	const DEFAULT_FIXED_EXPENSES = 0;
	const DEFAULT_SAVING_TARGET = 0;

	function defaultState(): State {
		const currentPeriodStartDate = dayjs('2023-03-25', 'YYYY-MM-DD').toDate();
		return {
			currentPeriodStartDate: currentPeriodStartDate,
			salary: { [currentPeriodStartDate.toISOString()]: DEFAULT_SALARY },
			fixedExpensesTotal: { [currentPeriodStartDate.toISOString()]: DEFAULT_FIXED_EXPENSES },
			savingTarget: { [currentPeriodStartDate.toISOString()]: DEFAULT_SAVING_TARGET },
			runningExpenses: []
		};
	}

	function stateToJSON(state: State): string {
		return JSON.stringify(state);
	}

	function stateFromJSON(stateJSON: string | null): State | null {
		if (stateJSON === null) return null;
		const toReturn = JSON.parse(stateJSON);
		toReturn.currentPeriodStartDate = new Date(Date.parse(toReturn.currentPeriodStartDate));

		for (let expense of toReturn.runningExpenses) {
			expense[0] = new Date(Date.parse(expense[0]));
		}
		return toReturn;
	}

	const DEFAULT_STORE_ID = 'honeycount';
	function stateToLocalStorage(state: State) {
		state.runningExpenses = state.runningExpenses.sort((a, b) => {
			if (a[0] > b[0]) {
				return -1;
			} else {
				return 1;
			}
		});
		window.localStorage.setItem(DEFAULT_STORE_ID, stateToJSON(state));
	}

	function stateFromLocalStorage(): State {
		return stateFromJSON(window.localStorage.getItem(DEFAULT_STORE_ID)) ?? defaultState();
	}

	let state: State = defaultState();

	$: sortedRunningExpenses = state.runningExpenses.sort((a, b) => {
		if (a[0] > b[0]) {
			return -1;
		} else {
			return 1;
		}
	});

	onMount(async () => {
		if (navigator.storage && navigator.storage.persist) {
			const isPersisted = await navigator.storage.persist();
			console.log(`Persisted storage granted: ${isPersisted}`);
		}

		state = stateFromLocalStorage();
	});

	$: currentPeriodEndDate = dayjs(state.currentPeriodStartDate).add(1, 'month').toDate();

	$: remainingMoney =
		state.salary[state.currentPeriodStartDate.toISOString()] -
		state.savingTarget[state.currentPeriodStartDate.toISOString()] -
		state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()] -
		arraySum(
			state.runningExpenses
				.filter(([date, _amount]) => {
					return date >= state.currentPeriodStartDate && date < currentPeriodEndDate;
				})
				.map(([_date, amount]) => {
					return amount;
				})
		);

	$: moneySpentToday = arraySum(
		state.runningExpenses
			.filter(([date, _amount]) => {
				let now = new Date();
				return (
					date.getDay() === now.getDay() &&
					date.getFullYear() === now.getFullYear() &&
					date.getMonth() === now.getMonth()
				);
			})
			.map(([_date, amount]) => {
				return amount;
			})
	);

	$: remainingMoneyExcludingToday = remainingMoney + moneySpentToday;

	let perDiem = 0;
	let perDiemTomorrow: null | number = null;
	$: {
		const now = new Date();
		if (now >= state.currentPeriodStartDate && now < currentPeriodEndDate) {
			perDiem = remainingMoneyExcludingToday / dayjs(currentPeriodEndDate).diff(dayjs(now), 'days');
			let remainDaysTomorrow = dayjs(currentPeriodEndDate).diff(dayjs(now), 'days') - 1;
			if (remainDaysTomorrow > 0) {
				perDiemTomorrow = remainingMoney / remainDaysTomorrow;
			}
		} else if (now < state.currentPeriodStartDate) {
			perDiem = perDiem =
				remainingMoney /
				dayjs(currentPeriodEndDate).diff(dayjs(state.currentPeriodStartDate), 'days');
		} else {
			perDiem = remainingMoney;
		}
	}

	function changePeriod(monthDelta: number) {
		const oldPeriodStartDate = state.currentPeriodStartDate;
		if (monthDelta < 0) {
			state.currentPeriodStartDate = dayjs(state.currentPeriodStartDate)
				.subtract(Math.abs(monthDelta), 'month')
				.toDate();
		} else {
			state.currentPeriodStartDate = dayjs(state.currentPeriodStartDate)
				.add(monthDelta, 'month')
				.toDate();
		}

		if (state.savingTarget[state.currentPeriodStartDate.toISOString()] === undefined) {
			state.savingTarget[state.currentPeriodStartDate.toISOString()] = DEFAULT_SAVING_TARGET;
		}

		if (state.salary[state.currentPeriodStartDate.toISOString()] === undefined) {
			state.salary[state.currentPeriodStartDate.toISOString()] =
				state.salary[oldPeriodStartDate.toISOString()];
		}

		if (state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()] === undefined) {
			state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()] =
				state.fixedExpensesTotal[oldPeriodStartDate.toISOString()];
		}
	}

	let inputExpenseDate = new Date();
	let inputExpenseAmount: null | number = null;
	let tableEditDisabled = true;
	let showSettings = false;
</script>

<main class="p-4">
	<div class="flex items-center mb-2 space-x-3">
		<div class="text-xs mb-2">
			{dayjs(state.currentPeriodStartDate).format('MMM DD, YYYY')} - {dayjs(
				currentPeriodEndDate
			).format('MMM DD, YYYY')}
		</div>
		<div class="flex space-x-3">
			<button
				class="border border-black px-2 hover:bg-gray-200"
				on:click={() => {
					changePeriod(-1);
					stateToLocalStorage(state);
				}}>←</button
			>
			<button
				class="border border-black px-2 hover:bg-gray-200"
				on:click={() => {
					changePeriod(1);
					stateToLocalStorage(state);
				}}>→</button
			>
			<button
				class="border border-black px-2 hover:bg-gray-200"
				on:click={() => {
					showSettings = !showSettings;
				}}>⚙️</button
			>
		</div>
	</div>

	{#if showSettings}
		<div class="flex flex-col items-start space-y-2 text-sm mb-2">
			<div>
				Period start date: <DateInput
					bind:date={state.currentPeriodStartDate}
					on:dateChanged={(_) => {
						if (state.savingTarget[state.currentPeriodStartDate.toISOString()] === undefined) {
							console.log('aa');
							state.savingTarget[state.currentPeriodStartDate.toISOString()] =
								DEFAULT_SAVING_TARGET;
						}

						if (state.salary[state.currentPeriodStartDate.toISOString()] === undefined) {
							state.salary[state.currentPeriodStartDate.toISOString()] = DEFAULT_SALARY;
						}

						if (
							state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()] === undefined
						) {
							state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()] =
								DEFAULT_FIXED_EXPENSES;
						}

						stateToLocalStorage(state);
					}}
				/>
			</div>
			<div>
				Period salary: <input
					type="number"
					class="border border-black p-1 w-4/12"
					bind:value={state.salary[state.currentPeriodStartDate.toISOString()]}
				/>
			</div>
			<div>
				Period fixed expenses: <input
					class="border border-black  p-1 w-4/12"
					type="number"
					bind:value={state.fixedExpensesTotal[state.currentPeriodStartDate.toISOString()]}
				/>
			</div>
		</div>
	{/if}

	<div class="flex space-x-4">
		<div class="border-black p-3 border-2 min-w-[150px] h-100">
			<h1>Remaining</h1>
			<div class="text-3xl text-center">{formatMoney(remainingMoney)}</div>
			<div class="text-sm mt-1">
				{formatMoney(perDiem)} <span class="text-xs italic">per diem</span>
			</div>
			<div
				class="text-sm mt-1"
				class:text-red-500={moneySpentToday > perDiem}
				class:font-bold={moneySpentToday > perDiem}
			>
				{formatMoney(moneySpentToday)} <span class="text-xs italic">spent today</span>
			</div>
			{#if perDiemTomorrow}
				<div class="text-sm mt-1">
					{formatMoney(perDiemTomorrow)} <span class="text-xs italic">per diem +1 day</span>
				</div>
			{/if}
		</div>
		<div class="border-black p-3 border-2 w-100 h-100">
			<h1>Saving target</h1>
			<div class="text-2xl text-center">
				{formatMoney(state.savingTarget[state.currentPeriodStartDate.toISOString()])}
			</div>
			<div class="mt-2 flex w-full justify-between px-2">
				<button
					class="border-2 border-black px-2 hover:bg-gray-200"
					on:click={() => {
						state.savingTarget[state.currentPeriodStartDate.toISOString()] -= 10;
						stateToLocalStorage(state);
					}}>-</button
				>
				<button
					class="border-2 border-black px-2 hover:bg-gray-200"
					on:click={() => {
						state.savingTarget[state.currentPeriodStartDate.toISOString()] += 10;
						stateToLocalStorage(state);
					}}>+</button
				>
			</div>
		</div>
	</div>

	<div class="mt-2">
		<div class="flex space-x-3 items-center">
			<h1 class="text-xl font-bold">Expenses</h1>
			<button
				on:click={() => {
					tableEditDisabled = !tableEditDisabled;
				}}>✍️</button
			>
			<!-- <button class="p-2 bg-blue-400 rounded-xl text-gray-100 font-bold hover:bg-blue-300"
				>New expense</button
			> -->
		</div>
		<form
			on:submit|preventDefault={() => {
				let now = new Date();

				if (
					dayjs(inputExpenseDate).format(TABLE_DATE_FORMAT) == dayjs(now).format(TABLE_DATE_FORMAT)
				) {
					inputExpenseDate = now;
				}
				state.runningExpenses.push([inputExpenseDate, inputExpenseAmount ?? 0]);
				state.runningExpenses = state.runningExpenses;
				inputExpenseAmount = null;
				stateToLocalStorage(state);
			}}
		>
			<table class="w-[300px]">
				<thead class="text-left ">
					<th class="">Date</th>
					<th class="w-4/12">Amount</th>
					<th class="w-2/12" />
				</thead>
				<tbody>
					<tr>
						<td><DateInput bind:date={inputExpenseDate} dateMin="2023-02-25" maxToday /></td>
						<td class=""
							><input
								class="border-black border w-10/12 px-2 py-1"
								type="number"
								bind:value={inputExpenseAmount}
								required
							/></td
						>
						<td class="w-2/12 py-4"><button type="submit">✅</button></td>
					</tr>
					{#each sortedRunningExpenses as [date, amount], i}
						<tr>
							<td class="">{dayjs(date).format('DD/MM')}</td>
							<td class="w-4/12">{formatMoney(amount)}</td>
							<td class="w-2/12 py-2"
								><button
									type="button"
									class="disabled:opacity-40"
									disabled={tableEditDisabled}
									on:click={() => {
										state.runningExpenses.splice(i, 1);
										state.runningExpenses = state.runningExpenses;
										tableEditDisabled = true;
										stateToLocalStorage(state);
									}}>❌</button
								></td
							>
						</tr>
					{/each}
				</tbody>
			</table>
		</form>
	</div>
</main>
