<template>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
  </head>
  <div :class="{ 'dark-mode': isDarkMode }" class="landing-container">
    <header class="landing-header">
      <img src="@/assets/paytaca.jpg" alt="Paytaca Logo" class="site-logo" />
      <a href="https://www.paytaca.com/#home" target="_blank" class="site-title">Paytaca.com</a>
      
      <!-- Dark Mode Toggle Button -->
      <button class="toggle-button" @click="toggleDarkMode">
  <span v-if="isDarkMode">🌙</span>
  <span v-else>🌞</span>
</button>

    </header>

    <div class="header-padding">
      <img src="@/assets/paper5.png" class="paper5-logo" />
      <h1 class="header-padding-text">Bitcoin Cash (BCH) Paper Wallet</h1>
    </div>

    <p class="wallet-description">Wallet Generated</p>

    <div class="wallet-container">
      <!-- Step One - Choose Your Design -->
      <div class="step-container">
        <div class="step-label1" @click="toggleDropdown">
          <img src="@/assets/dropdown.png" alt="dropdown" class="dropdown-image" />
          <h1 class="step-text">Step One - Choose Your Design</h1>
        </div>

        <div v-if="dropdownOpen" class="design-grid">
          <div v-for="design in designs" :key="design.id" class="design-preview">
            <img :src="design.image" :alt="'Design ' + design.id" class="design-image" />
            <div class="overlay">
            <h3 class="wallet-name" :style="{ name: design.name}">
              {{ design.name }}
            </h3>
          </div>
            <button class="select-button" @click="proceedToCustomization(design)">
              Select and Continue
            </button>
          </div>
        </div>
      </div>

      <!-- Step Two - Customization -->
      <div v-if="selectedDesign" class="step-container">
        <div class="step-label2" @click="toggleStep(2)">
          <img src="@/assets/dropdown.png" alt="dropdown" class="dropdown-image" />
          <h1 class="step-text">Step Two - Customize</h1>
        </div>

        <div v-if="activeStep === 2" class="customization-section">
          <label class="custom">Custom BCH Amount:</label>
          <select v-model="paymentDetails" @change="updatePublicQRCodes" class="dropdown">
            <option disabled value="">Amount</option>
            <option v-for="amount in [0.0001, 0.001, 0.005, 0.05, 0.1, 0.25, 0.5, 0.75, 1, 5, 10]" 
                    :key="amount" 
                    :value="amount">
              {{ amount }} BCH
            </option>
          </select>

          <label class="address">Addresses to generate:</label>
          
          <div class="loader-wrapper">
            <div v-if="loading" class="spinner"></div>
              <input class="input-bar" type="number" v-model.number="addressCount" @input="generateMultipleKeys" />
             <button class="encryption" @click="toggleAdvanceSettingdropdown">
                  {{ showAdvanceSettingdropdown ? 'Hide Advanced Settings' : 'Advance Settings' }}
            </button>  
            </div>
             
      
             <div class="dropdown-wrapper">
  <div v-if="showAdvanceSettingdropdown" class="dropdown-panel">
  <p><strong>Check the BIP38 option, enter a passphrase, and click "Generate" to create an encrypted wallet</strong></p>

  <!-- BIP38 Checkbox & Link -->

  <div class="advanced-settings-row">
  <!-- BIP38 Checkbox -->
  <input type="checkbox" v-model="encryptOption" id="bip38" />

  <!-- Label + Link -->
  <label for="bip38">
    BIP38 Encrypt? 
    <span class = "tooltip-container">
    <a class="what-is-this" href="#">(What's this?)</a>
    <span class="tooltip-text">
    Selecting this option allows you to encrypt your wallet with a password you choose.
    You will not be able to spend from the wallet without this password. The benefit is
    additional security, but be careful — there is no way to recover your password if you forget it!
  </span>
</span>
  </label>

  <!-- Passphrase -->
  <label for="passphrase" class="passphrase-label">Passphrase:</label>
  <input id="passphrase" type="text" v-model="passphrase" class="passphrase-input" />

  <!-- Generate Button -->
  <button v-if="encryptOption" class="generate-btn">Generate</button>
</div>


  
</div>
</div>


          


          <!-- Paper Wallet Container -->
          <div v-if="customAmount && addressCount" class="paper-wallet-container">
            <div class="paper-wallet">
              <div class="paper-wallet-content">
                <div class="generated-addresses"></div>
              </div>
            </div>
          </div>

          <!-- Design Image Display -->
          <div v-if="generatedWallets.length" class="wallets-container"id="printable-wallet">
            <div v-for="(wallet, index) in generatedWallets" :key="index" class="selected-design-container">
              <h3 class = "wallet-padding" v-if="index > 0"></h3> 
              <div class="selected-design">
                <div class="design-image-container" :style="{ color: wallet.design.textColor }">
                  <img :src="wallet.design.image" alt="Selected Design" class="design-image" />

                  <div class="qr-code-overlay">
                    <div class="bch-amount">
                      <p>{{ wallet.customAmount }} BCH</p> <!-- Show the correct amount per wallet -->
                    </div>

                    <!-- Public Address QR -->
                    <div class="qr-section public-section">
                      <img :src="wallet.qrCodePublic" alt="Public QR Code" class="qr-code public-qr" />
                    </div>
                    <p class="wallet-address">{{ wallet.address }}</p>

                    <!-- Private Key QR -->
                    <div class="qr-section private-section">
                      <img :src="wallet.qrCodePrivate" alt="Private QR Code" class="qr-code private-qr" />
                    </div>
                    <p class="private-key">{{ wallet.wif }}</p>
                  </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

      <!-- Step Three - Print -->
      <div v-if="selectedDesign" class="step-container">
        <div class="step-label3" @click="printWallet(3)">
          <img src="@/assets/dropdown.png" alt="dropdown" class="dropdown-image" />
          <h1 class="step-text">Step Three - Print</h1>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import CryptoJS from 'crypto-js';
import QRCode from "qrcode";
import { Buffer } from "buffer";
import secp256k1 from 'secp256k1';
import bs58 from "bs58";
import cashaddr from "cashaddrjs";
import html2canvas from 'html2canvas';

export default {
  data() {
    return {
      paymentDetails: "",
      privateKeyWIF: "",
      publicKeyHex: "",
      bitcoinCashAddress: "",
      qrCodeDataPublic: "",
      qrCodeDataPrivate: "",
      privateKeyHash: "",
      publicKeyHash: "", 
      dropdownOpen: true,     
      activeStep: null,       
      selectedDesign: null,   
      customAmount: null,
      addressCount: 1,
      generatedWallets: [],
      isLightMode: localStorage.getItem("lightMode") === "true" || localStorage.getItem("lightMode"),
      isDarkMode: localStorage.getItem("darkMode") === "true" && localStorage.getItem("lightMode") !== "true",
      loading: false,
      showAdvanceSettingdropdown: false,
      encryptOption: false,
      passphrase: '',
      designs: [
        { id: 1, image: "src/assets/1.png", textColor: 'black', name: "Paytaca ver.1" },
        { id: 2, image: "src/assets/2.png", textColor: 'black', name: "Paytaca ver.2"},
        { id: 3, image: "src/assets/3.png", textColor: 'black', name: "Ebony Gray Blue" },
        { id: 4, image: "src/assets/4.png", textColor: 'black', name: "Cool Blue" },
        { id: 5, image: "src/assets/5.png", textColor: 'black', name: "Ebony Gray Red" },
        { id: 6, image: "src/assets/6.png", textColor: 'black', name: "Carnation" },
        { id: 7, image: "src/assets/7.png", textColor: 'black', name: "Paytaca Gradient" },
        { id: 8, image: "src/assets/8.png", textColor: 'black', name: "Ebony Gray" },
        { id: 9, image: "src/assets/9.png", textColor: 'black', name: "Gradient Cool Blue" },
        { id: 10, image: "src/assets/10.png", textColor: 'black', name: "Gradient Carnation"},
      ],
    };
  },

  async created() {
  await this.generateNewKeys(); // Generate new keys upon component creation
  document.body.classList.toggle("dark-mode", this.isDarkMode);
  document.body.classList.toggle("light-mode", this.isLightMode);
},

  methods: {
    toggleAdvanceSettingdropdown() {
      this.showAdvanceSettingdropdown = !this.showAdvanceSettingdropdown;
    },


    //Dark Mode Method
    toggleDarkMode() {
  this.isDarkMode = !this.isDarkMode;
  this.isLightMode = !this.isLightMode; // Ensure only one mode is active

  // Save the mode in local storage
  localStorage.setItem("darkMode", this.isDarkMode);
  localStorage.setItem("lightMode", this.isLightMode);

  // Remove both modes first, then apply the correct one
  document.body.classList.remove("light-mode", "dark-mode");
  if (this.isDarkMode) {
    document.body.classList.add("dark-mode");
    document.body.classList.remove("light-mode");
  } else {
    document.body.classList.add("light-mode");
    document.body.classList.remove("light-mode");
  }
},

    // Computes SHA-256 hash for a given data input
    async sha256(data = '', encoding = 'utf8') {
      let buffer;

      if (data instanceof Uint8Array) {
        buffer = data;
      } else if (encoding === 'utf8') {
        buffer = new TextEncoder().encode(data);
      } else if (encoding === 'hex') {
        buffer = this.hexToBin(data);
      } else {
        throw new Error('Unsupported encoding type');
      }

      // Use window.crypto.subtle explicitly
      const hashBuffer = await window.crypto.subtle.digest('SHA-256', buffer);
      return this.binToHex(new Uint8Array(hashBuffer));
    },
    binToHex(uint8Array) {
      return Array.from(uint8Array)
        .map(byte => byte.toString(16).padStart(2, '0'))
        .join('');
    },

    // Convert Hex to Uint8Array
    hexToBin(hex) {
      return new Uint8Array(
        hex.match(/.{1,2}/g).map(byte => parseInt(byte, 16))
      );
    },

    // Convert Uint8Array to Base58 (if needed for WIF key)
    binToBase58(uint8Array) {
      return bs58.encode(uint8Array); // Requires 'bs58' library
    },

    ripemd160(buffer) {
      return new Uint8Array(
        CryptoJS.RIPEMD160(CryptoJS.enc.Hex.parse(this.binToHex(buffer)))
          .toString(CryptoJS.enc.Hex)
          .match(/.{1,2}/g)
          .map(byte => parseInt(byte, 16))
      );
    },

    
    encodeCashAddress({ prefix, type, payload }) {
      return cashaddr.encode(prefix, type.toUpperCase(), payload);
    },

    // Generates new private and public keys, WIF, and Bitcoin Cash address
    async generateNewKeys() {
      const privateKey = await this.generatePrivateKey();
      this.privateKeyWIF = privateKey.wif;
      this.publicKeyHex = privateKey.publicKey;
      this.bitcoinCashAddress = privateKey.address;
      await this.generatePublicQRCodes();
    },
    generatePublicKey(privateKeyHex) {
      const privateKey = Buffer.from(privateKeyHex, 'hex');
      const publicKey = secp256k1.getPublicKey(privateKey, true);  // 'true' for compressed public key
      return publicKey.toString('hex');
    },

    // Generate BCH address with proper format
    generateBCHAddress(publicKeyHex) {
      const publicKeyHash = this.ripemd160(this.sha256(publicKeyHex));  // Perform RIPEMD-160 on SHA-256 hash of public key
      return `bitcoincash:${publicKeyHash}`;  // BCH address should have 'bitcoincash:' prefix
    },

    // Generates a private key, derives public key and Bitcoin Cash address
    async generatePrivateKey() {
      // Generate a 32-byte private key using browser crypto API
      const privateKey = new Uint8Array(32);
      window.crypto.getRandomValues(privateKey);

      // Generate compressed public key
      const publicKey = secp256k1.publicKeyCreate(privateKey, true);

      const privateKeyHex = this.binToHex(privateKey);
      const publicKeyHex = this.binToHex(publicKey);

      const privateKeyHash = await this.sha256(privateKeyHex, 'hex');
      const publicKeyHash = await this.sha256(publicKeyHex, 'hex');

      const sha256Hash = await this.sha256(publicKeyHex, 'hex');
      const ripemdHash = this.ripemd160(this.hexToBin(sha256Hash));
      const extendedKey = new Uint8Array([0x80, ...privateKey, 0x01]);
      const hashWif1 = await this.sha256(extendedKey, 'hex');
      const hashWif2 = await this.sha256(this.hexToBin(hashWif1), 'hex');
      const checksumWif = this.hexToBin(hashWif2).slice(0, 4);
      const wifKey = new Uint8Array([...extendedKey, ...checksumWif]);
      const finalWIF = this.binToBase58(wifKey);

      return {
        privateKey: privateKeyHex,
        privateKeyHash,
        publicKey: publicKeyHex,
        publicKeyHash,
        address: this.encodeCashAddress({
          prefix: 'bitcoincash',
          type: 'P2PKH',
          payload: ripemdHash,
        }),
        wif: finalWIF
      };
    },
    // Generates multiple Bitcoin Cash addresses based on user input
    async generateMultipleKeys() {
      this.loading = true; // Start loader
    const MAX_WALLETS = 10; // Set your desired wallet limit

    // Ensure addressCount starts at 1
    if (!this.addressCount || this.addressCount < 1) {
        this.addressCount = 0;
    }

    // If no design is selected, prevent wallet generation
    if (!this.selectedDesign) {
        return;
    }

    this.customAmount = this.paymentDetails ? parseFloat(this.paymentDetails) : 0;

    // If no wallets exist, create the first static wallet
    if (this.generatedWallets.length === 0) {

        const firstWallet = await this.generatePrivateKey();
        if (!firstWallet || !firstWallet.address || !firstWallet.wif) {
            return;
        }

        try {
            firstWallet.qrCodePublic = await QRCode.toDataURL(`${firstWallet.address}?amount=${this.customAmount}`);
            firstWallet.qrCodePrivate = await QRCode.toDataURL(firstWallet.wif);
        } catch (error) {
            console.error("QR Code generation failed for static wallet:", error);
        }

        firstWallet.customAmount = this.customAmount;
        firstWallet.design = { ...this.selectedDesign };
        this.generatedWallets.push(firstWallet);
    }

    // Ensure address count does not exceed MAX_WALLETS
    this.addressCount = Math.min(Math.max(this.addressCount, 0), MAX_WALLETS);

    // Remove excess wallets while keeping the first one static
    if (this.generatedWallets.length > this.addressCount) {
          // Keep the first wallet, remove others
        this.generatedWallets = [this.generatedWallets[0], ...this.generatedWallets.slice(1, this.addressCount)];
    }

    // Generate additional wallets (excluding the static one)
    while (this.generatedWallets.length < this.addressCount) {
        if (this.generatedWallets.length >= MAX_WALLETS) {
            console.warn(`Wallet generation limit reached! Max allowed: ${MAX_WALLETS}`);
            break;
        }

        await new Promise(resolve => setTimeout(resolve, 500)); // 500ms delay

        const wallet = await this.generatePrivateKey();
        if (!wallet || !wallet.address || !wallet.wif) {
            console.error(`Wallet #${this.generatedWallets.length + 1} generation failed!`);
            continue;
        }

        try {
            wallet.qrCodePublic = await QRCode.toDataURL(`${wallet.address}?amount=${this.customAmount}`);
            wallet.qrCodePrivate = await QRCode.toDataURL(wallet.wif);
        } catch (error) {
            console.error(`QR Code generation failed for wallet #${this.generatedWallets.length + 1}:`, error);
        }

        wallet.customAmount = this.customAmount;
        wallet.design = { ...this.selectedDesign };

        this.generatedWallets.push(wallet);
    }
    this.loading = false; // Stop loader after generation
},

    // Updates the public QR code with a payment amount
    async updatePublicQRCodes() {
  if (!this.generatedWallets.length) {
    console.error("No generated wallets found!");
    return;
  }

  const amount = this.paymentDetails ? parseFloat(this.paymentDetails) : 0;

  for (const wallet of this.generatedWallets) {
    wallet.customAmount = amount; // Assign amount to each wallet

    // Ensure we don't duplicate "bitcoincash:" in the address
    const cleanAddress = wallet.address.replace(/^bitcoincash:/, '');

    // Construct QR code data with the specified amount
    let qrDataPublic = `bitcoincash:${cleanAddress}`;
    if (amount > 0) {
      qrDataPublic += `?amount=${amount}`;
    }

    try {
      wallet.qrCodePublic = await QRCode.toDataURL(qrDataPublic, {
        errorCorrectionLevel: 'L', 
      });
      console.log(`QR Code updated for ${cleanAddress}:`, wallet.qrCodePublic);
    } catch (error) {
      console.error(`Error generating QR code for ${cleanAddress}:`, error);
    }
  }
},
generateQRCode(address, amount) {
    return `https://chart.googleapis.com/chart?chs=150x150&cht=qr&chl=bitcoincash:${address}?amount=${amount}`;
  },
    // Converts the wallet section into an image and opens a print window
    async printWallet() { 
      const printable = document.getElementById("printable-wallet");
      if (!printable) {
        console.error("Printable wallet section not found!");
        return;
      }
      const canvas = await html2canvas(printable, { scale: 2 });
      const imageData = canvas.toDataURL("image/png");
      const printWindow = window.open("", "_blank");
      printWindow.document.write(`
        <html>
          <head>
            <title>Print Wallet</title>
            <style>body { text-align: center; margin: 0; padding: 0; } img { height: 100% width: 100%; max-width: 1000px; }</style>
          </head>
          <body><img src="${imageData}" alt="Printed Wallet"></body>
        </html>
      `);
      printWindow.document.close();
      printWindow.focus();
      setTimeout(() => {
        printWindow.print();
        printWindow.close();
      }, 500);
    },

    // Toggles dropdown for design selection
    toggleDropdown() {
      this.dropdownOpen = !this.dropdownOpen;
      if (!this.dropdownOpen) {
        this.selectedDesign = true;
        this.activeStep = 2;
      }
    },

    // Sets the selected design and updates UI state
    selectDesign(design) {
      this.selectedDesign = design;
      this.dropdownOpen = false;
      this.activeStep = 2;
    },

    // Toggles step visibility in UI
    toggleStep(step) {
      this.activeStep = this.activeStep === step ? null : step;
    },

    // Moves to customization step after design selection
    proceedToCustomization(design) {
      this.selectedDesign = design;
      this.generatedWallets.forEach(wallet => {
      wallet.design = { ...design };
      }); 
      this.dropdownOpen = false;
      this.activeStep = 2;
      this.addressCount = 1; 
      this.generateMultipleKeys();
    },
  },
};
</script>



<style scoped>

.tooltip-container {
  position: relative;
  display: inline-block;
  cursor: pointer;
}

.tooltip-text {
  line-height: 25px;
  visibility: hidden;
  width: 350px;
  background-color: #E2E8F0;
  color: rgb(51, 65, 85);
  text-align: left;
  padding: 8px 10px;
  border-radius: 5px;
  font-size: 0.85em;

  /* Positioning */
  position: absolute;
  z-index: 1;
  top: 125%; /* Below the link */
  left: 50%;
  transform: translateX(-50%);

  /* Optional Arrow */
  opacity: 0;
  transition: opacity 0.3s ease;
}

.tooltip-container:hover .tooltip-text {
  visibility: visible;
  opacity: 1;
}

.advanced-settings-row {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 10px;
}

.what-is-this {
  font-size: 10px;
  color: rgb(51, 65, 85);
}

.passphrase {
  margin-left: 20px;
}

.passphrase-input {
  padding: 4px 6px;
  border: 1px solid #ccc;
  border-radius: 3px;
  min-width: 150px;
}

.generate-btn {
  padding: 6px 10px;
  background-color: rgb(239, 246, 255);
  color: rgb(51, 65, 85);
  font-size: 15px;
  border: 2px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.generate-btn:hover {
  background-color: #E2E8F0;
  opacity: 10px;
}

.encryption {
  padding: 5px 10px;
  cursor: pointer;
  border: 1px solid #333;
  font-size: 13px;
  font-family: 'Lexend';
  margin-left: 40px;
  border-radius: 4px;
}

.dropdown-panel {
  margin-top: 10px;
  background-color: #f9f9f9;
  border-top: 1px solid #00cfcf;
  padding: 10px;
  width: 767px;
  display: relative;
}

.dropdown-panel input[type = "text"] {
  margin-left: 5px;
  padding: 4px;
  width: 100px;
}

.loader-wrapper {
  position: relative;
  display: inline-block;
}

.spinner {
  position: absolute;
  top: 13px;   /* Raise above input */
  left: 50px;   /* Position beside input */
  width: 16px;
  height: 16px;
  border: 2px solid #ccc;
  border-top: 2px solid #077732;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.input-bar {
  width: 60px;
  padding: 4px;
  margin-top: 10px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}



/* Dark Mode */
.dark-mode .landing-header {  /* Fixed Typo */
  background-color: #2c3440;
  color: white;
}

.dark-mode .header-padding {
  background-color: rgb(239, 246, 255);
  color: white;
}

.dark-mode .header-padding-text {
  color: #2c3440;
}

.dark-mode .landing-container,
.dark-mode .customization-section {
  background-color: #2c3440;
  color: white;
}

.dark-mode .customization-section {
  background-color: #E2E8F0;
  color: black;
}


.dark-mode .wallet-container {
  background-color: rgb(239, 246, 255);
  color: white;
}

.dark-mode .step-label1,
.dark-mode .step-label3 {
  background-color: #515863;
}

.dark-mode .step-label2 {
  background-color: #E63946;
}



/* Ensure specific text updates in dark mode */
.dark-mode .site-title,
.dark-mode .wallet-description {
  color: #E2E8F0 !important;
}

/* Light Mode */
.light-mode .landing-container,
.light-mode .landing-header,
.light-mode .wallet-container {
  background-color: rgb(239, 246, 255);
  color: black;
}

/* Ensure specific text updates in light mode */
.light-mode .site-title,
.light-mode .wallet-description{
  color: rgb(51, 65, 85) !important;
}

.light-mode .customization-section {
  background-color: #E2E8F0;
  color: black;
}




.toggle-button {
  position: fixed;
  top: 6px;
  right: 35px;
  width: 40px;
  height: 40px;
  background: #EEDC82;
  color: black;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  font-size: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
  transition: background 0.3s ease, transform 0.2s;
}

.toggle-button:hover {
  transform: scale(1.1);
}

.dark-mode .toggle-button {
  background: #121212;
  color: white;
}

.light-mode .toggle-button {
  background: #EEDC82;
  color: #333;
}





.landing-container {
  position: fixed; 
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh; 
  background-color: white;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: "Poppins", sans-serif;
  overflow-y: auto; 
  overflow-x: hidden;
}


.landing-header {
  background-color: white;
  width: calc(100% - 15px);
  padding: 15px 30px;
  position: fixed;
  top: 0;
  left: 0;
  display: flex;
  align-items: center; 
  gap: 10px; 
  z-index: 1000;
  box-sizing: border-box;
}

.header-padding {
width: 100%;
height: 140px;
background-color: rgb(30 41 59 );
margin-top: 1px;
margin-bottom: 100px;
display: flex;
padding: 5px 30px;
align-items: center;
text-align: center;
justify-content: center;
}

/*.darkmode-header {
  width: 100%;
  height: 140%;
  background-color: rgb(30 41 59 );
  margin-top: 1px;
  margin-bottom: 100px;
  display: flex;
  padding: 5px 30px;
  align-items: center;
  text-align: center;
  justify-content: center;
}*/

.header-padding-text {
font-size: 25px;
font-weight: bold;
color: white;
margin-top: 70px;
font-family: 'Lexend';
}

.site-logo {
width: 25px;
height: 25px;
}

.paper5-logo {
width: 50px;
height: 50px;
margin-right:5px;
margin-top: 50px;
}

.site-title {
font-size: 20px;
font-weight: bold;
color: rgb(30 41 59 );
font-family:'Lexend';
text-decoration: none;
}

.wallet-description {
text-align: center;
font-size: 25px;
font-weight: bold;
color: rgb(51 65 85);
margin-top: -80px;
margin-bottom: 20px;
font-family: 'Lexend';
}

.wallet-container {
  width: 60%;
  margin: 20px auto;
  border: 1px solid #ccc;
  border-radius: 8px;
  padding: 20px;
  background-color: #E2E8F0;
}

.step-container {
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 10px;
  overflow: hidden;
}

.step-label1 {
  display: flex;
  align-items: center;
  padding: 15px;
  background-color: rgb(51 65 85);
  color: white;
  cursor: pointer;
}

.step-label2 {
  display: flex;
  align-items: center;
  padding: 15px;
  background-color: #e05458;
  color: white;
  cursor: pointer;
}

.step-label3 {
  display: flex;
  align-items: center;
  padding: 15px;
  background-color: rgb(51 65 85);
  color: white;
  cursor: pointer;
}
.step-text {
  margin-left: 10px;
  font-size: 18px;
  font-weight: bold;
}

.dropdown-image {
  width: 30px;
  height: 30px;
}

.custom{
  font-weight: bold;
  margin-top: 10px;
  margin-bottom: 10px;
  margin-right: 1px;
}

.address {
  font-weight: bold;
  margin-left: 10px;
}

.input-bar {
  width: 30px;
  margin-left: 1px;
  margin-top: 10px;
}

.amount {
  width: 10px;
}

.customization-section {
  padding: 10px;
  background-color: #E2E8F0;
}

.design-grid {
  display: grid;
  margin-left: 30px;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(7, auto-fit);
  gap: 15px;
  justify-content: center;
  align-items: center;
  padding: 15px;
}

.design-preview {
  position: relative; 
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}

.design-preview img {
  width: 100%;
  height: auto; 
  object-fit: contain; 
  display: block;
}

.design-image {
  width: 100%;
  display: block;
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.4);
  opacity: 0;
  transition: opacity 0.3s ease;
}

.design-preview:hover .overlay {
  opacity: 3;
}

.select-button {
  font-family: 'Lexend', sans-serif;
  font-size: 15px; 
  position: absolute;
  bottom: 85px;
  background: rgb(51 65 85);
  color: #ccc;
  border: none;
  padding: 15px 15px;
  width: 200px;
  height: 20%;
  border-radius: 15px;
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.4s ease-in-out, transform 0.5s ease-in-out;
}


.design-preview:hover .select-button {
  opacity: 2;
}.selected-design-container {
  position: relative;
  position: relative;
  width: 100%;   
  height: 100%; 
  overflow: hidden;
}

.selected-design .design-image-container {
  position: relative;
  display: inline-block;
  padding-top: 10px;
}

.selected-design .design-image {
  width: 100%;
  height: auto;
}

.selected-design .qr-code-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 10px;
}

