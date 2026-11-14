<script>
    import { goto } from "$app/navigation";
    import { onMount } from "svelte";
    import { BleClient } from '@capacitor-community/bluetooth-le';
    import QRCode from "qrcode-generator";
    
    let {
        isOpen = $bindable(),
        message,
        status,
        amountCrypto,
        cryptoSymbol,
        amountFiat,
        fiatSymbol,
        paymentReference,
    } = $props();

    let now = new Date();
    let dateOptions = { weekday: "short", year: "numeric", month: "short", day: "numeric" };
    let timeOptions = { hour12: true, hour: "2-digit", minute: "2-digit" };
    
    let terminalName = $state("");
    let currency_type = $state("");
    
    // BLE State
    let connectedDeviceId = null;
    let statusMessage = $state('Ready to connect to a printer.');

    // QR State
    let showQR = $state(false);
    let receiptQrUrl = $state("");

    // Dynamic Receipt Data based on Crypto Type
    let networkName = $derived(cryptoSymbol === 'BTC' ? 'Lightning' : (cryptoSymbol === 'USDC' ? 'Solana' : 'TRON'));
    let cryptoName = $derived(cryptoSymbol === 'BTC' ? 'Bitcoin' : cryptoSymbol);
    let formattedCrypto = $derived(Number(amountCrypto).toFixed(cryptoSymbol === 'BTC' ? 8 : 2));

    onMount(() => {
        terminalName = localStorage.getItem("terminal_name") || "Terminal-01";
        currency_type = localStorage.getItem("Currency")?.toUpperCase() || fiatSymbol;
    });

    function closeModal() {
        isOpen = false;
        goto("/");
    }

    // --- QR RECEIPT LOGIC ---
    function toggleQRReceipt() {
        if (!showQR && !receiptQrUrl) {
            const qr = QRCode(0, "L");
            qr.addData(`invoice-${paymentReference}`);
            qr.make();
            receiptQrUrl = qr.createDataURL(8, 0); // Scale 8, margin 0
        }
        showQR = !showQR;
    }

    // --- BLUETOOTH LOGIC ---
    async function connectToPrinter() {
        try {
            statusMessage = 'Initializing Bluetooth...';
            await BleClient.initialize();

            // 1. Try to connect to a previously saved printer first
            const savedPrinterId = localStorage.getItem("saved_printer_id");
            if (savedPrinterId) {
                try {
                    statusMessage = 'Connecting to saved printer...';
                    await BleClient.connect(savedPrinterId);
                    connectedDeviceId = savedPrinterId;
                    statusMessage = 'Connected to saved printer!';
                    return true;
                } catch (e) {
                    console.log("Failed to connect to saved printer. Requesting new device...", e);
                    localStorage.removeItem("saved_printer_id"); // Clear bad ID
                }
            }

            // 2. If no saved printer or connection failed, request manually
            statusMessage = 'Please select a printer...';
            const device = await BleClient.requestDevice({ services: [] });

            statusMessage = `Connecting to ${device.name}...`;
            await BleClient.connect(device.deviceId);
            
            connectedDeviceId = device.deviceId;
            localStorage.setItem("saved_printer_id", connectedDeviceId); // Save for next time
            statusMessage = `Connected to ${device.name}!`;
            return true;

        } catch (error) {
            statusMessage = `Error: ${error.message}`;
            console.error('Bluetooth Connection Error', error);
            alert(error);
            return false;
        }
    }

    async function printTestReceipt() {
        if (!connectedDeviceId) {
            statusMessage = 'No printer connected.';
            return;
        }

        try {
            statusMessage = 'Sending data to printer...';
            const PRINTER_SERVICE = '000018f0-0000-1000-8000-00805f9b34fb'; 
            const PRINTER_CHARACTERISTIC = '00002af1-0000-1000-8000-00805f9b34fb'; 

            const encoder = new TextEncoder();
            const commands = [
                '=============================\n',
                '       Payment Receipt\n',
                '=============================\n',
                '\n\n',
                `Payment ${status === "success" ? "Successful" : "Failed"}\n`,
                `Date: ${now.toLocaleDateString("en-US", dateOptions)}\n`,
                `Time: ${now.toLocaleTimeString("en-US", timeOptions)}\n`,
                `Reference: ${paymentReference}\n`,
                `Terminal: ${terminalName}\n`,
                `=============================\n`,
                '\n\n',
                `Currency: ${cryptoName}\n`,
                `Network: ${networkName}\n`,
                `Amount (${cryptoSymbol}): ${formattedCrypto}\n`,
                `Amount (${currency_type}): ${Number(amountFiat).toFixed(2)}\n`,
                '\x1D\x56\x41\x00' // Paper cut command
            ].join('');

            const dataToSend = encoder.encode(commands);

            await BleClient.write(
                connectedDeviceId,
                PRINTER_SERVICE,
                PRINTER_CHARACTERISTIC,
                dataToSend
            );
            
            statusMessage = 'Print command sent successfully!';
        } catch (error) {
            statusMessage = `Error: ${error.message}`;
            console.error('Bluetooth Write Error', error);
            alert("Printer write failed: " + error.message);
        }
    }

    async function disconnect() {
        if (connectedDeviceId) {
            await BleClient.disconnect(connectedDeviceId);
            connectedDeviceId = null;
            statusMessage = 'Disconnected.';
        }
    }

    async function printReceipt(){
        const connected = await connectToPrinter();
        if (connected) {
            await printTestReceipt();
            await disconnect();
        }
    }
