<template>
	<div class="container">
		<div class="gotchi-frame">
			<div class="frame-screen">
				<div class="frame-wrapper">
					<img alt="" :src="spriteImagePath">
				</div>
			</div>
		</div>
	</div>
</template>

<script>
export default {
  props: {
	startGotchi: Boolean,
	currentFrame: String,
	spritePath: String,
	spriteDirection: String,
  },
  computed: {
  spriteImagePath() {
    return require(`@/assets/sprites/${this.currentFrame}`);
  },

},
};
</script>


<style scoped>

.container {
	max-width: 600px;
	margin-left: auto;
	margin-right: auto;
}

.gotchi-frame {
	display: flex;
	justify-content: center;
	align-items: flex-end;
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
		border: solid 8px black;
		border-radius: 1rem;
		height: 18rem;
		width: 15rem;
		display: flex;
		justify-content: center;
		margin-bottom: 33%;
	}

	.frame-wrapper {
		width: 80%;
		align-self: center;
		> img {
			width: 100%;
			height: auto;
			image-rendering: crisp-edges;
			object-fit: contain;

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
