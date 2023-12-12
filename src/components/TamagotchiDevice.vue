<template>
	<div class="container">
		<div class="gotchi-frame">
			<div class="frame-screen">
				<div class="frame-wrapper">
					<img alt="" :src="spriteImagePath">
				</div>
			</div>
			<div class="button-bar">
				<button>Eat</button>
				<button>Play</button>
				<button>Sleep</button>
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
	margin-left: auto;
	margin-right: auto;
	height: 100%;
	width: 100%;
	max-width: 600px;
	display: flex;
	align-items: center;
}

.gotchi-frame {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	width: 100%;
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
		margin-top: 5rem;
		margin-bottom: 3rem;
	}

	.frame-wrapper {
		width: 80%;
		align-self: center;
		> img {
			width: 100%;
			height: auto;
			image-rendering: crisp-edges;
			filter: grayscale(1);
			object-fit: contain;

		}
	}
}

.button-bar {
	display: flex;
	gap: 1rem;
	button {
		all: unset;
		display: inline-flex;
		align-items: center;
		justify-content: center;
		padding: 1.2rem;
		border-radius: 100%;
		background-color: #FFFFFF;
		aspect-ratio: 1/1;
		text-align: center;
		font-family: "Comic Sans MS", "Comic Sans", cursive;
		font-size: 1rem;
		font-weight: 900;
		border: solid 5px #000000;
		cursor: pointer;
		user-select: none;
		box-shadow: rgba(0, 0, 0, 0.5) 1px 1px 5px;
		&:hover {
			background-color: salmon;
			color: #000000;
			transform: translateY(0.1rem);
			box-shadow: rgba(0, 0, 0, 0.5) 1px 1px 3px;
		}
		&:active {
			transform: translateY(0.3rem);
			box-shadow: rgba(0, 0, 0, 0.3) 1px 1px 0px;

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