.selected-design .qr-section {
  position: relative;
  text-align: center;
}

.selected-design .public-section {
  position: absolute;
  display: flex;
  right: 89.9px;
  top: 90px;

}

.selected-design .private-section {
  position: absolute;
  display: flex;
  left: 10.2px; 
  top: 42px;
}

.selected-design .qr-code {
  width: 101.9px; 
  height: 101.9px;
}

.wallet-name {
  text-align: center;
  font-size: 18px;
  font-weight: 400;
  margin-bottom: 5cap;
  font-family: 'Lexend', sans-serif;
  text-transform: uppercase;
  letter-spacing: 2px;
  color: #fcf9f6f2;
}

.wallet-address{
  position: absolute;
  font-size: 0.8rem;
  color: black;
  font-weight: bold;
  top: 3%; 
  left: 71%; 
  transform: translateX(-50%);
  white-space: nowrap;
  max-width: 90%
}

.private-key {
  font-weight: 90%;
  color: white;
  position: absolute;
  font-size: 10.9px;
  top: 17vh; 
  left: 18%;
  right: 36.3%;
  transform: translateX(-53%) translateY(90%) rotate(-45.7deg); 
  white-space: nowrap;
  max-width: 90%;
  overflow: hidden;
  text-overflow: ellipsis;
}

