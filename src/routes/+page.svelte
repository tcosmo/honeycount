<script lang="ts">
	import { each } from 'svelte/internal';
	import '../app.css';

	const moneyFormatter = new Intl.NumberFormat('ie-IE', { style: 'currency', currency: 'EUR' });

	function formatMoney(money: number): string {
		return moneyFormatter.format(money);
	}

	function arraySum(array: number[]): number {
		return array.reduce((partialSum, a) => partialSum + a, 0);
	}

	const salary = 2395;
	const fixedExpenses = [750, 40];
	let savingTarget = 500;
	let runningExpenses: [string, number][] = [
		['19/03', 55],
		['18/03', 25],
		['17/03', 2]
	];
	$: remainingMoney =
		salary -
		savingTarget -
		arraySum(fixedExpenses) -
		arraySum(
			runningExpenses.map((couple) => {
				return couple[1];
			})
		);
</script>

<main class="p-4">
	<div class="text-xs mb-2">Current period: Feb 25th 2023 - Mar 25th 2023</div>
	<div class="flex space-x-4">
		<div class="border-black p-3 border-2 w-100 h-100">
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
		<h1 class="text-xl font-bold">Expenses</h1>
		<table class="w-[400px]">
			<thead class="text-left">
				<th class="w-2/12">Date</th>
				<th>Amount</th>
			</thead>
			<tbody>
				{#each runningExpenses as [date, amount]}
					<tr>
						<td class="w-2/12">{date}</td>
						<td>{formatMoney(amount)}</td>
					</tr>
				{/each}
			</tbody>
		</table>
	</div>
</main>
