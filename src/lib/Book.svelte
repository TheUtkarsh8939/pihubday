<script>
    import { onMount } from 'svelte';

    // Book content for all pages
    const pages = [
        {
            left: `<h1>TBD</h1>
                <p>To be completed</p>`,
            right: `<h2>TBD</h2>
                `
        },
        {
            left: `<h1>TBD</h1>
            `,
            right: `<h2>TBD</h2>
               `
        },
        {
            left: ``,
            right: ``
        },
        {
            left: ``,
            right: ``
        }
    ];

    let currentPage = 0;
    let isAnimating = false;
    let isFlipped = false;
    let showBackContent = '';

    $: leftPageNum = currentPage * 2 + 1;
    $: rightPageNum = currentPage * 2 + 2;
    $: backPageNum = currentPage * 2 + 3;
    $: canGoPrev = currentPage > 0;
    $: canGoNext = currentPage < pages.length - 1;

    function nextPage() {
        if (isAnimating || !canGoNext) return;
        
        isAnimating = true;
        showBackContent = '';
        isFlipped = true;
        
        setTimeout(() => {
            currentPage = currentPage + 1;
            isFlipped = false;
            
            setTimeout(() => {
                isAnimating = false;
            }, 50);
        }, 700);
    }

    function prevPage() {
        if (isAnimating || !canGoPrev) return;
        
        isAnimating = true;
        showBackContent = pages[currentPage - 1].right;
        
        setTimeout(() => {
            isFlipped = true;
            
            setTimeout(() => {
                isFlipped = false;
                
                setTimeout(() => {
                    currentPage = currentPage - 1;
                    showBackContent = '';
                    isAnimating = false;
                }, 700);
            }, 50);
        }, 50);
    }

    function handleKeydown(event) {
        if (event.key === 'ArrowLeft' && canGoPrev) prevPage();
        if (event.key === 'ArrowRight' && canGoNext) nextPage();
    }

    onMount(() => {
        const keyHandler = (e) => handleKeydown(e);
        window.addEventListener('keydown', keyHandler);
        return () => window.removeEventListener('keydown', keyHandler);
    });
</script>

