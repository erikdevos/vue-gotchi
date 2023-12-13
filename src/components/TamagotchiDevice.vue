<template>
	<div class="container">
		<div class="gotchi-frame">
			<div class="frame-screen">
				<!-- Happyness / Energy bars -->
				<div class="status-bars-wrapper">
					<div class="status-bar happyness-bar" >
						<div class="bar-value happyness-bar-value" :style="happynessBarStyles()"></div>
						<span class="bar-label">happy</span>
					</div>
					<div class="status-bar energy-bar">
						<div class="bar-value energy-bar-value" :style="energyBarStyles()"></div>
						<span class="bar-label">tired</span>
					</div>
				</div>

				<!-- The animation/sprite -->
				<div class="frame-wrapper">
					<img alt="" :src="spriteImagePath">
				</div>
			</div>
			<!-- Button -->
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
// Get values fro parent App
  props: {
	startGotchi: Boolean,
	spritePath: String,
	spriteDirection: String,
	gotchiStatus: String,
	gotchiEnergy: Number,
	gotchiHappyness: Number,
},

  data() {
	return {
		localHappyness: this.gotchiHappyness,
		happynessInterval: null,
		currentFrame: null,
		gotchiIdle: true
	};
  },

  computed: {
    spriteImagePath() {
      // Always return the computed path based on the currentFrame
      return require(`@/assets/sprites/${this.currentFrame}`);
    },
  },

	watch: {
	gotchiStatus(value) {
	if (value === 'dead') {
		console.log("if gotchi is dead case");
		// If gotchiStatus is 'dead', set the frame to 'sprite_shocked.png'
		this.gotchiIdle = false;
		console.log(this.gotchiIdle);
		return this.currentFrame = "sprite_shocked.png";
	} else {
		console.log("else case");
	}
	},


// Inside the gotchiHappyness method
gotchiHappyness() {
    if (this.gotchiHappyness <= 0) {
        this.$emit('gotchi-status-update', 'dead'); // Emit event to update gotchiStatus in parent
        console.log('Gotchi died! :(');
    } else {
        console.log('Gotchi alive :)');
    }
},

// Inside the gotchiEnergy watcher
gotchiEnergy() {
    if (this.gotchiEnergy <= 0) {
        this.$emit('gotchi-status-update', 'tired'); // Emit event to update gotchiStatus in parent
        console.log("Gotchi too tired");
    } else {
        console.log("Gotchi rested");
    }
},
	
},

  methods: {
	getBarStyles(value) {
		if (value <= 0) {
		return { width: "0%" };
		} else if (value <= 30) {
		return { "background-color": "red", width: value + "%" };
		} else if (value <= 50) {
		return { "background-color": "orange", width: value + "%" };
		} else {
		return { width: value + "%" };
		}
	},

	happynessBarStyles() {
		return this.getBarStyles(this.localHappyness);
	},

	energyBarStyles() {
		return this.getBarStyles(this.gotchiEnergy);
	},

	decreaseHappyness() {
		this.happynessInterval = setInterval(() => {
		if (this.localHappyness > 0) {
			this.localHappyness -= 1;
		}

		if (this.localHappyness <= 0) {
			this.$emit('gotchi-status-update', 'dead');
			console.log("Gotchi died! :(");
		} else {
			console.log("Gotchi alive :)");
		}
		}, 100); // Decrease happyness every 1000 milliseconds (1 second)
	},

	idle() {
  console.log('idle');
  this.clearIdleTimeouts();

  if (this.gotchiIdle === true) {
    const generateFrameName = (name) => `sprite_${name}.png`;
    const animationFrameNames = ['calm', 'casual', 'calm', 'casual', 'neutral'];

    const numFrames = animationFrameNames.length;
    const animationFrames = animationFrameNames.map(generateFrameName);

    let index = 0;

    const animate = () => {
      if (this.gotchiIdle === true) {
        this.currentFrame = animationFrames[index];

        // Increment the index or reset it to 0 if it reaches the end of the array
        index = (index + 1) % numFrames;

        // Set a timeout for the next frame change (adjust the delay as needed)
        this.idleTimeout = setTimeout(animate, 1000); // 1000 milliseconds = 1 second
      }
    };

    // Start the animation
    console.log("animate idle");
    animate();
  } else {
    console.log("not idle");
  }
},

// Method to clear any existing timeouts
clearIdleTimeouts() {
  if (this.idleTimeout) {
    clearTimeout(this.idleTimeout);
    this.idleTimeout = null;
  }
},
  },

  created() {
	// Start decreasing happyness when the component is created
	this.idle();
	this.decreaseHappyness();
	},

	
  beforeUnmount() {
	// Clear the interval when the component is about to be destroyed
	clearInterval(this.happynessInterval);
	},
};
</script>

<!-- Style -->
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
		position: relative;
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

	.status-bars-wrapper {
		position: absolute;
		width: calc(100% - 1rem);
		margin-top: 0.5rem;
		margin-left: 0.5rem;
		margin-right: 0.5rem;
		gap: 0.5rem;
		display: flex;
	}

	.status-bar {
		display: flex;
		align-items: center;
		justify-content: center;
		height: 1rem;
		border: solid 3px #000000;
		border-radius: 0.3rem;
		font-weight: bold;
		font-size: 0.8rem;
		flex: 1;
		text-align: center;
		position: relative;

		.bar-value {
			height: 100%;
			position: absolute;
			left: 0;
			top: 0;
			width: 100%;
			z-index: 2;
		}
		&.happyness-bar {
			.happyness-bar-value {
				background-color: #53bf8e;
			}
		}

		&.energy-bar {
			.energy-bar-value {
				background-color: #6673dd;
			}
		}

		.bar-label {
			z-index: 3;
		}
	}

	.frame-wrapper {
		width: 80%;
		align-self: end;
		margin-bottom: 1rem;
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
