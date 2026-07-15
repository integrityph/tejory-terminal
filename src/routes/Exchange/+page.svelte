<script>
    import { onMount } from "svelte";
    import { goto } from "$app/navigation";
    import "../../app.css";

    // Core state
    let cryptoId = $state("btc"); // Default, overwritten by URL hash
    let fiatAmount = $state(""); 
    let selectedCurrency = $state("usd");
    let exchangeRates = $state({});
    let currencyList = $state(["usd", "php"]);
    let invoice = $state("");

    // Computed / Derived State
    let isStablecoin = $derived(["usdc", "usdt"].includes(cryptoId));
    let cryptoSymbol = $derived(cryptoId === "btc" ? "BTC" : cryptoId.toUpperCase());
    let parsedFiat = $derived(parseFloat(fiatAmount) || 0);

    // Dynamic calculations based on token type
    let cryptoAmount = $derived.by(() => {
        if (!exchangeRates[selectedCurrency]) return 0;
        
        if (isStablecoin) {
            // Convert selected fiat to USD, then 1:1 with stablecoin
            const usdRate = exchangeRates["usd"]; 
            const fiatRate = exchangeRates[selectedCurrency]; 
            const conversionToUsd = fiatRate / usdRate;
            return parsedFiat / conversionToUsd;
        } else {
            // Direct fiat to BTC conversion
            return parsedFiat / exchangeRates[selectedCurrency];
        }
    });

    // Compute base USD format for your serverless function
    let fiatAmountBaseUSDB = $derived.by(() => {
        if (!exchangeRates[selectedCurrency] || !parsedFiat) return "0";
        const usdRate = exchangeRates["usd"];
        const fiatRate = exchangeRates[selectedCurrency];
        return ((parsedFiat / (fiatRate / usdRate)).toFixed(6) * 1_000_000).toString();
    });

    // Display string for the exchange rate hint
    let exchangeRateHint = $derived.by(() => {
        if (!exchangeRates[selectedCurrency]) return 'Loading rate...';
        if (isStablecoin) {
            const usdRate = exchangeRates["usd"];
            const fiatRate = exchangeRates[selectedCurrency];
            const conversionToUsd = fiatRate / usdRate;
            return `1 ${cryptoSymbol} = ${conversionToUsd.toLocaleString(undefined, {maximumFractionDigits: 2})} ${selectedCurrency.toUpperCase()}`;
        } else {
            return `1 BTC = ${exchangeRates[selectedCurrency].toLocaleString()} ${selectedCurrency.toUpperCase()}`;
        }
    });

    onMount(() => {
        // Grab the coin ID from the URL (e.g., #btc, #usdc)
        const hash = window.location.hash.substring(1);
        if (hash) cryptoId = hash;

        let current = localStorage.getItem("Currency");
        if (current != null) selectedCurrency = current;
        
        fetchExchangeRates();

        const interval = setInterval(() => {
            if (invoice) checkPayment(invoice.payment_hash);
        }, 10000);
        return () => clearInterval(interval);
    });

    const priceProviders = [fetchRateFromCoingecko];
    
    async function fetchExchangeRates() {
        let rate = null;
        for (let i = 0; i < priceProviders.length; i++) {
            rate = await priceProviders[i](); 
            if (rate !== null) break;
        }
        if (rate) exchangeRates = rate;
    }

    async function fetchRateFromCoingecko() {
        try {
            const resRates = await fetch(
                `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=${currencyList.join(",")}`
            );
            const data = await resRates.json();
            return data.bitcoin;
        } catch (error) {
            console.error("Error fetching exchange rates:", error);
            return null;
        }
    }

    function handleKeypadInput(value) {
        if (value === ".") {
            if (!fiatAmount.includes(".")) fiatAmount += ".";
        } else {
            fiatAmount += value.toString();
        }
    }

    function handleGenerate() {
        localStorage.setItem("Currency", selectedCurrency);
        // Pass the cryptoId along so the QR page knows which address/network to use
        goto(`GenerateQR#${cryptoId},${fiatAmount},${fiatAmountBaseUSDB}`);
    }
</script>

