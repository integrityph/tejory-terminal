<script>
	import { onMount } from "svelte";
	import QRCode from "qrcode-generator";

	let initialized = $state(false);
	let loading = $state(true);
	let tempToken = null;
	let qrCodeDataUrl = $state();
	let remoteToken = $state('');

  // Function to handle the token submission
  function handleLinkWithToken() {
    if (!remoteToken.trim()) return; // Simple validation
    
		let tokenParts = remoteToken.split(":");

		if (tokenParts.length != 2){
			console.log("tokenParts.length is not 2", tokenParts.length);
			return;
		}

    	localStorage.setItem("public_key", tokenParts[0]);
		localStorage.setItem("terminal_name", tokenParts[1]);
		initialized = true;
		setTimeout(() => {
			loading = false;
		}, 1000);
    
		// Clear input after submission
		remoteToken = '';
  	}

	function buf2hex(buffer) {
		// buffer is an ArrayBuffer
		return [...new Uint8Array(buffer)]
			.map((x) => x.toString(16).padStart(2, "0"))
			.join("");
	}

	// Simulate a loading delay (remove this in actual use)
	onMount(() => {
		// check the URL for initialization code
		// 46 is 33 bytes public key + sep + terminal name (1 letter) * 0.75 (for base64)
		// check the URL for initialization code
		if (location.hash.length >= 46) {
			try {
				let b64Url = location.hash.substring(1);
				let b64 = b64Url.replace(/-/g, '+').replace(/_/g, '/');
				
				// Pad with '=' if the length isn't a multiple of 4
				while (b64.length % 4 !== 0) {
					b64 += '=';
				}

				let binaryString = atob(b64);
				let bytes = new Uint8Array(binaryString.length);
				for (let i = 0; i < binaryString.length; i++) {
					bytes[i] = binaryString.charCodeAt(i);
				}
				let linkingString = new TextDecoder().decode(bytes);

				let parts = linkingString.split("\x1f");
				if (parts.length >= 2) {
					localStorage.setItem("public_key", parts[0]);
					localStorage.setItem("terminal_name", parts[1]);
					if (parts.length > 2) localStorage.setItem("business_name", parts[2]);
					if (parts.length > 3) localStorage.setItem("merchant_email", parts[3]);
					
					// Clean up: Remove the hash from the URL so it isn't visible/re-processed on refresh
					window.history.replaceState(null, null, window.location.pathname);
				}
			} catch (e) {
				console.error("Failed to parse terminal setup link:", e);
			}
		}

		let pubkey = localStorage.getItem("public_key");
		let terminal_name = localStorage.getItem("terminal_name");

		if (pubkey != null && terminal_name != null) {
			initialized = true;
			setTimeout(() => {
				loading = false;
			}, 3000);
		}
	});
</script>

{#if loading}
	<div
		class="loader item-center absolute z-10 flex h-dvh w-full items-center justify-center bg-white"
		style="background-image:url(pattern.png);"
	>
		<div
			style="opacity:0.9"
			class="loader item-center absolute z-10 flex h-dvh w-full items-center justify-center bg-neutral-500"
		></div>
		<div
			style="background-image: linear-gradient(to bottom, rgba(255, 0, 0, 0), rgb(0 0 0));"
			class="loader item-center absolute z-10 flex h-dvh w-full items-center justify-center"
		>
			{#if initialized}
				<p class="blinks text-[70px] font-bold">TEJ</p>
				<div class="loads">
					<div class="spinner">
						<img src="loader.png" alt="" />
					</div>
				</div>

				<p class="blinks text-[70px] font-bold">RY</p>
			{:else}
				<div style="display: block" class="px-3">
					<h2 class="rounded bg-white p-2 text-center font-bold">
						Activate Your Terminal
					</h2>
					<div>
						{#if qrCodeDataUrl}
							<img
								src={qrCodeDataUrl}
								alt="QR Code"
								class="w-full rounded-md border-15 border-white shadow-2xl shadow-black"
							/>
						{/if}
					</div>
					<div class="my-4 rounded bg-[#ffffffa9] p-4 text-gray-800 shadow">
						<p class="text-sm font-semibold mb-2">To link via QR Code:</p>
						<ul class="list-inside list-disc space-y-1">
							<li>Go to "Settings" in Tejory Wallet</li>
							<li>Click "Link Terminal"</li>
							<li>Scan the QR code to activate this terminal</li>
						</ul>

						<div class="relative flex items-center my-2">
							<div class="flex-grow border-t border-gray-300"></div>
							<span class="flex-shrink mx-4 text-sm text-gray-500">OR</span>
							<div class="flex-grow border-t border-gray-300"></div>
						</div>

						<div>
							<label for="token-input" class="block text-sm font-semibold mb-2">
								Paste your initialization token here:
							</label>
							<div class="flex space-x-2">
								<input
									id="token-input"
									type="text"
									bind:value={remoteToken}
									placeholder="e.g., 021234..."
									class="flex-grow w-full rounded border border-gray-300 bg-white/80 p-2 font-mono focus:ring-blue-500 focus:border-blue-500"
								/>
								<button
									onclick={handleLinkWithToken}
									disabled={!remoteToken.trim()}
									class="px-4 py-2 font-semibold text-white bg-blue-600 rounded shadow hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed"
								>
									Link
								</button>
							</div>
						</div>
					</div>

					
				</div>
			{/if}
		</div>
	</div>
{/if}

<style>
	@font-face {
		font-family: "Nexa";
		font-style: normal;
		font-weight: 300;
		src:
			local("Nexa"),
			url("https://fonts.cdnfonts.com/s/16221/NexaLight.woff") format("woff");
	}
	@font-face {
		font-family: "Nexa";
		font-style: normal;
		font-weight: 700;
		src:
			local("Nexa"),
			url("https://fonts.cdnfonts.com/s/16221/NexaBold.woff") format("woff");
	}

	* {
		font-family: "Nexa";
	}
	.loads {
		transform: translateY(-10px);
	}

	.spinner {
		background-image: url("/loader.png");
		width: 65px;
		height: 65px;
		animation: spin 8s linear infinite;
	}
	.blinks {
		animation: blink 2s;
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}
	@keyframes blink {
		0% {
			opacity: 0;
		}
		100% {
			opacity: 1;
		}
	}
</style>