.bch-amount {
  display: flex;
  justify-content: center;
  align-items: center; 
  text-align: center;
  font-size: 25px; 
  font-weight: bold;
  color: #DAA425;
  padding: 29%; 
  background-color: transparent;
  border-radius: 5px;
  margin: 30px auto;
  width: fit-content; 
  margin-left: 130px;
  margin-right: -50px;
  margin-top: -10px;
  margin-bottom: 10px;
}

.wallet-padding {
  margin-top: -20px;
  padding-top: -80px;
  padding: -80px;
  padding-bottom: 1px;
}

/* Responsive Styles */
@media (max-width: 1440px) {
  .design-grid {
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  }

  .design-preview {
    max-width: 300px;
  }
}

@media (max-width: 424px) {
  .landing-header {
    padding: 1px 30px;
  }

  .header-padding-text {
    font-size: 14px;
  }
}

@media (max-width: 1440px) {
  .design-grid {
    grid-template-columns: 1fr;
  }

  .select-button {
    font-size: 12px;
    padding: 8px 16px;
  }

  .wallet-description {
    font-size: 25px;
    margin-top: -40px;
  }

  .wallet-container {
    width: 100%;
    padding: 10px;
  }

  .step-text {
    font-size: 16px;
  }

  .select-button {
    font-size: 13px;
    padding: 6px 12px;
  }
}