<!-- h-dvh and overflow-hidden prevent the white scroll bug -->
<main class="relative flex h-dvh w-full flex-col overflow-hidden bg-transparent">
    
    <!-- Backgrounds fixed to inset-0 so they never scroll -->
    <div
        style="background-image:url(pattern.png);"
        class="absolute inset-0 z-[-2] bg-neutral-500 opacity-80"
    ></div>
    <div
        style="background-image: linear-gradient(to bottom, rgba(141, 141, 141, 0.45), rgba(0, 0, 0, 0.91));"
        class="absolute inset-0 z-[-1]"
    ></div>

    <div class="relative flex h-full w-full flex-col p-4 gap-4">
        
        <!-- NEW: Top Header / Back Button -->
        <div class="flex w-full items-center justify-start pt-2">
            <button
                onclick={() => goto("/")}
                class="flex items-center gap-2 text-white/80 transition-colors hover:text-white"
            >
                <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2">
                    <path stroke-linecap="round" stroke-linejoin="round" d="M15 19l-7-7 7-7" />
                </svg>
                <span class="text-lg font-bold drop-shadow-md">Back</span>
            </button>
        </div>

        <!-- Top Section: Inputs -->
        <section class="flex flex-col gap-3">
            <!-- Fiat Currency Input -->
            <div class="rounded-md bg-white p-2 shadow">
                <select
                    bind:value={selectedCurrency}
                    class="block w-full border-0 bg-gray-50 text-gray-500 text-md ring-0 outline-none"
                >
                    {#each currencyList as currency}
                        <option value={currency}>{currency.toUpperCase()}</option>
                    {/each}
                </select>
                <div
                    style="font-family:DSEG7"
                    class="flex w-full items-center px-4 pt-2 text-7xl"
                >
                    {fiatAmount || "0"}
                </div>
            </div>

            <!-- Crypto Display -->
            <div class="rounded-md bg-white p-2 shadow">
                <div class="flex h-10 items-center indent-2 text-gray-500 text-md">
                    Estimated {cryptoSymbol} (<span class="font-mono">{exchangeRateHint}</span>)
                </div>
                <div
                    style="font-family:DSEG7"
                    class="w-full px-4 pt-2 text-7xl text-gray-500"
                >
                    {cryptoAmount.toFixed(isStablecoin ? 2 : 8)}
                </div>
            </div>
        </section>

        <!-- Numeric Keypad: flex-1 allows it to take remaining vertical space automatically -->
        <section class="grid flex-1 grid-cols-3 grid-rows-4 gap-3 rounded-xl bg-neutral-700 p-4 shadow-lg">
            {#each [1, 2, 3, 4, 5, 6, 7, 8, 9, ".", 0] as key}
                <button 
                    class="group relative flex w-full items-center justify-center rounded-full bg-neutral-900 shadow-[0_4px_0_#171717] transition-all active:translate-y-[4px] active:shadow-none"
                    onclick={() => handleKeypadInput(key)}
                >
                    <div class="flex h-full w-full items-center justify-center rounded-full bg-gradient-to-b from-neutral-200 to-neutral-400 text-4xl font-bold text-neutral-800 shadow-inner group-active:bg-neutral-300">
                        {key}
                    </div>
                </button>
            {/each}
            
            <button 
                class="group relative flex w-full items-center justify-center rounded-full bg-neutral-900 shadow-[0_4px_0_#171717] transition-all active:translate-y-[4px] active:shadow-none"
                onclick={() => (fiatAmount = "")}
            >
                <div class="flex h-full w-full items-center justify-center rounded-full bg-gradient-to-b from-red-200 to-red-400 text-4xl font-bold text-red-900 shadow-inner group-active:bg-red-300">
                    C
                </div>
            </button>
        </section>

        <!-- Action Button -->
        <section class="flex flex-col items-center pb-2 pt-2">
            <button
                onclick={handleGenerate}
                disabled={cryptoAmount === 0}
                class="w-full rounded-lg bg-neutral-800 px-3 py-4 text-xl font-bold text-white transition-colors hover:bg-neutral-700 disabled:opacity-40 disabled:cursor-not-allowed"
            >
                Generate {cryptoSymbol} Invoice
            </button>
        </section>
    </div>
</main>