<template>
	<div class="container">
		<div class="gotchi-frame">
			<div class="frame-screen">
				<!-- Status bars -->
				<div class="status-bars-wrapper">
					<div class="status-bar happiness-bar">
						<div class="bar-value happiness-bar-value" :style="barStyle(happiness)"></div>
						<span class="bar-label">happy</span>
					</div>
					<div class="status-bar energy-bar">
						<div class="bar-value energy-bar-value" :style="barStyle(energy)"></div>
						<span class="bar-label">energy</span>
					</div>
				</div>

				<!-- Sprite or Game -->
				<div class="frame-wrapper" :class="{ 'is-acting': isActing }">
					<!-- Normal sprite -->
					<img v-if="!isPlayingGame" alt="" :src="currentSprite">
					
					<!-- Games -->
					<div v-if="isPlayingGame" class="game-board">
						<!-- Gotchi sprite during game -->
						<div class="game-gotchi-sprite">
							<img alt="" :src="currentSprite">
						</div>
						
						<!-- Rock Paper Scissors -->
						<div v-if="currentGame === 'rps'" class="rps-game">
							<div class="game-title">Rock Paper Scissors VS GOTCHI</div>
							
							<!-- Choice buttons -->
							<div v-if="gameState === 'playing'" class="game-choices">
								<button @click="makeChoice('rock')" class="choice-btn">
									<div class="choice-icon rock-icon"></div>
									<span>ROCK</span>
								</button>
								<button @click="makeChoice('paper')" class="choice-btn">
									<div class="choice-icon paper-icon"></div>
									<span>PAPER</span>
								</button>
								<button @click="makeChoice('scissors')" class="choice-btn">
									<div class="choice-icon scissors-icon"></div>
									<span>SCISSORS</span>
								</button>
							</div>
							
							<!-- Result display -->
							<div v-if="gameState === 'reveal'" class="game-result">
								<div class="choices-display">
									<div class="choice-display">
										<div class="choice-icon" :class="playerChoice + '-icon'"></div>
										<span>YOU</span>
									</div>
									<div class="vs-text">VS</div>
									<div class="choice-display">
										<div class="choice-icon" :class="gotchiChoice + '-icon'"></div>
										<span>GOTCHI</span>
									</div>
								</div>
								<div class="result-text" :class="gameResult">
									{{ gameResult.toUpperCase() }}!
								</div>
							</div>
						</div>
						
						<!-- Tic-Tac-Toe -->
						<div v-if="currentGame === 'tictactoe'" class="ttt-game">
							<div class="game-title">Tic-Tac-Toe VS GOTCHI</div>
							
							<!-- Game board -->
							<div v-if="gameState === 'playing'" class="ttt-board">
								<button
									v-for="(cell, index) in tttBoard"
									:key="index"
									@click="makeTTTMove(index)"
									:disabled="cell !== null"
									class="ttt-cell"
								>
									<span v-if="cell === 'X'" class="ttt-x">✕</span>
									<span v-if="cell === 'O'" class="ttt-o">○</span>
								</button>
							</div>
							
							<!-- Result display -->
							<div v-if="gameState === 'reveal'" class="game-result">
								<div class="result-text">
									{{ gameResult }}
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>

			<!-- Buttons -->
			<div class="button-bar">
				<button @click="doEat" :disabled="!canAct" title="Feed your gotchi (+energy)">
					Eat
				</button>
				<button @click="doPlay" :disabled="!canAct" title="Play with your gotchi (+happy, -energy)">
					Play
				</button>
				<button @click="doSleep" :disabled="isDead || isActing" title="Put your gotchi to sleep">
					{{ isSleeping ? 'Wake' : 'Sleep' }}
				</button>
				<button @click="doPet" :disabled="!canAct" title="Pet your gotchi (+happiness)">
					Pet
				</button>
				<button @click="doGame" :disabled="!canAct" title="Play Rock Paper Scissors">
					Game
				</button>
				<button v-if="isDead" @click="restart" class="restart-btn" title="Start a new gotchi">
					Restart
				</button>
			</div>
		</div>
	</div>
</template>

