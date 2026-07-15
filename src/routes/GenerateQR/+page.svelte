<script>
    import "../../app.css";
    import { goto } from "$app/navigation";
    import Modal from "./Modal.svelte";
    import { onMount, onDestroy } from "svelte";
    import QRCode from "qrcode-generator";

    // Lifecycle & Process Management
    let isComponentMounted = true;
    let qrInterval; 
    let activePollAborter = new AbortController();

    // Core Data
    let cryptoId = $state("btc"); // 'btc', 'usdc', 'usdt'
    let amountFiat = $state(0.0);
    let fiatAmountBaseUSDB = $state(0);
    let fiatSymbol = $state("USD");
    
    // UI State
    let isGenerating = $state(true);
    let generationError = $state("");
    let isQRExpired = $state(false);
    let isModalOpen = $state(false);
    
    // Payment State
    let status = $state("pending"); // "pending", "success", "failed"
    let qrStatus = $state(""); 
    let qrMessage = $state("");
    let paymentReference = $state("");
    
    // QR / Invoice Data
    let qrCodeDataUrl = $state();
    let invoiceData = $state(); 
    let sats = $state("");
    let cryptoAmountDisplay = $state("");

    // Progress Bar
    const duration = 5 * 60000; // 5 mins
    const intervalTime = 100;
    const decrement = 100 / (duration / intervalTime);
    let progress = $state(100);

	// For partial payments
	let amountReceived = $state(0);
	let amountRemaining = $state(0);

	let networkName = $derived(cryptoId === "btc" ? "Lightning" : (cryptoId === "usdc" ? "Solana" : "TRON"));
	let tokenName = $derived(cryptoId === "btc" ? "Bitcoin" : cryptoId.toUpperCase());

    onMount(() => {
        let current = localStorage.getItem("Currency");
        if (current) fiatSymbol = current.toUpperCase();

        // Parse hash: #btc,100,1000000
        const hashParts = location.hash.replace("#", "").split(",");
        if (hashParts.length === 3) {
            cryptoId = hashParts[0];
            amountFiat = parseFloat(hashParts[1]);
            fiatAmountBaseUSDB = parseInt(hashParts[2]);
        }

        startQrProcess();
    });

    onDestroy(() => {
        isComponentMounted = false;
        activePollAborter.abort(); // Cancel any pending fetch requests instantly
        if (qrInterval) clearInterval(qrInterval);
    });

    function navigateHome() {
        goto(`Exchange#${cryptoId}`);
    }

    async function startQrProcess() {
        // Reset states
        isGenerating = true;
        generationError = "";
        isQRExpired = false;
        progress = 100;
        status = "pending";
        qrStatus = "";
        qrMessage = "";
        if (qrInterval) clearInterval(qrInterval);

        // 15-second timeout to prevent infinite loading
        const timeoutPromise = new Promise((_, reject) => 
            setTimeout(() => reject(new Error("Connection timed out. Please try again.")), 15000)
        );

        try {
            if (cryptoId === "btc") {
                await Promise.race([generateBtcQr(), timeoutPromise]);
            } else {
                await Promise.race([generateStablecoinQr(cryptoId), timeoutPromise]);
            }
            
            // Start expiration visual countdown
            qrInterval = setInterval(() => {
                if (progress > 0) {
                    progress = Math.max(progress - decrement, 0);
                } else {
                    clearInterval(qrInterval);
                    isQRExpired = true;
                }
            }, intervalTime);

        } catch (error) {
            generationError = error.message || "Failed to generate QR code";
            isGenerating = false;
        }
    }

    // --- BTC LIGHTNING LOGIC ---
    async function generateBtcQr() {
        const reqObj = {
            invoiceRequest: {
                amountUSDCents: Math.round(fiatAmountBaseUSDB / 10_000),
                publicKey: localStorage.getItem("public_key"),
            }
        };

        const response = await fetch("https://faas-sgp1-18bc02ac.doserverless.co/api/v1/web/fn-a6380fe5-7756-40c6-81ae-70c49e8d07f9/tejoryinvoices/invoicerequest", {
            method: "POST",
            mode: "cors",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(reqObj),
        });

        const resObjRaw = await response.text();
        const obj = JSON.parse(resObjRaw);

        if (obj.status !== "ok") {
            throw new Error(obj.err_msg || "Failed to create Lightning invoice");
        }

        invoiceData = obj.invoice;
        sats = obj.amountSats;
        cryptoAmountDisplay = (parseInt(sats) / 100000000).toFixed(8);

        // Render QR
        const qr = QRCode(0, "L");
        qr.addData(invoiceData);
        qr.make();
        qrCodeDataUrl = qr.createDataURL(12, 0);
        isGenerating = false;

        // Fire & Forget Polling
        pollBtcPayment(obj.invoiceId, obj.txId);
    }

    async function pollBtcPayment(invoiceId, tempTxId) {
        const publicKey = localStorage.getItem("public_key");
        const terminalName = localStorage.getItem("terminal_name");
        let networkErrorCount = 0;

        const reqObj = {
            invoiceStatus: {
                publicKey,
                invoiceId,
                txId: tempTxId, 
                terminalId: terminalName,
            }
        };

        while (!isQRExpired && status === "pending" && isComponentMounted) {
            try {
                const response = await fetch("https://faas-sgp1-18bc02ac.doserverless.co/api/v1/web/fn-a6380fe5-7756-40c6-81ae-70c49e8d07f9/tejoryinvoices/invoicerequest", {
                    method: "POST",
                    mode: "cors",
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify(reqObj),
                    signal: activePollAborter.signal
                });

                if (!isComponentMounted) return; // Exit immediately if user navigated away during fetch
                networkErrorCount = 0; // Reset network faults on successful reach

                const obj = await response.json();

                if (obj.status === "error" && response.status === 402) {
                    // Still pending, wait 2 seconds before next poll
                    await new Promise(resolve => setTimeout(resolve, 2000));
                    if (!isComponentMounted) return; 

                } else if (obj.status === "error") {
                    // Fatal Terminal Error
                    qrStatus = "failed";
                    qrMessage = obj.err_msg || "Payment failed";
                    status = "failed";
                    isModalOpen = true;
                } else {
                    // Success!
                    paymentReference = obj.txId ? obj.txId.substring(obj.txId.length - 10) : "N/A";
                    qrStatus = "success";
                    qrMessage = "Your payment has been received!";
                    status = "success";
                    isModalOpen = true;
                }
                
            } catch (error) {
                if (!isComponentMounted || error.name === 'AbortError') return;
                
                networkErrorCount++;
                if (networkErrorCount > 5) {
                    // Max retries hit, likely lost Wi-Fi completely
                    qrStatus = "failed";
                    qrMessage = "Lost connection to payment server.";
                    status = "failed";
                    isModalOpen = true;
                    break;
                }
                await new Promise(resolve => setTimeout(resolve, 3000));
            }
        }
    }

    // --- STABLECOIN LOGIC (PLACEHOLDERS) ---
    async function generateStablecoinQr(coinId) {
        try {
            // Determine the network based on the coinId
            const network = coinId === "usdc" ? "Solana" : "TRON";
            
            // 1. Fetch address/payload using your new helper function
            const addressData = await getStablecoinAddress(coinId.toUpperCase(), network);
            
            if (!addressData || !addressData.address) {
                throw new Error(`Invalid address received for ${coinId.toUpperCase()}`);
            }

            // 2. Format cryptoAmountDisplay (Stablecoins are pegged 1:1 with USD)
            // We use the amountFiat parsed from the URL hash earlier
            cryptoAmountDisplay = amountFiat.toFixed(2);

            // 3. Construct the exact string for the QR Code
            if (coinId === "usdc") {
                // Solana Pay URI format: solana:<address>?amount=<amount>&spl-token=<mint_address>
                // The official SPL-token mint address for USDC on Solana mainnet is EPj...TDt1v
                const usdcMint = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v";
                invoiceData = `solana:${addressData.address}?amount=${cryptoAmountDisplay}&spl-token=${usdcMint}`;
				invoiceData = addressData.address;
            } else if (coinId === "usdt") {
                // TRON generally relies on just scanning the raw base58 address for TRC20 token transfers
                // as not all wallets support a unified URI scheme like Solana Pay.
                invoiceData = addressData.address; 
            }

            // Render QR Code
            const qr = QRCode(0, "L");
            qr.addData(invoiceData);
            qr.make();
            qrCodeDataUrl = qr.createDataURL(12, 0); // Scale 12, margin 0
            
            // Turn off the loading spinner
            isGenerating = false;

            // 4. Fire the polling function using the returned address ID
            pollStablecoinPayment(coinId, addressData.address);
            
        } catch (error) {
            console.error(`Error generating ${coinId.toUpperCase()} QR:`, error);
            generationError = error.message || `Failed to setup ${coinId.toUpperCase()} payment`;
            isGenerating = false;
        }
    }

    async function pollStablecoinPayment(coinId, address) {
        const publicKey = localStorage.getItem("public_key");
        const terminalName = localStorage.getItem("terminal_name");
        let networkErrorCount = 0;
        
        // Define network and formatting based on your Go code requirements
        const network = coinId === "usdc" ? "solana" : "tron";
        const expectedAmountStr = amountFiat.toFixed(2);

        // Construct the payload mapping to your Go map[string]any
        const reqObj = {
            action: "get-stablecoin-balance", // Adjust this if your DO function uses a different action router
            public_key: publicKey,
            terminal_name: terminalName,
            value_type: coinId.toUpperCase(),
            address: address, 
            amount: expectedAmountStr,
            address_type: "internal",
            transfer_types: network
        };

        while (!isQRExpired && status === "pending" && isComponentMounted) {
            try {
                const response = await fetch("https://faas-sgp1-18bc02ac.doserverless.co/api/v1/web/fn-a6380fe5-7756-40c6-81ae-70c49e8d07f9/get_usdb/sf_braleservices", {
                    method: "POST",
                    mode: "cors",
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify(reqObj),
                    signal: activePollAborter.signal
                });

                if (!isComponentMounted) return; // Exit if user navigated away
                networkErrorCount = 0; // Reset network faults

                const httpStatus = response.status;
                const obj = await response.json();

                // 202 Accepted: Still processing (0 balance OR Partial Payment)
                if (httpStatus === 202) {
                    
                    if (obj.status === "pending" && obj.amount_remaining) {
                        // TODO for later: UI Update for Partial Payments
                        // You can update the QR code here with a new expected amount, 
                        // or show a warning to the user like:
                        console.log(`Partial Payment Detected! Received: ${obj.amount_received}, Remaining: ${obj.amount_remaining}`);
                    } else {
                        // Standard waiting (0 balance)
                        console.log("Waiting for payment on-chain...");
                    }

                    // On-chain txs are slower than Lightning. Wait 3-4 seconds to save DO bandwidth.
                    await new Promise(resolve => setTimeout(resolve, 3500));
                    if (!isComponentMounted) return;

                } 
                // 200 OK: Transfer completed and routed to Spark successfully
                else if (httpStatus === 200 && obj.status === "ok") {
                    // Assuming the Go 'tx' object has an 'id' or 'hash' field
                    const txId = obj.txId ?? "N/A";
                    paymentReference = txId !== "N/A" ? txId.substring(txId.length - 10) : "N/A";
                    
                    qrStatus = "success";
                    qrMessage = "Your payment has been received!";
                    status = "success";
                    isModalOpen = true;
                } 
                // 400, 404, 409, 500: Fatal Errors (e.g. massive overpayment, unsupported network)
                else {
                    qrStatus = "failed";
                    qrMessage = obj.err_msg || "Payment failed or rejected.";
                    status = "failed";
                    isModalOpen = true;
                }
                
            } catch (error) {
                if (!isComponentMounted || error.name === 'AbortError') return;
                
                networkErrorCount++;
                if (networkErrorCount > 5) {
                    // Max retries hit, likely lost Wi-Fi completely
                    qrStatus = "failed";
                    qrMessage = "Lost connection to payment server.";
                    status = "failed";
                    isModalOpen = true;
                    break;
                }
                // Back-off slightly longer on network errors
                await new Promise(resolve => setTimeout(resolve, 4000));
            }
        }
    }

	async function getStablecoinAddress(cryptoSymbol, network) {
		const cacheKey = "TerminalAddresses";
		
		// 1. Check if we already have it in localStorage
		let cachedData = JSON.parse(localStorage.getItem(cacheKey) || "{}");
		
		if (cachedData[cryptoSymbol] && cachedData[cryptoSymbol][network]) {
			console.log(`Loaded ${cryptoSymbol} address from cache!`);
			return cachedData[cryptoSymbol][network];
		}

		// 2. If not, fetch it from your serverless function
		console.log(`Fetching ${cryptoSymbol} address from server...`);
		const publicKey = localStorage.getItem("public_key");
		const terminalName = localStorage.getItem("terminal_name");
		
		// Replace this URL with dynamic params based on cryptoSymbol and network
		const url = `https://faas-sgp1-18bc02ac.doserverless.co/api/v1/web/fn-a6380fe5-7756-40c6-81ae-70c49e8d07f9/get_usdb/sf_braleservices?action=get-stablecoin-address&public_key=${publicKey}&address_type=internal&transfer_types=${network.toLowerCase()}&terminal_name=${encodeURIComponent(terminalName)}`;

		const response = await fetch(url);
		const data = await response.json();

		if (data.status === "ok") {
			// 3. Update the cache object
			if (!cachedData[cryptoSymbol]) cachedData[cryptoSymbol] = {};
			
			cachedData[cryptoSymbol][network] = {
				address: data.address, 
				id: data.addressId 
			};

			// 4. Save it back to localStorage
			localStorage.setItem(cacheKey, JSON.stringify(cachedData));
			
			return cachedData[cryptoSymbol][network];
		} else {
			throw new Error("Failed to fetch stablecoin address");
		}
	}