/* Small devices (phones, 425px - 767px) */
@media (max-width: 767px) {
  .landing-header {
    padding: 10px 20px;
  }
  .selected-design .qr-code {
  width: 30px; 
  height: 30px;
  }
  .selected-design .public-section {
  position: absolute;
  display: flex;
  right: 57px; 
  top: 40px;
  }

  .selected-design .private-section {
    position: absolute;
    display: flex;
    left: 18.9px;   
    top: 41.5px;
  }
  .wallet-address{
  position: absolute;
  font-size: 20px;
  top: 3%;
  left: 67.9%; 
  transform: translateX(-50%); 
  white-space: nowrap;
  }

  .private-key {
  position: absolute;
  font-size: clamp(9px, 1.2vw, 12px); 
  top: 30.5%; 
  left: 50%; 
  transform: translateX(-50%) rotate(-45.7deg); 
  white-space: nowrap;
}
  
  .bch-amount {
    display: flex;
    justify-content: center;
    align-items: center; 
    text-align: center;
    font-size: 40px; 
    font-weight: bold;
    color: #10b01b;
    padding: 29%; 
    background-color: transparent;
    border-radius: 5px;
    margin: 30px auto;
    width: fit-content; 
    margin-left: -13.5%;
    margin-top: -10.5%;
    transform: rotate(-45.7deg);
  }

  .wallet-description {
    font-size: 20px;
    margin-top: -60px;
  }

  .wallet-container {
    width: 80%;
    padding: 15px;
  }

  .design-grid {
    grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); 
  }

  .select-button {
    font-size: 12px;
    padding: 8px 14px;
  }
}

/* Medium devices (tablets, 768px - 1023px) */
@media (max-width: 1023px) {
  .landing-header {
    padding: 15px 30px;
  }

  .wallet-container {
    width: 70%;
  }

  .design-grid {
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); 
  }
}

/* Large devices (desktops, 1024px and up) */
@media (min-width: 1024px) {
  .wallet-container {
    width: 60%;
  }

  .design-grid {
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); 
  }
}

@media (max-width: 1440px) {
  .selected-design .public-section {
    right: 11.8%; 
    top: 14.5%;
  }

  .selected-design .private-section {
    left: 1.17%;
    top: 35px;
  }

  .selected-design .qr-code {
    width: min(14vw, 100px); 
    height: min(14vw, 100px);
  }
}

@media (max-width: 768px) {
  .selected-design .public-section,
  .selected-design .private-section {
    position: static;
    text-align: center;
    margin-bottom: 20px;
  }

  .selected-design .qr-code {
    width: min(40vw, 90px);
    height: min(40vw, 90px);
  }
}
</style>

