<script lang="ts">
	import { onMount } from 'svelte';
	import { SvelteSet } from 'svelte/reactivity';

	// Think of these as small shared types you'd often keep in a React app's `types.ts`.
	type Bullet = {
		id: number;
		x: number;
		y: number;
	};

	type Ball = {
		id: number;
		x: number;
		y: number;
		vx: number;
		vy: number;
		radius: number;
	};

	const GUN_WIDTH = 36;
	const GUN_HEIGHT = 72;
	const TOUCH_MOVE_ZONE_WIDTH = GUN_WIDTH + 10;
	const GUN_SPEED = 420;
	const BULLET_SPEED = 900;
	const BULLET_COOLDOWN_MS = 120;
	const BULLET_RADIUS = 4;

	const SHOTS_PER_BALL = 5;
	const ROUND_TRANSITION_MS = 900;
	const START_BALL_RADIUS_BASE = 30;
	const START_BALL_RADIUS_LEVEL_MULTIPLIER = 5;
	const AREA_CONSERVING_SPLIT_RADIUS_FACTOR = Math.SQRT1_2;

	function countBallsInSplitTree(radius: number, minSplitRadius: number): number {
		const nextRadius = radius * AREA_CONSERVING_SPLIT_RADIUS_FACTOR;

		if (nextRadius < minSplitRadius) {
			return 1;
		}

		return 1 + 2 * countBallsInSplitTree(nextRadius, minSplitRadius);
	}

	let currentLevelNumber = $state(1);
	let currentLevelName = $derived(`Level ${currentLevelNumber}`);
	let startBallRadius = $derived(
		currentLevelNumber * START_BALL_RADIUS_LEVEL_MULTIPLIER + START_BALL_RADIUS_BASE
	);
	let minSplitRadius = $derived(Math.max(20, Math.round(startBallRadius * 0.12)));
	let startBallSpeedX = $derived(150);
	let startBallSpeedY = $derived(130);
	let splitSpeedBoost = $derived(1 + currentLevelNumber / 10);
	let maxBallsForLevel = $derived(countBallsInSplitTree(startBallRadius, minSplitRadius));
	let maxBulletsForLevel = $derived(maxBallsForLevel * SHOTS_PER_BALL);

	let gameWidth = $state(0);
	let gameHeight = $state(0);
	let gameElement: HTMLDivElement | undefined;

	let gunY = $state(0);
	let bullets = $state<Bullet[]>([]);
	let balls = $state<Ball[]>([]);
	let gameOver = $state(false);
	let win = $state(false);
	let gameOverReason = $state('');
	let isRoundTransition = $state(false);
	let upcomingLevelNumber = $state(2);
	let showHelpModal = $state(false);

	let bulletIdCounter = 0;
	let ballIdCounter = 0;
	let shotsFired = $state(0);
	let lastShotAt = 0;
	let lastFrameAt = 0;
	let moveTouchPointerId: number | null = null;
	let moveTouchStartY = 0;
	let moveTouchStartGunY = 0;
	let roundTransitionTimeout: ReturnType<typeof setTimeout> | null = null;

	// Keyboard input state: in React you'd often keep this in refs to avoid rerenders.
	// In Svelte, this `Set` lives in module state and we read it inside the frame loop.
	const pressedKeys = new SvelteSet<string>();

	function randomInRange(min: number, max: number): number {
		return Math.random() * (max - min) + min;
	}

	function randomSign(): number {
		return Math.random() < 0.5 ? -1 : 1;
	}

	function clearRoundTransitionTimeout() {
		if (roundTransitionTimeout) {
			clearTimeout(roundTransitionTimeout);
			roundTransitionTimeout = null;
		}
	}

	function buildStartingBall(): Ball {
		const effectiveWidth = Math.max(gameWidth, 800);
		const effectiveHeight = Math.max(gameHeight, 500);
		const minX = Math.max(effectiveWidth / 2 + startBallRadius, startBallRadius);
		const maxX = Math.max(minX, effectiveWidth - startBallRadius);
		const minY = startBallRadius;
		const maxY = Math.max(minY, effectiveHeight - startBallRadius);

		return {
			id: ++ballIdCounter,
			x: randomInRange(minX, maxX),
			y: randomInRange(minY, maxY),
			vx: startBallSpeedX * randomSign(),
			vy: startBallSpeedY * randomSign(),
			radius: startBallRadius
		};
	}

	function resetGame(levelNumber = currentLevelNumber) {
		clearRoundTransitionTimeout();
		currentLevelNumber = levelNumber;
		bulletIdCounter = 0;
		ballIdCounter = 0;
		shotsFired = 0;
		lastShotAt = 0;
		gameOver = false;
		win = false;
		gameOverReason = '';
		isRoundTransition = false;
		upcomingLevelNumber = levelNumber + 1;
		bullets = [];
		gunY = Math.max(0, gameHeight / 2 - GUN_HEIGHT / 2);
		balls = [buildStartingBall()];
	}

	function startNextLevel() {
		resetGame(currentLevelNumber + 1);
	}

	function queueNextLevelTransition() {
		if (gameOver || isRoundTransition) return;

		isRoundTransition = true;
		upcomingLevelNumber = currentLevelNumber + 1;
		pressedKeys.clear();

		clearRoundTransitionTimeout();
		roundTransitionTimeout = setTimeout(() => {
			startNextLevel();
		}, ROUND_TRANSITION_MS);
	}

	function shoot(now: number) {
		if (now - lastShotAt < BULLET_COOLDOWN_MS) return;
		if (shotsFired >= maxBulletsForLevel) return;
		if (gameOver || win || isRoundTransition) return;

		lastShotAt = now;
		shotsFired += 1;
		bullets = [
			...bullets,
			{
				id: ++bulletIdCounter,
				x: GUN_WIDTH,
				y: gunY + GUN_HEIGHT / 2
			}
		];
	}

	function startMove(direction: 'ArrowUp' | 'ArrowDown') {
		pressedKeys.add(direction);
	}

	function endMove(direction: 'ArrowUp' | 'ArrowDown') {
		pressedKeys.delete(direction);
	}

	function fireFromButton() {
		shoot(performance.now());
	}

	function isInTouchMoveZone(clientX: number): boolean {
		if (!gameElement) return false;
		const rect = gameElement.getBoundingClientRect();
		return clientX - rect.left <= TOUCH_MOVE_ZONE_WIDTH;
	}

	function startTouchMove(event: PointerEvent) {
		if (event.pointerType !== 'touch') return;
		moveTouchPointerId = event.pointerId;
		moveTouchStartY = event.clientY;
		moveTouchStartGunY = gunY;
		event.preventDefault();
		event.stopPropagation();
	}

	function continueTouchMove(event: PointerEvent) {
		if (event.pointerId !== moveTouchPointerId) return;
		const deltaY = event.clientY - moveTouchStartY;
		const nextGunY = moveTouchStartGunY + deltaY;
		gunY = Math.max(0, Math.min(gameHeight - GUN_HEIGHT, nextGunY));
		event.preventDefault();
		event.stopPropagation();
	}

	function endTouchMove(event: PointerEvent) {
		if (event.pointerId !== moveTouchPointerId) return;
		moveTouchPointerId = null;
		moveTouchStartY = 0;
		moveTouchStartGunY = gunY;
		event.preventDefault();
		event.stopPropagation();
	}

	function fireFromGameTap(event: PointerEvent) {
		if (event.pointerType !== 'touch') return;
		if (isInTouchMoveZone(event.clientX)) return;
		shoot(performance.now());
	}

	function closeHelpModal() {
		showHelpModal = false;
	}

	function handleHelpBackdropKeydown(event: KeyboardEvent) {
		if (event.key === 'Escape' || event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			closeHelpModal();
		}
	}

	function circlePointCollision(ball: Ball, pointX: number, pointY: number): boolean {
		const dx = pointX - ball.x;
		const dy = pointY - ball.y;
		return dx * dx + dy * dy <= ball.radius * ball.radius;
	}

	function circleRectCollision(
		ball: Ball,
		rx: number,
		ry: number,
		rw: number,
		rh: number
	): boolean {
		const closestX = Math.max(rx, Math.min(ball.x, rx + rw));
		const closestY = Math.max(ry, Math.min(ball.y, ry + rh));
		const dx = ball.x - closestX;
		const dy = ball.y - closestY;
		return dx * dx + dy * dy <= ball.radius * ball.radius;
	}

	function splitBall(ball: Ball): Ball[] {
		// Area-conserving split: 2 * pi * r_child^2 = pi * r_parent^2 => r_child = r_parent / sqrt(2)
		const nextRadius = ball.radius * AREA_CONSERVING_SPLIT_RADIUS_FACTOR;

		if (nextRadius < minSplitRadius) {
			// Too small to split again, so this hit fully destroys it.
			return [];
		}

		const speedX = Math.abs(ball.vx) * splitSpeedBoost;
		const speedY = Math.abs(ball.vy) * splitSpeedBoost;

		return [
			{
				id: ++ballIdCounter,
				x: ball.x,
				y: ball.y,
				vx: -speedX,
				vy: -speedY,
				radius: nextRadius
			},
			{
				id: ++ballIdCounter,
				x: ball.x,
				y: ball.y,
				vx: speedX,
				vy: speedY,
				radius: nextRadius
			}
		];
	}

	function tick(now: number) {
		if (!lastFrameAt) lastFrameAt = now;
		const dt = Math.min((now - lastFrameAt) / 1000, 0.033);
		lastFrameAt = now;

		if (gameHeight > 0) {
			gunY = Math.max(0, Math.min(gameHeight - GUN_HEIGHT, gunY));
		}

		if (!gameOver && !win && !isRoundTransition && gameWidth > 0 && gameHeight > 0) {
			if (pressedKeys.has('ArrowUp')) {
				gunY -= GUN_SPEED * dt;
			}
			if (pressedKeys.has('ArrowDown')) {
				gunY += GUN_SPEED * dt;
			}

			gunY = Math.max(0, Math.min(gameHeight - GUN_HEIGHT, gunY));

			bullets = bullets
				.map((bullet) => ({
					...bullet,
					x: bullet.x + BULLET_SPEED * dt
				}))
				.filter((bullet) => bullet.x - BULLET_RADIUS < gameWidth);

			balls = balls.map((ball) => {
				let nextX = ball.x + ball.vx * dt;
				let nextY = ball.y + ball.vy * dt;
				let nextVx = ball.vx;
				let nextVy = ball.vy;

				if (nextY - ball.radius <= 0) {
					nextY = ball.radius;
					nextVy = Math.abs(nextVy);
				} else if (nextY + ball.radius >= gameHeight) {
					nextY = gameHeight - ball.radius;
					nextVy = -Math.abs(nextVy);
				}

				if (nextX - ball.radius <= 0) {
					nextX = ball.radius;
					nextVx = Math.abs(nextVx);
				} else if (nextX + ball.radius >= gameWidth) {
					nextX = gameWidth - ball.radius;
					nextVx = -Math.abs(nextVx);
				}

				return {
					...ball,
					x: nextX,
					y: nextY,
					vx: nextVx,
					vy: nextVy
				};
			});

			// Resolve bullet collisions.
			// Similar to a reducer pass in React: consume current arrays and produce next arrays.
			const consumedBullets = new SvelteSet<number>();
			const nextBalls: Ball[] = [];

			for (const ball of balls) {
				let wasHit = false;

				for (const bullet of bullets) {
					if (consumedBullets.has(bullet.id)) continue;
					if (circlePointCollision(ball, bullet.x, bullet.y)) {
						wasHit = true;
						consumedBullets.add(bullet.id);
						break;
					}
				}

				if (wasHit) {
					nextBalls.push(...splitBall(ball));
				} else {
					nextBalls.push(ball);
				}
			}

			bullets = bullets.filter((bullet) => !consumedBullets.has(bullet.id));
			balls = nextBalls;

			if (balls.some((ball) => circleRectCollision(ball, 0, gunY, GUN_WIDTH, GUN_HEIGHT))) {
				gameOver = true;
				gameOverReason = 'A ball hit your gun.';
			}

			if (!gameOver && !win && shotsFired >= maxBulletsForLevel && bullets.length === 0) {
				gameOver = true;
				gameOverReason = 'You ran out of bullets.';
			}

			if (!gameOver && balls.length === 0) {
				queueNextLevelTransition();
			}
		}

		requestAnimationFrame(tick);
	}

	onMount(() => {
		const onKeyDown = (event: KeyboardEvent) => {
			if (event.key === 'ArrowUp' || event.key === 'ArrowDown' || event.key === ' ') {
				event.preventDefault();
			}

			pressedKeys.add(event.key);

			if (event.key === ' ') {
				shoot(performance.now());
			}
		};

		const onKeyUp = (event: KeyboardEvent) => {
			pressedKeys.delete(event.key);
		};

		window.addEventListener('keydown', onKeyDown);
		window.addEventListener('keyup', onKeyUp);

		resetGame();
		requestAnimationFrame(tick);

		return () => {
			clearRoundTransitionTimeout();
			window.removeEventListener('keydown', onKeyDown);
			window.removeEventListener('keyup', onKeyUp);
		};
	});
