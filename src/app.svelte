<script>
    import { fade } from "svelte/transition";
    import { cubicOut } from "svelte/easing";
    import Footer from "./lib/footer.svelte";
    import MuteIcon from "./lib/mute-icon.svelte";
    import VolumeIcon from "./lib/volume-icon.svelte";
    import MoonIcon from "./lib/moon-icon.svelte";
    import SunIcon from "./lib/sun-icon.svelte";

    const TIME_LIMIT = 60;
    const COMPLETION_BONUS = 100;

    let grid = [];
    let score = 0;
    let timeRemaining = TIME_LIMIT;
    let selectedSquares = [];
    let gameOver = false;
    let gameStarted = false;
    let visibleSquares = [];
    let gameWon = false;
    let timerInterval;
    let flashedScore = undefined;
    let muted = false;

    const prefersDarkScheme = window.matchMedia("(prefers-color-scheme: dark)");

    let darkMode = prefersDarkScheme.matches;

    $: console.log(darkMode);

    let matchSound = new Audio("matched.mp3");
    let wrongSound = new Audio("wrong.mp3");
    let winSound = new Audio("win.mp3");
    let timesUpSound = new Audio("times_up.mp3");

    function initializeGrid() {
        const letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
        const shuffledLetters = letters
            .split("")
            .sort(() => 0.5 - Math.random())
            .slice(0, 20);

        grid = shuffledLetters
            .flatMap((letter) => [
                { letter: letter.toLowerCase(), matched: false },
                { letter: letter.toUpperCase(), matched: false },
            ])
            .sort(() => 0.5 - Math.random());
    }

    function handleSquareClick(index) {
        if (gameOver || grid[index].matched) return;

        if (selectedSquares.includes(index)) {
            selectedSquares = selectedSquares.filter((i) => i !== index);
        } else {
            selectedSquares = [...selectedSquares, index];
        }

        if (selectedSquares.length === 2) {
            const [first, second] = selectedSquares;
            if (
                grid[first].letter.toLowerCase() ===
                    grid[second].letter.toLowerCase() &&
                grid[first].letter !== grid[second].letter
            ) {
                grid[first].matched = true;
                grid[second].matched = true;
                score += 20;
                flashScore(20);
                !muted && matchSound.play();
                checkForWin();
            } else {
                score = Math.max(0, score - 15);
                flashScore(-15);
                !muted && wrongSound.play();
            }
            setTimeout(() => {
                selectedSquares = [];
            }, 500);
            grid = [...grid];
        }
    }

    function flashScore(amount) {
        flashedScore = amount;
        setTimeout(() => {
            flashedScore = undefined;
        }, 400);
    }

    function checkForWin() {
        if (grid.every((square) => square.matched)) {
            gameWon = true;
            endGame();
        }
    }

    function endGame() {
        gameOver = true;
        clearInterval(timerInterval);
        if (gameWon) {
            !muted && winSound.play();
            const completionPoints = COMPLETION_BONUS + timeRemaining * 10;
            score += completionPoints;
        } else {
            !muted && timesUpSound.play();
        }
    }

    function startTimer() {
        timerInterval = setInterval(() => {
            timeRemaining--;
            if (timeRemaining <= 0) {
                clearInterval(timerInterval);
                endGame();
            }
        }, 1000);
    }

    function startGame() {
        gameStarted = true;
        gameOver = false;
        gameWon = false;
        score = 0;
        selectedSquares = [];
        timeRemaining = TIME_LIMIT;
        initializeGrid();
        staggerSquareAppearance();
        startTimer();
    }

    function toggleMute() {
        muted = !muted;
    }

    function toggleDarkMode() {
        darkMode = !darkMode;
        if (prefersDarkScheme.matches) {
            document.body.classList.toggle("light-theme");
        } else {
            document.body.classList.toggle("dark-theme");
        }
    }

    function staggerSquareAppearance() {
        const totalDuration = 300;
        const intervalDuration = totalDuration / grid.length;

        visibleSquares = [];
        const indexes = Array.from({ length: grid.length }, (_, i) => i);

        function showNextSquare() {
            if (indexes.length > 0) {
                const randomIndex = Math.floor(Math.random() * indexes.length);
                const squareIndex = indexes.splice(randomIndex, 1)[0];
                visibleSquares = [...visibleSquares, squareIndex];
                setTimeout(showNextSquare, intervalDuration);
            }
        }

        showNextSquare();
    }
</script>

