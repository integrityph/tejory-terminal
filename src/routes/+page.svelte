<script lang="ts">
    import { goto } from "$app/navigation";
    import "../app.css";
    import Loader from "./loader.svelte";

    const currencies = [
        {
            id: "btc",
            label: "Bitcoin (Lightning)",
            disabled: false,
            icon: `btc.svg`
        },
        {
            id: "usdc",
            label: "USDC (Solana)",
            disabled: false,
            icon: `usdc.svg`
        },
        {
            id: "usdt",
            label: "USDT (TRON)",
            disabled: true,
            icon: `usdt.svg`
        }
    ];

    function handleSelect(coinId: string) {
        goto(`Exchange#${coinId}`);
    }
</script>

<main class="relative h-dvh w-full bg-transparent">
    <Loader />
    <div
        style="opacity:0.8; background-image:url(pattern.png);"
        class="absolute z-[-2] flex h-dvh w-full items-center justify-center bg-neutral-500"
    ></div>
    <div
        style="background-image: linear-gradient(to bottom, rgba(141, 141, 141, 0.45), rgba(0, 0, 0, 0.91));"
        class="absolute z-[-1] flex h-dvh w-full items-center justify-center"
    ></div>
    
    <div class="main-content absolute flex w-full flex-col items-end justify-end p-5 md:h-dvh">
        {#each currencies as coin, i}
            <button
                disabled={coin.disabled}
                onclick={() => handleSelect(coin.id)}
                type="button"
                class="
                    flex w-full items-center justify-center gap-5 rounded bg-neutral-700 px-4 py-3 text-white transition-colors
                    hover:bg-neutral-600 
                    disabled:blur-[0.8px] disabled:hover:bg-neutral-700 disabled:cursor-not-allowed
                    {i === currencies.length - 1 ? 'mb-24' : 'mb-5'}
                "
            >
                <img alt={coin.id} src={coin.icon}>
                <p class="w-36">{coin.label}</p>
            </button>
        {/each}
    </div>
</main>