<script>
// ─── Animation Config ────────────────────────────────────────────────────────
// Edit frame sequences and timing here without touching game logic.
// Frame names map to /assets/sprites/sprite_<name>.png
const ANIMATIONS = {
	idle:     { frames: ['calm', 'casual', 'neutral', 'calm'],                        interval: 900,  loop: true  },
	idle_happy: { frames: ['happy', 'joy', 'happy', 'calm'],                         interval: 900,  loop: true  },
	idle_sad:   { frames: ['sad', 'neutral', 'sad', 'calm'],                         interval: 900,  loop: true  },
	idle_mad:   { frames: ['mad', 'sad', 'mad', 'neutral'],                          interval: 900,  loop: true  },
	eating:   { frames: ['active', 'happy', 'joy', 'happy', 'neutral'],               interval: 380,  loop: false },
	playing:  { frames: ['fun', 'kiss_01', 'kiss_02', 'happy', 'joy', 'neutral'],     interval: 380,  loop: false },
	petting:  { frames: ['kiss_01', 'kiss_02', 'happy', 'joy', 'neutral'],           interval: 400,  loop: false },
	sleeping: { frames: ['sleep_01', 'sleep_02'],                                      interval: 2000, loop: true  },
	waking:   { frames: ['neutral', 'casual', 'calm'],                                interval: 600,  loop: false },
	sick:     { frames: ['mad', 'sad', 'mad'],                                         interval: 800,  loop: true  },
	overfeeding: { frames: ['mad', 'sad', 'mad', 'sad'],                               interval: 400,  loop: false },
	attention: { frames: ['mad', 'sad', 'mad', 'neutral'],                            interval: 600,  loop: true  },
	dead:     { frames: ['shocked', 'sad'],                                           interval: 1400, loop: true  },
};

// ─── Game Constants ───────────────────────────────────────────────────────────
const DECAY_INTERVAL  = 3000; // ms per game tick
const HAPPINESS_DECAY = 1.5;  // base happiness lost per tick
const ENERGY_DECAY    = 0.75; // energy lost per tick
const SLEEP_REGEN     = 8;    // energy gained per tick while sleeping
const ENERGY_CRISIS_THRESHOLD = 25; // below this, happiness decays faster
const SICKNESS_THRESHOLD = 15; // below this energy, gotchi gets sick
const ATTENTION_THRESHOLD = 35; // below this happiness, gotchi seeks attention
const MOOD_INERTIA = 0.1; // how quickly mood changes (0-1)
const OVERFEED_BASE_PENALTY = 5; // base happiness lost when overfed
const OVERFEED_SICKNESS_THRESHOLD = 4; // overfeed count to trigger sickness
const OVERPET_THRESHOLD = 8; // pet count to trigger overpetting penalty
const OVERPET_HAPPINESS_PENALTY = 3; // happiness lost when overpetted

