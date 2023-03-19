<script lang="ts">
	import dayjs from 'dayjs';

	export let format = 'YYYY-MM-DD';
	export let date = new Date();
	export let dateMin: null | string = null;
	export let dateMax: null | string = null;
	export let maxToday: boolean = false;

	if (maxToday) {
		dateMax = dayjs(date).format(format);
	}

	let internal: any;

	const input = (x: any) => (internal = dayjs(x).format(format));
	const output = (x: any) => {
		date = dayjs(x, format).toDate();
	};

	$: input(date);
	$: output(internal);
</script>

<input type="date" bind:value={internal} class="border-black border" min={dateMin} max={dateMax} />
