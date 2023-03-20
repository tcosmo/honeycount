<script lang="ts">
	import dayjs from 'dayjs';
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

	const salary = 2395;
	const fixedExpenses = [750, 40];
	let savingTarget = 500;
	let runningExpenses: [Date, number][] = [];
	$: remainingMoney =
		salary -
		savingTarget -
		arraySum(fixedExpenses) -
		arraySum(
			runningExpenses.map((couple) => {
				return couple[1];
			})
		);

	let inputExpenseDate = new Date();
	let inputExpenseAmount: null | number = null;
	let tableEditDisabled = true;
</script>

<main class="p-4">
	<div class="text-xs mb-2">Mar 25th 2023 - Apr 25th 2023</div>
	<div class="flex space-x-4">
		<div class="border-black p-3 border-2 min-w-[150px] h-100">
			<h1>Remaining</h1>
			<div class="text-3xl text-center">{formatMoney(remainingMoney)}</div>
			<div class="text-sm mt-1">
				{formatMoney(remainingMoney / 30)} <span class="text-xs italic">per diem</span>
			</div>
		</div>
		<div class="border-black p-3 border-2 w-100 h-100">
			<h1>Saving target</h1>
			<div class="text-2xl text-center">{formatMoney(savingTarget)}</div>
			<div class="mt-2 flex w-full justify-between px-2">
				<button
					class="border-2 border-black px-2 hover:bg-gray-200"
					on:click={() => {
						savingTarget -= 10;
					}}>-</button
				>
				<button
					class="border-2 border-black px-2 hover:bg-gray-200"
					on:click={() => {
						savingTarget += 10;
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
				runningExpenses.push([inputExpenseDate, inputExpenseAmount ?? 0]);
				runningExpenses = runningExpenses;
				inputExpenseAmount = null;
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
					{#each runningExpenses.sort((a, b) => {
						if (a[0] > b[0]) {
							return -1;
						} else {
							return 1;
						}
					}) as [date, amount], i}
						<tr>
							<td class="">{dayjs(date).format('DD/MM')}</td>
							<td class="w-4/12">{formatMoney(amount)}</td>
							<td class="w-2/12 py-2"
								><button
									type="button"
									class="disabled:opacity-50"
									disabled={tableEditDisabled}
									on:click={() => {
										runningExpenses.splice(i, 1);
										runningExpenses = runningExpenses;
										tableEditDisabled = true;
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