<div class="book-container">
    <div class="book">
        <div class="pages-wrapper">
            <div class="pages-container">
                <!-- Static left page -->
                <div class="page page-left">
                    <div class="page-content">
                        {@html pages[currentPage].left}
                    </div>
                    <div class="page-number">{leftPageNum}</div>
                </div>

                <!-- Flipping right page -->
                <div class="page-flip" class:flipped={isFlipped}>
                    <div class="page-front">
                        <div class="page-content">
                            {@html pages[currentPage].right}
                        </div>
                        <div class="page-number">{rightPageNum}</div>
                    </div>
                    <div class="page-back">
                        <div class="page-content">
                            {#if showBackContent}
                                {@html showBackContent}
                            {:else if currentPage < pages.length - 1}
                                {@html pages[currentPage + 1].left}
                            {/if}
                        </div>
                        {#if showBackContent || currentPage < pages.length - 1}
                            <div class="page-number">{backPageNum}</div>
                        {/if}
                    </div>
                </div>
            </div>

            <div class="navigation">
                <svg 
                    class="arrow-button arrow-left" 
                    class:disabled={!canGoPrev}
                    on:click={() => canGoPrev && prevPage()}
                    role="button"
                    tabindex={canGoPrev ? 0 : -1}
                    aria-label="Previous page"
                    viewBox="0 0 100 100"
                >
                    <circle cx="50" cy="50" r="45" fill="#8B4513" stroke="#654321" stroke-width="3"/>
                    <path d="M 60 25 L 35 50 L 60 75" fill="none" stroke="#FFD700" stroke-width="6" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                
                <svg 
                    class="arrow-button arrow-right" 
                    class:disabled={!canGoNext}
                    on:click={() => canGoNext && nextPage()}
                    role="button"
                    tabindex={canGoNext ? 0 : -1}
                    aria-label="Next page"
                    viewBox="0 0 100 100"
                >
                    <circle cx="50" cy="50" r="45" fill="#8B4513" stroke="#654321" stroke-width="3"/>
                    <path d="M 40 25 L 65 50 L 40 75" fill="none" stroke="#FFD700" stroke-width="6" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
            </div>
        </div>
    </div>
</div>

<style>
    :global(body) {
        display: flex;
        justify-content: center;
        align-items: center;
        min-height: 100vh;
        background: whitesmoke;
        font-family: 'Courier New', monospace;
        perspective: 2000px;
        margin: 0;
        padding: 0;
    }

    .book-container {
        position: relative;
        width: 100%;
        aspect-ratio: 3/2;
    }

    .book {
        position: relative;
        width: 100%;
        height: 100%;
        transform-style: preserve-3d;
    }

    .pages-wrapper {
        position: absolute;
        width: 100%;
        height: 100%;
        display: block;
    }

    .pages-container {
        position: relative;
        width: 100%;
        height: 100%;
        transform-style: preserve-3d;
    }

    .page {
        position: absolute;
        width: 50%;
        height: 100%;
        background: #FFF9E6;
        border: 4px solid #8B4513;
        padding: 40px 30px;
   
        box-shadow: 
            0 5px 20px rgba(0,0,0,0.2),
            inset 0 0 30px rgba(139, 69, 19, 0.1);
        overflow: hidden;
        transform-style: preserve-3d;
        backface-visibility: hidden;
    }

    .page::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: 
            repeating-linear-gradient(
                0deg,
                transparent,
                transparent 24px,
                rgba(139, 69, 19, 0.08) 24px,
                rgba(139, 69, 19, 0.08) 26px
            );
        pointer-events: none;
    }

    .page-left {
        left: 0;
        border-right: 2px solid #8B4513;
        border-top-left-radius: 8px;
        border-bottom-left-radius: 8px;
        margin-right:30%;
    }

    .page-right {
        right: 0;
        border-left: 2px solid #8B4513;
        border-top-right-radius: 8px;
        border-bottom-right-radius: 8px;
    }

    .page-flip {
        position: absolute;
        right: 0;
        width: 45%;
        height: 100%;
        transform-origin: left center;
        transform-style: preserve-3d;
        transition: transform 0.7s ease-in-out;
    }

    .page-flip .page-front,
    .page-flip .page-back {
        position: absolute;
        width: 100%;
        height: 100%;
        background: #FFF9E6;
        border: 4px solid #8B4513;
        padding: 40px 30px;
        box-shadow: 
            0 5px 20px rgba(0,0,0,0.2),
            inset 0 0 30px rgba(139, 69, 19, 0.1);
        overflow: hidden;
        backface-visibility: hidden;
        border-left: 2px solid #8B4513;
        border-top-right-radius: 8px;
        border-bottom-right-radius: 8px;
    }

    .page-flip .page-front::before,
    .page-flip .page-back::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: 
            repeating-linear-gradient(
                0deg,
                transparent,
                transparent 24px,
                rgba(139, 69, 19, 0.08) 24px,
                rgba(139, 69, 19, 0.08) 26px
            );
        pointer-events: none;
    }

    .page-flip .page-back {
        transform: rotateY(180deg);
        border-right: 2px solid #8B4513;
        border-left: none;
        border-top-left-radius: 8px;
        border-bottom-left-radius: 8px;
        border-top-right-radius: 0;
        border-bottom-right-radius: 0;
    }

    .page-flip.flipped {
        transform: rotateY(-180deg);
    }

    .page-content {
        position: relative;
        z-index: 1;
        font-size: 15px;
        line-height: 1.8;
        color: #2C1810;
    }

    .page-content :global(h1) {
        font-size: 22px;
        margin-bottom: 15px;
        color: #654321;
        border-bottom: 2px solid #8B4513;
        padding-bottom: 8px;
    }

    .page-content :global(h2) {
        font-size: 18px;
        margin-top: 15px;
        margin-bottom: 8px;
        color: #654321;
    }

    .page-content :global(p) {
        margin-bottom: 12px;
    }

    .page-number {
        position: absolute;
        bottom: 15px;
        font-size: 14px;
        color: #8B4513;
        font-weight: bold;
    }

    .page-left .page-number,
    .page-front .page-number {
        left: 30px;
    }

    .page-right .page-number,
    .page-back .page-number {
        right: 30px;
    }

    .navigation {
        position: absolute;
        bottom: 50%;
        width: 100%;
        display: flex;
        justify-content: space-between;
        padding: 0 -30px;
        z-index: 100;
        pointer-events: none;
        transform: translateY(50%);
    }

    .arrow-button {
        width: 50px;
        height: 50px;
        cursor: pointer;
        transition: all 0.3s ease;
        pointer-events: all;
        filter: drop-shadow(2px 2px 4px rgba(0,0,0,0.3));
    }

    .arrow-button:hover:not(.disabled) {
        transform: scale(1.2);
        filter: drop-shadow(3px 3px 6px rgba(0,0,0,0.5));
    }

    .arrow-button.disabled {
        opacity: 0.3;
        cursor: not-allowed;
    }

    .arrow-left {
        margin-left: -60px;
    }

    .arrow-right {
        margin-right: -60px;
    }
</style>
