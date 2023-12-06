<template>
	<div class="container">
		<h1>{{ pageTitle }}</h1>
		<div class="gotchi-frame">

			<div class="frame-screen">
				<img src="../assets/sprites/sprite_neutral.png" alt="">
			</div>
		</div>
	</div>
</template>

<script>
export default {
	props: {
		pageTitle: String
	}
}
</script>


<style scoped>

.container {
	max-width: 800px;
	margin-left: auto;
	margin-right: auto;
}

.gotchi-frame {
	display: flex;
	justify-content: center;
	align-items: center;
	height: 80vh;
	background-image: url("../assets/egg-shape.svg");
	background-position: center center;
	background-repeat: no-repeat;
	background-size: contain;

	svg {
		width: 100%;
		max-height: 80vh;
	}

	.frame-screen {
		background-color: salmon;
		border: solid 2px black;
		height: 16rem;
		width: 16rem;
		display: flex;
		justify-content: center;
		
		> img {
			image-rendering: crisp-edges;
		}
	}
}

h3 {
	margin: 40px 0 0;
}

ul {
	list-style-type: none;
	padding: 0;
	li {
		display: inline-block;
		margin: 0 10px;
	}
}

a {
	color: #42b983;
}

</style>
