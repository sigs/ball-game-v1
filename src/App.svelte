<script lang="ts">
	import PlayArea from "./PlayArea.svelte"
	let width:number, height:number
</script>

<main>
	<h1>BALL GAME &nbsp; v.1</h1>
	<div id="loading">
		<div style="height:100px"></div>
		<img style="width:30%" src="res/loading.png" alt="loading...">
		<h2 style="text-align:center">LOADING...</h2>
	</div>
	<div id="playarea" bind:clientWidth={width} bind:clientHeight={height}>
		<PlayArea {width} {height}/>
	</div>
</main>

<style>
	main {
		text-align: center;
		padding: 0;
		margin: 0 auto;
		height: 100%;
	}

	h1 {
		color: #ff3e00;
		font-size: 4em;
		font-weight: 100;
	}

	#playarea {
		width: 100%;
		height: 70%;
	}

	#loading {
		width: 100%;
		position: absolute;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>