</script>

<svelte:window on:blur={() => pressedKeys.clear()} />

<main>
	<header>
		<div class="header-copy">
			<h1>Split Shot: {currentLevelName}</h1>
			<p>Bullets left: {maxBulletsForLevel - shotsFired}</p>
		</div>

		<button
			type="button"
			class="help-button"
			aria-label="How to play"
			onclick={() => (showHelpModal = true)}
		>
			?
		</button>
	</header>

	<div
		class="game"
		role="application"
		aria-label="Game playfield"
		bind:this={gameElement}
		bind:clientWidth={gameWidth}
		bind:clientHeight={gameHeight}
		onpointerdown={fireFromGameTap}
	>
		<button
			type="button"
			class="touch-move-zone"
			aria-label="Touch and slide to move gun"
			onpointerdown={startTouchMove}
			onpointermove={continueTouchMove}
			onpointerup={endTouchMove}
			onpointercancel={endTouchMove}
		></button>

		<div class="gun" style={`top: ${gunY}px; width: ${GUN_WIDTH}px; height: ${GUN_HEIGHT}px;`}>
			<div class="barrel"></div>
		</div>

		{#each bullets as bullet (bullet.id)}
			<div
				class="bullet"
				style={`left: ${bullet.x - BULLET_RADIUS}px; top: ${bullet.y - BULLET_RADIUS}px; width: ${BULLET_RADIUS * 2}px; height: ${BULLET_RADIUS * 2}px;`}
			></div>
		{/each}

		{#each balls as ball (ball.id)}
			<div
				class="ball"
				style={`left: ${ball.x - ball.radius}px; top: ${ball.y - ball.radius}px; width: ${ball.radius * 2}px; height: ${ball.radius * 2}px;`}
			></div>
		{/each}

		{#if gameOver}
			<div class="overlay">
				<h2>You lost</h2>
				<p>{gameOverReason}</p>
				<button onclick={() => resetGame()}>Restart</button>
			</div>
		{:else if isRoundTransition}
			<div class="overlay">
				<h2>Round {upcomingLevelNumber}</h2>
				<p>Get ready...</p>
			</div>
		{/if}
	</div>

	<div class="mobile-controls" aria-label="Mobile controls">
		<button
			type="button"
			class="control-button"
			onpointerdown={() => startMove('ArrowUp')}
			onpointerup={() => endMove('ArrowUp')}
			onpointercancel={() => endMove('ArrowUp')}
			onpointerleave={() => endMove('ArrowUp')}
		>
			Up
		</button>

		<button type="button" class="control-button fire-button" onclick={fireFromButton}>Fire</button>

		<button
			type="button"
			class="control-button"
			onpointerdown={() => startMove('ArrowDown')}
			onpointerup={() => endMove('ArrowDown')}
			onpointercancel={() => endMove('ArrowDown')}
			onpointerleave={() => endMove('ArrowDown')}
		>
			Down
		</button>
	</div>

	{#if showHelpModal}
		<div
			class="help-modal-backdrop"
			role="button"
			aria-label="Close help"
			tabindex="0"
			onclick={closeHelpModal}
			onkeydown={handleHelpBackdropKeydown}
		>
			<div
				class="help-modal"
				role="dialog"
				aria-modal="true"
				aria-label="How to play"
				tabindex="-1"
				onpointerdown={(event) => event.stopPropagation()}
			>
				<div class="help-modal-header">
					<h2>How to play</h2>
					<button
						type="button"
						class="help-close-button"
						aria-label="Close help"
						onclick={closeHelpModal}
					>
						×
					</button>
				</div>

				<p>Desktop controls</p>
				<ul>
					<li>Arrow Up and Arrow Down to move.</li>
					<li>Space bar to fire.</li>
				</ul>

				<p>Mobile controls</p>
				<ul>
					<li>Slide your finger on the left edge to move up and down.</li>
					<li>Tap the playfield or Fire button to shoot.</li>
				</ul>

				<p>Goal</p>
				<ul>
					<li>Hit bouncing balls to split and clear them.</li>
					<li>Do not let a ball hit your gun.</li>
					<li>Do not run out of bullets.</li>
				</ul>
			</div>
		</div>
	{/if}
</main>

<style>
	main {
		height: 100vh;
		display: grid;
		grid-template-rows: auto 1fr auto;
		padding: 0.7rem;
		gap: 0.55rem;
		background:
			radial-gradient(circle at 20% 20%, #16213e 0%, transparent 30%),
			radial-gradient(circle at 85% 10%, #0f3460 0%, transparent 25%),
			linear-gradient(180deg, #081126 0%, #03070f 100%);
		color: #e8f0ff;
	}

	header {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: 0.5rem;
	}

	.header-copy {
		display: grid;
		gap: 0.1rem;
	}

	header h1 {
		margin: 0;
		font-size: clamp(1rem, 1.4vw, 1.25rem);
	}

	header p {
		margin: 0;
		font-size: clamp(0.8rem, 1vw, 0.9rem);
		opacity: 0.85;
	}

	.help-button {
		width: 2rem;
		height: 2rem;
		padding: 0;
		border-radius: 9999px;
		background: #a9d9ff;
		font-size: 1rem;
		line-height: 1;
		font-weight: 800;
	}

	.game {
		position: relative;
		overflow: hidden;
		touch-action: manipulation;
		border-radius: 12px;
		border: 1px solid #32507c;
		box-shadow: inset 0 0 0 1px #172746;
		background:
			linear-gradient(0deg, rgba(27, 47, 78, 0.4), rgba(27, 47, 78, 0.1)),
			repeating-linear-gradient(
				0deg,
				rgba(148, 176, 225, 0.08) 0px,
				rgba(148, 176, 225, 0.08) 1px,
				transparent 1px,
				transparent 28px
			);
	}

	.touch-move-zone {
		border: 0;
		padding: 0;
		margin: 0;
		background: transparent;
		color: transparent;
		border-radius: 0;
		cursor: default;
		font-size: 0;
		position: absolute;
		left: 0;
		top: 0;
		bottom: 0;
		width: 46px;
		touch-action: none;
		z-index: 4;
	}

	.gun {
		position: absolute;
		left: 0;
		border-radius: 0 8px 8px 0;
		background: linear-gradient(180deg, #fca311 0%, #d07e00 100%);
		display: grid;
		place-items: center;
	}

	.barrel {
		position: absolute;
		right: -14px;
		width: 16px;
		height: 14px;
		border-radius: 0 7px 7px 0;
		background: #ffd166;
	}

	.bullet {
		position: absolute;
		background: #fff3b0;
		border-radius: 9999px;
	}

	.ball {
		position: absolute;
		border-radius: 9999px;
		background: radial-gradient(circle at 30% 28%, #9ad1ff 0%, #4a8cca 42%, #1d4f80 100%);
		box-shadow:
			inset -6px -8px 12px rgba(7, 18, 31, 0.4),
			0 0 14px rgba(88, 171, 255, 0.36);
	}

	.overlay {
		position: absolute;
		inset: 0;
		display: grid;
		place-content: center;
		gap: 0.6rem;
		justify-items: center;
		background: rgba(5, 10, 22, 0.7);
		backdrop-filter: blur(2px);
		text-align: center;
	}

	.overlay h2,
	.overlay p {
		margin: 0;
	}

	button {
		border: 0;
		border-radius: 8px;
		padding: 0.5rem 0.9rem;
		cursor: pointer;
		font-weight: 700;
		background: #7cc9ff;
		color: #00182c;
	}

	.mobile-controls {
		display: none;
		grid-template-columns: 1fr 1fr 1fr;
		gap: 0.6rem;
	}

	.control-button {
		padding: 0.75rem 0.5rem;
		font-size: 1rem;
		font-weight: 700;
		touch-action: none;
		user-select: none;
	}

	.fire-button {
		background: #ffb454;
	}

	.help-modal-backdrop {
		position: fixed;
		inset: 0;
		z-index: 30;
		display: grid;
		place-items: center;
		padding: 1rem;
		background: rgba(2, 7, 18, 0.7);
		backdrop-filter: blur(3px);
	}

	.help-modal {
		width: min(560px, 100%);
		max-height: min(80vh, 680px);
		overflow: auto;
		padding: 1rem;
		border-radius: 12px;
		background: #0f1b33;
		border: 1px solid #395a8d;
		box-shadow: 0 10px 40px rgba(0, 0, 0, 0.4);
	}

	.help-modal-header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.5rem;
		margin-bottom: 0.25rem;
	}

	.help-modal h2 {
		margin: 0;
		font-size: 1.05rem;
	}

	.help-modal p {
		margin: 0.75rem 0 0.25rem;
		font-weight: 700;
	}

	.help-modal ul {
		margin: 0;
		padding-left: 1.1rem;
		opacity: 0.92;
	}

	.help-close-button {
		width: 1.8rem;
		height: 1.8rem;
		padding: 0;
		border-radius: 9999px;
		font-size: 1.2rem;
		line-height: 1;
		background: #9bcfff;
	}

	@media (max-width: 720px) {
		main {
			padding: 0.55rem;
			gap: 0.5rem;
		}

		.mobile-controls {
			display: grid;
		}
	}
</style>