</script>

<main class="relative flex h-dvh w-dvw flex-col items-center justify-center bg-transparent">
    <!-- Backgrounds -->
    <div style="background-image:url(pattern.png);" class="absolute inset-0 z-[-2] bg-neutral-500 opacity-80"></div>
    <div style="background-image: linear-gradient(to bottom, rgba(141, 141, 141, 0.45), rgba(0, 0, 0, 0.91));" class="absolute inset-0 z-[-1]"></div>

    <!-- 1. REMOVED the 'main-content' class to prevent app.css from injecting a solid background -->
    <div class="flex h-full w-full max-w-md flex-col items-center justify-center px-4">
        
        {#if isGenerating}
            <!-- LOADING STATE -->
            <div class="flex flex-col items-center justify-center gap-5 rounded-xl bg-white p-10 text-center shadow-2xl">
                <div class="h-12 w-12 animate-spin rounded-full border-4 border-neutral-300 border-t-neutral-800"></div>
                <p class="text-xl font-bold text-neutral-800">Generating Secure QR Code...</p>
                <p class="text-neutral-500">Contacting payment server</p>
            </div>

        {:else if generationError}
            <!-- FATAL GENERATION ERROR -->
            <div class="flex flex-col items-center justify-center gap-6 rounded-xl bg-white p-8 text-center shadow-2xl">
                <div class="flex h-16 w-16 items-center justify-center rounded-full bg-red-100">
                    <svg class="h-8 w-8 text-red-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
                    </svg>
                </div>
                <div>
                    <h2 class="text-2xl font-bold text-neutral-800">Generation Failed</h2>
                    <p class="mt-2 text-neutral-600">{generationError}</p>
                </div>
                <div class="mt-2 flex w-full gap-3">
                    <button onclick={navigateHome} class="flex-1 rounded-lg bg-neutral-200 py-3 font-bold text-neutral-800 hover:bg-neutral-300">Back</button>
                    <button onclick={startQrProcess} class="flex-1 rounded-lg bg-neutral-800 py-3 font-bold text-white hover:bg-neutral-700">Retry</button>
                </div>
            </div>

        {:else}
            <!-- ACTIVE QR OR EXPIRED STATE -->
            <!-- 2. CHANGED to bg-white/10 (frosted glass) and increased blur -->
            <div class="w-full overflow-hidden rounded-2xl border border-white/20 bg-white/10 shadow-2xl">
                
                <!-- Notice Header -->
                <div class="bg-black/20 p-5 pb-3">
                    <h1 class="font-bold text-white">Notice:</h1>
                    <p class="text-white">This invoice will expire in <span class="font-bold text-red-400">5 minutes.</span></p>
                </div>

                <!-- Tailwind Progress Bar -->
                <div class="h-2.5 w-full bg-black/40 shadow-[0_3px_5px_rgba(0,0,0,0.5)_inset]">
                    <div 
                        class="h-full transition-all duration-100 ease-linear" 
                        style="width: {progress}%; background-color:hsl({progress}, 100%, 50%);"
                    ></div>
                </div>

                <!-- QR Payload Area -->
               <div class="flex flex-col items-center justify-center p-6 sm:p-8">
					{#if !isQRExpired}
						<div class="flex w-full flex-col items-center">
							
							<!-- PARTIAL PAYMENT WARNING -->
							{#if amountReceived > 0}
								<div class="mb-4 w-full animate-pulse rounded-lg border-2 border-orange-400 bg-orange-100 p-3 text-center shadow-lg">
									<p class="font-bold text-orange-800">Partial Payment Detected</p>
									<p class="text-sm font-medium text-orange-700">Received: {fiatSymbol} {amountReceived.toFixed(2)}</p>
									<p class="mt-1 text-lg font-bold text-red-600">Still Owed: {fiatSymbol} {amountRemaining.toFixed(2)}</p>
								</div>
							{/if}

							<!-- AMOUNT DISPLAY -->
							<p class="mb-3 w-full rounded bg-white py-2 text-center text-3xl font-bold text-neutral-900 shadow-md transition-all">
								{fiatSymbol} {amountRemaining > 0 ? amountRemaining.toFixed(2) : amountFiat.toFixed(2)}
							</p>
							
							<!-- NETWORK INDICATOR -->
							<div class="mb-3 w-full rounded-md border border-white/20 bg-neutral-900 py-2 text-center text-sm font-bold text-white shadow-md">
								Send {tokenName} via <span class="text-blue-400">{networkName}</span> Network
							</div>
							
							<!-- QR CODE -->
							<img 
								src={qrCodeDataUrl} 
								alt="Payment QR Code" 
								class="w-full max-w-[350px] aspect-square rounded-lg border-[12px] border-white shadow-[0_10px_25px_rgba(0,0,0,0.8)] transition-all" 
							/>
							
							<!-- Invoice String -->
							<div class="mt-6 w-full rounded bg-black/40 p-2">
								<p class="w-full break-all font-mono text-sm text-white/80 line-clamp-2">
									{invoiceData}
								</p>
							</div>

							<button onclick={navigateHome} class="mt-6 w-full rounded-lg bg-red-600 py-4 text-lg font-bold text-white shadow-lg transition-colors hover:bg-red-700">
								Cancel Invoice
							</button>
						</div>
					{:else}
						<!-- Expired State -->
						<div class="flex flex-col items-center justify-center gap-6 py-10">
							<div class="flex h-24 w-24 items-center justify-center rounded-full bg-black/40 shadow-inner">
								<svg class="h-16 w-16 text-red-500" fill="none" viewBox="0 0 24 24" stroke="currentColor">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
								</svg>
							</div>
							<p class="text-2xl font-bold text-white shadow-black drop-shadow-md">QR Code Expired</p>
							<button onclick={startQrProcess} class="w-full rounded-lg bg-green-500 px-8 py-4 text-lg font-bold text-neutral-900 shadow-lg transition-colors hover:bg-green-400">
								Regenerate new QR code
							</button>
						</div>
					{/if}
				</div>
            </div>
        {/if}

        <Modal
            isOpen={isModalOpen}
            message={qrMessage}
            status={qrStatus}
            amountCrypto={cryptoAmountDisplay}
            cryptoSymbol={cryptoId === "btc" ? "BTC" : cryptoId.toUpperCase()}
            {amountFiat}
            {fiatSymbol}
            {paymentReference}
        />
    </div>
</main>