<main class:dark-mode={darkMode}>
    <div class="game-info">
        <h1 class="title">Letter Matching Game</h1>
        <p>Time: {timeRemaining}s</p>
        <div class="controls">
            <button class="clear-button" on:click={toggleDarkMode}>
                {#if darkMode}
                    <MoonIcon />
                {:else}
                    <SunIcon />
                {/if}
            </button>
            <button class="clear-button" on:click={toggleMute}>
                {#if muted}
                    <MuteIcon />
                {:else}
                    <VolumeIcon />
                {/if}
            </button>
        </div>
    </div>

    {#if !gameStarted}
        <button class="start-button" on:click={startGame}>Start Game</button>
    {:else}
        <div class="grid" class:game-over={gameOver}>
            {#each grid as square, i}
                {#if visibleSquares.includes(i)}
                    <button
                        class="square"
                        class:selected={selectedSquares.includes(i)}
                        class:matched={square.matched}
                        on:click={() => handleSquareClick(i)}
                        transition:fade={{ duration: 500, easing: cubicOut }}
                    >
                        {square.letter}
                    </button>
                {/if}
            {/each}
        </div>

        {#if flashedScore}
            <div class="flashed-score" transition:fade={{ duration: 300 }}>
                {flashedScore > 0 ? "+" : ""}{flashedScore}
            </div>
        {/if}

        {#if gameOver}
            <div class="game-over-content" transition:fade={{ duration: 300 }}>
                <h2>🎉 Game Over!</h2>
                {#if gameWon}
                    <p>Completed in {TIME_LIMIT - timeRemaining} seconds.</p>
                    <p>{COMPLETION_BONUS} points completion bonus!</p>
                    <p>Plus {timeRemaining * 10} points for speed.</p>
                {/if}
                <h3>Final Score: {score}</h3>
                <button class="play-again-button" on:click={startGame}>
                    Play Again
                </button>
            </div>
        {/if}
    {/if}
</main>

<Footer />

<style>
    main {
        display: flex;
        background-color: var(--background-color);
        color: var(--text-color);
        transition:
            background-color 0.3s,
            color 0.3s;
        flex-direction: column;
        justify-content: space-between;
        align-items: center;
        max-width: 600px;
        padding: 0 0.5rem;
        margin: 0 auto;
        position: relative;
        min-height: calc(100vh - 3.7rem);
    }

    .game-info {
        display: flex;
        padding: 1rem;
        justify-content: space-between;
        align-items: baseline;
        width: 100%;
        margin-bottom: 20px;
    }

    .grid {
        display: grid;
        grid-template-columns: repeat(8, 1fr);
        gap: 10px;
        width: 100%;
    }

    .grid.game-over {
        opacity: 0.2;
        pointer-events: none;
        transition: opacity 0.3s;
    }
    .title {
        color: var(--title-color);
    }

    .square {
        aspect-ratio: 1;
        font-size: clamp(16px, 4vw, 24px);
        font-family: inherit;
        border: 1px solid var(--square-border);
        border-radius: 5px;
        color: var(--text-color);
        background-color: var(--square-bg);
        cursor: pointer;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        transition:
            background-color 0.3s,
            opacity 0.5s;
        padding: 0;
    }
    @media (max-width: 600px) {
        .game-info {
            flex-wrap: wrap;
        }
        .title {
            margin: 0 0 0.5rem;
            padding: 0;
            width: 100%;
            order: -1;
            clear: both;
        }
    }

    .square:hover:not(:disabled) {
        background-color: var(--square-hover);
    }

    .square.selected {
        background-color: var(--square-selected);
        border: 2px solid var(--square-selected-border);
        box-shadow: 0 3px 6px rgba(0, 0, 0, 0.2);
    }

    .square.matched {
        opacity: 0.05;
        cursor: default;
        border: none;
        box-shadow: none;
    }

    .game-over-content {
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        background-color: white;
        padding: 20px;
        border: 1px solid var(--square-border);
        background-color: var(--background-color);
        border-radius: 10px;
        text-align: center;
        z-index: 10;
    }
    .game-over-content h3 {
        margin: 2rem 0 1rem;
    }
    .game-over-content h2 {
        margin: 1rem 0 2rem;
    }

    .flashed-score {
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        color: var(--title-color);
        font-size: 2rem;
    }

    .start-button,
    .play-again-button {
        font-size: 1rem;
        padding: 1rem 2rem;
        margin: 1rem;
        background-color: var(--button-bg);
        color: var(--button-text);
        border: none;
        border-radius: 5px;
        cursor: pointer;
        transition: background-color 0.3s;
    }

    .start-button:hover,
    .play-again-button:hover {
        background-color: var(--button-hover);
    }
    .clear-button {
        background-color: transparent;
        border: none;
        color: var(--text-color);
        cursor: pointer;
    }
</style>