</script>

{#if isOpen}
    <div class="fixed inset-0 z-50 flex items-center justify-center bg-black/60 p-4 backdrop-blur-sm">
        <div class="flex w-full max-w-sm flex-col items-center rounded-2xl bg-white p-6 text-center shadow-2xl">
            
            <h2 class="text-2xl font-bold {status === 'success' ? 'text-green-600' : 'text-red-600'}">
                {status === "success" ? "Payment Successful!" : "Payment Failed"}
            </h2>
            <p class="mt-2 text-neutral-600">{message}</p>
            
            <!-- Receipt Container -->
            <div class="my-6 w-full overflow-hidden border-y-2 border-dashed border-neutral-300 py-6 text-left font-mono text-sm text-neutral-800">
                
                {#if showQR}
                    <!-- QR Code View -->
                    <div class="flex flex-col items-center justify-center gap-4 fade-in">
                        <p class="text-center font-bold">Scan for Digital Receipt</p>
                        <img src={receiptQrUrl} alt="Receipt QR" class="rounded-lg shadow-md" />
                        <p class="text-xs text-neutral-500">Ref: {paymentReference}</p>
                    </div>
                {:else}
                    <!-- Paper Receipt View (Animated) -->
                    <div class="receipt-animate relative rounded border border-neutral-200 bg-neutral-50 p-4 shadow-[3px_7px_9px_2px_rgba(0,0,0,0.1)]">
                        =============================<br />
                        &nbsp;&nbsp;Tejory Payment Receipt<br />
                        =============================<br /><br />
                        
                        {status === "success" ? "Payment Successful!" : "Payment Failed"}<br />
                        Date: {now.toLocaleDateString("en-US", dateOptions)}<br />
                        Time: {now.toLocaleTimeString("en-US", timeOptions)}<br />
                        Ref: {paymentReference}<br />
                        Term: {terminalName}<br />
                        =============================<br /><br />
                        
                        Currency: {cryptoName}<br />
                        Network: {networkName}<br />
                        Amt ({cryptoSymbol}): {formattedCrypto}<br />
                        Amt ({currency_type}): {Number(amountFiat).toFixed(2)}<br />
                        <br /><br />
                    </div>
                {/if}
            </div>

            <!-- Action Buttons -->
            <div class="flex w-full flex-col gap-3">
                <div class="flex w-full gap-3">
                    <button 
                        onclick={toggleQRReceipt} 
                        class="flex-1 rounded-lg bg-neutral-200 py-3 font-bold text-neutral-800 transition-colors hover:bg-neutral-300"
                    >
                        {showQR ? "Show Paper" : "QR Receipt"}
                    </button>
                    
                    {#if !showQR}
                        <button 
                            onclick={printReceipt} 
                            class="flex-1 rounded-lg bg-green-500 py-3 font-bold text-white transition-colors hover:bg-green-600"
                        >
                            Print
                        </button>
                    {/if}
                </div>
                
                <button 
                    onclick={closeModal} 
                    class="w-full rounded-lg bg-neutral-800 py-3 font-bold text-white transition-colors hover:bg-neutral-700"
                >
                    Start New Transaction
                </button>
            </div>
            
            <p class="mt-4 text-xs text-neutral-400">{statusMessage}</p>
        </div>
    </div>
{/if}

<style>
    .receipt-animate {
        animation: print 3s ease-out;
    }
    
    .fade-in {
        animation: fadeIn 0.3s ease-in;
    }

    @keyframes print {
        0% { transform: translateY(90%); opacity: 0; }
        20% { transform: translateY(70%); opacity: 1; }
        40% { transform: translateY(50%); }
        60% { transform: translateY(30%); }
        80% { transform: translateY(10%); }
        100% { transform: translateY(0%); }
    }
    
    @keyframes fadeIn {
        from { opacity: 0; transform: scale(0.95); }
        to { opacity: 1; transform: scale(1); }
    }
</style>