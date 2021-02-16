<script lang="ts">
	import PlayArea from "./PlayArea.svelte"
	let width:number, height:number
</script>

<main>
	<h1>Ball game</h1>
	<div id="playarea" bind:clientWidth={width} bind:clientHeight={height}>
		<PlayArea {width} {height}/>
	</div>
</main>

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 4em;
		font-weight: 100;
	}

	#playarea {
		width: 100%;
		height: 100%;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>