export default {
	name: 'TamagotchiDevice',

	data() {
		return {
			happiness:   80,
			energy:      80,
			mood:        80, // 0-100, changes slowly based on happiness
			status:      'idle', // idle | eating | playing | sleeping | waking | sick | attention | dead | game
			currentFrame: 'calm',
			isActing:    false,  // true only during one-shot action animations
			animTimeout: null,
			decayInterval: null,
			attentionInterval: null, // for attention-seeking behavior
			petCount: 0, // track pet usage for diminishing returns
			// Game state
			currentGame: null, // 'rps' | 'tictactoe' | null
			gameState: null, // 'playing' | 'reveal' | null
			// RPS state
			playerChoice: null, // 'rock' | 'paper' | 'scissors'
			gotchiChoice: null,
			gameResult: null, // 'win' | 'lose' | 'tie',
			// Tic-Tac-Toe state
			tttBoard: [null, null, null, null, null, null, null, null, null], // 3x3 grid
			tttPlayerSymbol: 'X',
			tttGotchiSymbol: 'O',
			tttCurrentTurn: 'X', // 'X' or 'O'
			overfeedCount: 0, // track overfeeding for sickness/death
		};
	},

	computed: {
		currentSprite() {
			return require(`@/assets/sprites/sprite_${this.currentFrame}.png`);
		},
		isSleeping() {
			return this.status === 'sleeping';
		},
		isDead() {
			return this.status === 'dead';
		},
		canAct() {
			return !this.isDead && !this.isSleeping && !this.isActing && this.status !== 'attention' && this.status !== 'game';
		},

		isPlayingGame() {
			return this.status === 'game';
		},

		moodState() {
			if (this.mood >= 80) return 'happy';
			if (this.mood >= 50) return 'neutral';
			if (this.mood >= 25) return 'sad';
			return 'mad';
		},
	},

	methods: {
		// ── Visuals ──────────────────────────────────────────────────────────
		barStyle(value) {
			const v = Math.max(0, Math.min(100, value));
			let bg = '#53bf8e';
			if (v <= 30) bg = '#e74c3c';
			else if (v <= 55) bg = '#f39c12';
			return { width: v + '%', backgroundColor: bg };
		},

		// ── Animation engine ─────────────────────────────────────────────────
		playAnimation(animKey, onDone) {
			this.stopAnimation();
			const anim = ANIMATIONS[animKey];
			if (!anim) return;

			this.isActing = !anim.loop;
			let index = 0;

			const tick = () => {
				this.currentFrame = anim.frames[index];
				index++;

				if (index < anim.frames.length) {
					this.animTimeout = setTimeout(tick, anim.interval);
				} else if (anim.loop) {
					index = 0;
					this.animTimeout = setTimeout(tick, anim.interval);
				} else {
					this.isActing = false;
					if (onDone) onDone();
				}
			};

			tick();
		},

		stopAnimation() {
			if (this.animTimeout) {
				clearTimeout(this.animTimeout);
				this.animTimeout = null;
			}
			this.isActing = false;
		},

		startIdle() {
			// Choose idle animation based on mood
			let animKey = 'idle';
			if (this.mood >= 80) animKey = 'idle_happy';
			else if (this.mood >= 50) animKey = 'idle';
			else if (this.mood >= 25) animKey = 'idle_sad';
			else animKey = 'idle_mad';

			this.status = 'idle';
			this.playAnimation(animKey);
		},

		// ── Games ─────────────────────────────────────────────────────────────
		doGame() {
			if (!this.canAct) return;
			
			// Randomly select a game (extensible for future games)
			const availableGames = ['rps', 'tictactoe'];
			this.currentGame = availableGames[Math.floor(Math.random() * availableGames.length)];
			
			// Start game with neutral idle animation
			this.status = 'game';
			this.gameState = 'playing';
			
			// Reset game states
			this.playerChoice = null;
			this.gotchiChoice = null;
			this.gameResult = null;
			this.tttBoard = [null, null, null, null, null, null, null, null, null];
			this.tttCurrentTurn = 'X';
			
			this.stopAnimation();
			this.playAnimation('idle');
		},

		makeChoice(choice) {
			if (this.gameState !== 'playing') return;
			
			this.playerChoice = choice;
			
			// Gotchi makes random choice
			const choices = ['rock', 'paper', 'scissors'];
			this.gotchiChoice = choices[Math.floor(Math.random() * 3)];
			
			// Determine winner
			this.gameResult = this.determineWinner(this.playerChoice, this.gotchiChoice);
			
			// Show gotchi reaction based on result
			this.stopAnimation();
			if (this.gameResult === 'win') {
				// Player wins = gotchi loses (sad)
				this.playAnimation('idle_sad');
				this.happiness = Math.min(100, this.happiness + 15);
				this.energy = Math.max(0, this.energy - 5);
			} else if (this.gameResult === 'lose') {
				// Player loses = gotchi wins (happy)
				this.playAnimation('idle_happy');
				this.happiness = Math.max(0, this.happiness - 5);
			} else {
				// Tie = neutral
				this.playAnimation('idle');
			}
			
			// Show result briefly, then return to normal
			this.gameState = 'reveal';
			setTimeout(() => {
				this.endGame();
			}, 5000);
		},

		determineWinner(player, gotchi) {
			if (player === gotchi) return 'tie';
			
			if (
				(player === 'rock' && gotchi === 'scissors') ||
				(player === 'paper' && gotchi === 'rock') ||
				(player === 'scissors' && gotchi === 'paper')
			) {
				return 'YOU WIN';
			}
			
			return 'YOU LOSE';
		},

		endGame() {
			this.status = 'idle';
			this.currentGame = null;
			this.gameState = null;
			this.playerChoice = null;
			this.gotchiChoice = null;
			this.gameResult = null;
			this.tttBoard = [null, null, null, null, null, null, null, null, null];
			this.tttCurrentTurn = 'X';
			this.startIdle();
		},

		// ── Tic-Tac-Toe Game ──────────────────────────────────────────────────
		makeTTTMove(index) {
			if (this.gameState !== 'playing' || this.tttBoard[index] !== null) return;
			if (this.tttCurrentTurn !== this.tttPlayerSymbol) return;
			
			// Player makes move
			this.tttBoard[index] = this.tttPlayerSymbol;
			
			// Check for win/tie
			const result = this.checkTTTWinner();
			if (result) {
				this.endTTTGame(result);
				return;
			}
			
			// Switch to gotchi's turn
			this.tttCurrentTurn = this.tttGotchiSymbol;
			
			// Gotchi makes move after short delay
			setTimeout(() => {
				this.makeGotchiTTTMove();
			}, 600);
		},

		makeGotchiTTTMove() {
			if (this.gameState !== 'playing') return;
			
			// Smart AI with priority system
			const move = this.getBestTTTMove();
			if (move === -1) return;
			
			this.tttBoard[move] = this.tttGotchiSymbol;
			
			// Check for win/tie
			const result = this.checkTTTWinner();
			if (result) {
				this.endTTTGame(result);
				return;
			}
			
			// Switch back to player's turn
			this.tttCurrentTurn = this.tttPlayerSymbol;
		},

		getBestTTTMove() {
			const board = this.tttBoard;
			const ai = this.tttGotchiSymbol;
			const player = this.tttPlayerSymbol;
			
			// Priority 1: Win if possible
			for (let i = 0; i < 9; i++) {
				if (board[i] === null) {
					board[i] = ai;
					if (this.checkWinnerForSymbol(ai)) {
						board[i] = null; // Reset
						return i;
					}
					board[i] = null; // Reset
				}
			}
			
			// Priority 2: Block player from winning
			for (let i = 0; i < 9; i++) {
				if (board[i] === null) {
					board[i] = player;
					if (this.checkWinnerForSymbol(player)) {
						board[i] = null; // Reset
						return i;
					}
					board[i] = null; // Reset
				}
			}
			
			// Priority 3: Take center (best strategic position)
			if (board[4] === null) return 4;
			
			// Priority 4: Take corners
			const corners = [0, 2, 6, 8];
			const availableCorners = corners.filter(i => board[i] === null);
			if (availableCorners.length > 0) {
				return availableCorners[Math.floor(Math.random() * availableCorners.length)];
			}
			
			// Priority 5: Take any remaining empty cell
			for (let i = 0; i < 9; i++) {
				if (board[i] === null) return i;
			}
			
			return -1; // No moves available
		},

		checkWinnerForSymbol(symbol) {
			const board = this.tttBoard;
			const winPatterns = [
				[0, 1, 2], [3, 4, 5], [6, 7, 8], // rows
				[0, 3, 6], [1, 4, 7], [2, 5, 8], // columns
				[0, 4, 8], [2, 4, 6]             // diagonals
			];
			
			for (const pattern of winPatterns) {
				const [a, b, c] = pattern;
				if (board[a] === symbol && board[b] === symbol && board[c] === symbol) {
					return true;
				}
			}
			return false;
		},

		checkTTTWinner() {
			const board = this.tttBoard;
			const winPatterns = [
				[0, 1, 2], [3, 4, 5], [6, 7, 8], // rows
				[0, 3, 6], [1, 4, 7], [2, 5, 8], // columns
				[0, 4, 8], [2, 4, 6]             // diagonals
			];
			
			// Check for winner
			for (const pattern of winPatterns) {
				const [a, b, c] = pattern;
				if (board[a] && board[a] === board[b] && board[a] === board[c]) {
					return board[a] === this.tttPlayerSymbol ? 'YOU WIN' : 'YOU LOSE';
				}
			}
			
			// Check for tie
			if (board.every(cell => cell !== null)) {
				return 'TIE';
			}
			
			return null;
		},

		endTTTGame(result) {
			this.gameResult = result;
			
			// Show gotchi reaction
			this.stopAnimation();
			if (result === 'YOU WIN') {
				this.playAnimation('idle_sad');
				this.happiness = Math.min(100, this.happiness + 15);
				this.energy = Math.max(0, this.energy - 5);
			} else if (result === 'YOU LOSE') {
				this.playAnimation('idle_happy');
				this.happiness = Math.max(0, this.happiness - 5);
			} else {
				this.playAnimation('idle');
			}
			
			this.gameState = 'reveal';
			setTimeout(() => {
				this.endGame();
			}, 5000);
		},

		// ── Actions ───────────────────────────────────────────────────────────
		doEat() {
			if (!this.canAct) return;

			// Check for overfeeding
			if (this.energy >= 100) {
				this.overfeedCount++;
				
				// Escalating penalty: each overfeed hurts more
				const penalty = OVERFEED_BASE_PENALTY + (this.overfeedCount - 1) * 2;
				this.happiness = Math.max(0, this.happiness - penalty);

				// Check sickness threshold
				if (this.overfeedCount >= OVERFEED_SICKNESS_THRESHOLD) {
					this.startSickness();
					return;
				}

				// Show overfeeding animation
				this.status = 'overfeeding';
				this.playAnimation('overfeeding', () => this.startIdle());
			} else {
				this.energy = Math.min(100, this.energy + 25);
				this.happiness = Math.min(100, this.happiness + 5);
				this.status = 'eating';
				this.playAnimation('eating', () => this.startIdle());
			}
		},

		doPlay() {
			if (!this.canAct) return;
			this.happiness = Math.min(100, this.happiness + 20);
			this.energy    = Math.max(0, this.energy - 15);
			this.status = 'playing';
			this.playAnimation('playing', () => this.startIdle());
		},

		doPet() {
			if (!this.canAct) return;

			// Check for overpetting
			if (this.petCount >= OVERPET_THRESHOLD) {
				this.happiness = Math.max(0, this.happiness - OVERPET_HAPPINESS_PENALTY);
			} else {
				// Diminishing returns: less effective if overused
				const boost = Math.max(2, 8 - this.petCount * 0.5);
				this.happiness = Math.min(100, this.happiness + boost);
			}
			
			this.petCount++;
			this.status = 'petting';
			this.playAnimation('petting', () => this.startIdle());
		},

		doSleep() {
			if (this.isDead || this.isActing) return;
			if (this.isSleeping) {
				this.stopAnimation();
				this.happiness = Math.min(100, this.happiness + 10);
				this.status = 'waking';
				this.playAnimation('waking', () => this.startIdle());
			} else {
				this.stopAnimation();
				this.status = 'sleeping';
				this.playAnimation('sleeping');
			}
		},

		// ── Game loop ─────────────────────────────────────────────────────────
		die() {
			this.stopAnimation();
			this.status = 'dead';
			this.playAnimation('dead');
			clearInterval(this.decayInterval);
		},

		restart() {
			this.happiness = 80;
			this.energy = 80;
			this.mood = 80;
			this.petCount = 0;
			this.overfeedCount = 0;
			this.status = 'idle';
			this.startIdle();
			this.startDecay();
		},

		startDecay() {
			this.decayInterval = setInterval(() => {
				if (this.isDead) return;

				if (this.isSleeping) {
					this.energy = Math.min(100, this.energy + SLEEP_REGEN);
				} else {
					// Energy crisis: happiness decays faster when energy is low
					let happinessDecay = HAPPINESS_DECAY;
					if (this.energy < ENERGY_CRISIS_THRESHOLD) {
						// Scale up: at 0 energy, decay is 2x; at 25 energy, decay is 1x
						const factor = 1 + (ENERGY_CRISIS_THRESHOLD - this.energy) / ENERGY_CRISIS_THRESHOLD;
						happinessDecay = HAPPINESS_DECAY * factor;
					}
					this.happiness = Math.max(0, this.happiness - happinessDecay);
					this.energy    = Math.max(0, this.energy - ENERGY_DECAY);
				}

				// Update mood slowly based on happiness
				const moodDiff = this.happiness - this.mood;
				this.mood += moodDiff * MOOD_INERTIA;

				// Handle sickness
				if (this.energy < SICKNESS_THRESHOLD && this.status !== 'sick' && !this.isSleeping) {
					this.startSickness();
				} else if (this.energy >= SICKNESS_THRESHOLD && this.status === 'sick') {
					this.stopSickness();
				}

				// Handle overfeeding recovery (slowly reduce overfeed count)
				if (this.overfeedCount > 0 && this.energy < 90) {
					this.overfeedCount = Math.max(0, this.overfeedCount - 0.1);
				}

				// Handle attention-seeking
				if (this.happiness < ATTENTION_THRESHOLD && this.status !== 'attention' && !this.isSleeping && this.status !== 'sick') {
					this.startAttentionSeeking();
				} else if (this.happiness >= ATTENTION_THRESHOLD && this.status === 'attention') {
					this.stopAttentionSeeking();
				}

				// Reset pet count over time
				if (this.petCount > 0) {
					this.petCount = Math.max(0, this.petCount - 0.2);
				}

				if (this.happiness <= 0 || this.energy <= 0) {
					this.die();
				}
			}, DECAY_INTERVAL);
		},

		// ── Special states ───────────────────────────────────────────────────────
		startSickness() {
			this.stopAnimation();
			this.status = 'sick';
			this.playAnimation('sick');
		},

		stopSickness() {
			this.stopAnimation();
			this.startIdle();
		},

		startAttentionSeeking() {
			this.stopAnimation();
			this.status = 'attention';
			this.playAnimation('attention');

			// Flash screen to get attention
			this.attentionInterval = setInterval(() => {
				if (this.status !== 'attention') {
					clearInterval(this.attentionInterval);
					return;
				}
				// Screen flash effect handled by CSS
			}, 3000);
		},

		stopAttentionSeeking() {
			if (this.attentionInterval) {
				clearInterval(this.attentionInterval);
				this.attentionInterval = null;
			}
			this.stopAnimation();
			this.startIdle();
		},
	},

	created() {
		this.startIdle();
		this.startDecay();
	},

	beforeUnmount() {
		this.stopAnimation();
		clearInterval(this.decayInterval);
		if (this.attentionInterval) clearInterval(this.attentionInterval);
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
	height: 100%;
	max-height: 90vh;
	background-image: url("../assets/egg-shape.svg");
	background-position: center center;
	background-repeat: no-repeat;
	background-size: contain;
}

@media (max-width: 600px) {
	.gotchi-frame {
		height: 90vh;
	}
}

.frame-screen {
	position: relative;
	display: flex;
	background-color: #c8d8a0;
	border: solid 5px #000000;
	border-radius: 1rem;
	height: 20rem;
	width: 17rem;
	justify-content: center;
	margin-top: 4rem;
	margin-bottom: 1.5rem;
	box-shadow: inset 0 0 12px rgba(0,0,0,0.25);
	transition: background-color 0.3s ease;
}

@media (max-width: 600px) {
	.frame-screen {
		height: 14rem;
		width: 11.5rem;
		margin-top: 2rem;
		margin-bottom: 1.5rem;
		border-width: 6px;
	}
}

@media (max-width: 400px) {
	.frame-screen {
		height: 12rem;
		width: 10rem;
		margin-top: 1.5rem;
		margin-bottom: 1rem;
		border-width: 5px;
	}
}

.frame-screen.attention-flash {
	background-color: #ffeb3b;
	animation: attention-pulse 0.6s ease;
}

.status-bars-wrapper {
	display: flex;
	flex-direction: row;
	position: absolute;
	width: calc(100% - 1rem);
	margin-top: 0.5rem;
	margin-left: 0.5rem;
	margin-right: 0.5rem;
	gap: 0.5rem;
}

.status-bar {
	display: flex;
	align-items: center;
	justify-content: center;
	border: solid 3px #000000;
	border-radius: 0.3rem;
	font-weight: 700;
	font-size: 0.7rem;
	flex: 1;
	text-align: center;
	position: relative;
	overflow: hidden;
}

.status-bar .bar-value {
	height: 100%;
	position: absolute;
	left: 0;
	top: 0;
	z-index: 2;
	transition: width 0.4s ease, background-color 0.4s ease;
}

.status-bar .bar-label {
	z-index: 3;
	position: relative;
	padding-top: 1px;
	padding-bottom: 1px;
}

.frame-wrapper {
	width: 85%;
	align-self: end;
	margin-bottom: 0.5rem;
}

@media (max-width: 600px) {
	.frame-wrapper {
		margin-bottom: 0.3rem;
	}
}

.frame-wrapper.is-acting img {
	animation: sprite-bounce 0.35s ease infinite alternate;
}

.frame-wrapper > img {
	width: 100%;
	height: auto;
	image-rendering: crisp-edges;
	filter: grayscale(1);
	object-fit: contain;
}

.button-bar {
	display: flex;
	flex-wrap: wrap;
	justify-content: center;
	gap: 0.5rem;
	max-width: 69%;
	
}

.button-bar button {
	display: inline-flex;
	align-items: center;
	justify-content: center;
	padding: 0.5rem;
	border-radius: 100%;
	background-color: #FFFFFF;
	width: 2.5rem;
	aspect-ratio: 1/1;
	text-align: center;
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.8rem;
	font-weight: 900;
	border: solid 4px #000000;
	cursor: pointer;
	user-select: none;
	box-shadow: 1px 1px 5px rgba(0,0,0,0.5);
	transition: background-color 0.15s ease;
}

@media (max-width: 600px) {
	.button-bar button {
		width: 1.5rem;
		padding: 0.5rem;
		font-size: 0.75rem;
		border-width: 3px;
	}
}

.button-bar button:hover:not(:disabled) {
	background-color: salmon;
	color: #000000;
	transform: translateY(0.1rem);
	box-shadow: 1px 1px 3px rgba(0,0,0,0.5);
}

.button-bar button:active:not(:disabled) {
	transform: translateY(0.3rem);
	box-shadow: inset 1px 1px 3px rgba(0,0,0,0.4);
}

.button-bar button:disabled {
	opacity: 0.4;
	cursor: not-allowed;
}

.button-bar button.restart-btn {
	background-color: #e74c3c;
	color: #FFFFFF;
	font-weight: 900;
}

.button-bar button.restart-btn:hover:not(:disabled) {
	background-color: #c0392b;
}

@keyframes sprite-bounce {
	from { transform: translateY(0); }
	to   { transform: translateY(-6px); }
}

@keyframes attention-pulse {
	0%, 100% { background-color: #c8d8a0; }
	50% { background-color: #ffeb3b; }
}

/* ─── Rock Paper Scissors Game UI ───────────────────────────────────────────── */
.game-board {
	width: 100%;
	height: 100%;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
}

.game-gotchi-sprite {
	width: 50%;
	margin-bottom: 0.5rem;
	display: flex;
	justify-content: center;
	align-items: center;
}

.game-gotchi-sprite img {
	width: 100%;
	height: auto;
	image-rendering: crisp-edges;
	filter: grayscale(1);
	object-fit: contain;
}

.game-title {
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.8rem;
	font-weight: 900;
	color: #000;
	margin-bottom: 0.5rem;
	text-align: center;
}

.game-choices {
	display: flex;
	flex: 1;
	width: 100%;
	gap: 0.5rem;
	margin-bottom: 1rem;
}

.choice-btn {
	all: unset;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	aspect-ratio: 1/1;
	padding: 0.15rem;
	border: solid 3px #000;
	cursor: pointer;
	user-select: none;
	transition: all 0.15s ease;
	flex: 1;
}

.choice-btn:hover {
	transform: translateY(-1px);
	box-shadow: 1px 1px 4px rgba(0,0,0,0.6);
}

.choice-btn:active {
	transform: translateY(1px);
	box-shadow: inset 1px 1px 2px rgba(0,0,0,0.4);
}

.choice-btn span {
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.5rem;
	font-weight: 900;
	margin-top: 0.2rem;
}

.choice-icon {
	width: 20px;
	height: 20px;
}

/* Rock - Simple rounded square */
.rock-icon {
	background: #000;
	border-radius: 4px;
	position: relative;
}

.rock-icon::before {
	content: '';
	position: absolute;
	width: 4px;
	height: 4px;
	background: #c8d8a0;
	top: 4px;
	left: 4px;
}

/* Paper - Rectangle with lines */
.paper-icon {
	background: #000;
	position: relative;
	height: 24px;
}

.paper-icon::before,
.paper-icon::after {
	content: '';
	position: absolute;
	height: 2px;
	background: #c8d8a0;
	left: 3px;
	right: 3px;
}

.paper-icon::before { top: 6px; }
.paper-icon::after { top: 12px; }

/* Scissors - X shape */
.scissors-icon {
	position: relative;
}

.scissors-icon::before,
.scissors-icon::after {
	content: '';
	position: absolute;
	width: 3px;
	height: 100%;
	background: #000;
	left: 50%;
	transform-origin: center;
}

.scissors-icon::before { transform: rotate(45deg); }
.scissors-icon::after { transform: rotate(-45deg); }

.game-result {
	display: flex;
	flex-direction: column;
	align-items: center;
	width: 100%;
	margin-bottom: 1rem;
}

.choices-display {
	display: flex;
	align-items: center;
	justify-content: space-around;
	width: 100%;
	margin-bottom: 0.5rem;
}

.choice-display {
	display: flex;
	flex-direction: column;
	align-items: center;
}

.choice-display span {
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.5rem;
	font-weight: 900;
	margin-top: 0.2rem;
}

.vs-text {
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.6rem;
	font-weight: 900;
	color: #000;
}

.result-text {
	font-family: "Comic Sans MS", "Comic Sans", sans-serif;
	font-size: 0.8rem;
	font-weight: 900;
	padding: 0.2rem 0.5rem;
	border-radius: 0.3rem;
	margin-top: 0.3rem;
	border: solid 2px #000;
	color: black;
}

/* ─── Tic-Tac-Toe Game ───────────────────────────────────────────── */
.ttt-game {
	display: flex;
	justify-content: center;
	flex-direction: column;
	align-items: center;
}

.ttt-board {
	display: grid;
	grid-template-columns: repeat(3, 1fr);
	gap: 0.3rem;
	width: 100%;
	max-width: 5rem;
	aspect-ratio: 1/1;
	margin-bottom: 0.5rem;
}

.ttt-cell {
	all: unset;
	display: flex;
	align-items: center;
	justify-content: center;
	border: solid 2px #000;
	aspect-ratio: 1/1;
	cursor: pointer;
	user-select: none;
	transition: all 0.15s ease;
	font-size: 2rem;
	font-weight: 900;
	color: #000;
}

.ttt-cell:hover:not(:disabled) {
	transform: scale(1.05);
	box-shadow: 1px 1px 4px rgba(0,0,0,0.6);
}

.ttt-cell:active:not(:disabled) {
	transform: scale(0.95);
	box-shadow: inset 1px 1px 2px rgba(0,0,0,0.4);
}

.ttt-cell:disabled {
	cursor: default;
}

.ttt-x {
	font-size: 1.5rem;
	line-height: 1;
}

.ttt-o {
	font-size: 2rem;
	line-height: 1;
}

@media (max-width: 600px) {
	.game-title {
		font-size: 0.6rem;
	}
	
	.game-choices {
		gap: 0.2rem;
	}
	
	.choice-btn {
		padding: 0.2rem;
		border-width: 2px;
	}
	
	.choice-btn span {
		font-size: 0.4rem;
	}
	
	.choice-icon {
		width: 16px;
		height: 16px;
	}
	
	.result-text {
		font-size: 0.7rem;
	}
	
	.ttt-board {
		max-width: 7rem;
		gap: 0.2rem;
	}
	
	.ttt-cell {
		border-width: 2px;
		font-size: 1.3rem;
	}
	
	.ttt-x, .ttt-o {
		font-size: 1.5rem;
	}
}

</style>
