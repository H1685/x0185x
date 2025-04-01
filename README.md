# Secure Journal Android App

## Overview

This Android application serves as the client-side component for a secure, offline-verifiable accounting or journaling system. It allows users to create journal entries, sign them using a Bitcoin private key, encrypt them for secure transit, and later view verified entries retrieved from a public-facing server.

The core idea is to leverage Bitcoin's cryptographic signing capabilities for data integrity and RSA encryption for confidentiality during transit to an offline verification server.

## Core Concept & Workflow

The system follows a specific workflow designed for security and verifiability:

1.  **Key Management (App):**
    * The app generates a standard Bitcoin private key (WIF format) and its corresponding public address (P2PKH).
    * **(Security Note:** In this example, the key is held in memory. Production apps **must** use secure storage like Android Keystore).

2.  **Entry Creation & Signing (App):**
    * The user creates a journal entry (Date, Amount, Type (Debit/Credit), Description, Category).
    * The app formats these details into a standardized message string (e.g., `"YYYY-MM-DD,Amount,,Description"` for debit).
    * The app uses the `bitcoinj` library and the user's private key to sign this message according to the standard "Bitcoin Signed Message" format. This creates a unique cryptographic signature proving the message originated from the holder of the private key.

3.  **Payload Preparation & Encryption (App):**
    * The app constructs a JSON payload containing the user's Bitcoin address, the original message string, and the generated Base64 signature.
    * This entire JSON payload is then encrypted using the **RSA public key** of a designated **Offline Server**. This ensures only the Offline Server (which holds the corresponding RSA private key) can read the entry details.

4.  **Secure Transmission (App -> Dumb Pipe Server):**
    * The app sends the **encrypted payload** to a simple, internet-facing "Dumb Pipe" server. This server's only job is to receive encrypted data and queue it for transfer to the Offline Server. It cannot decrypt or read the content.

5.  **Offline Verification (Offline Server - *Not part of this app*):**
    * The encrypted payload is securely transferred (e.g., via USB, QR code, one-way link) from the Dumb Pipe Server to the **air-gapped Offline Server**.
    * The Offline Server uses its **RSA private key** to decrypt the payload, revealing the address, message, and signature.
    * It then uses standard Bitcoin cryptographic functions (like those in `bitcoinj`) to **verify** that the signature is valid for the given message and corresponds to the provided Bitcoin address's public key.
    * If verification succeeds, the entry is deemed authentic.

6.  **Data Formatting & Transfer (Offline Server -> Clean Data Server - *Not part of this app*):**
    * The Offline Server parses the verified message into structured data (Date, Debit, Credit, Description).
    * It prepares a **plaintext** JSON object containing the structured data, the original address, and the original signature.
    * This clean, verified data is securely transferred to the "Clean Data Server".

7.  **Public Storage & Access (Clean Data Server - *Not part of this app*):**
    * The internet-facing, read-only Clean Data Server receives the plaintext verified entries.
    * It stores these entries in a database (e.g., SQL).
    * It provides a public API (e.g., `GET /entries?address=...`) allowing anyone to query entries for a specific Bitcoin address. It returns the date, debit/credit, description, and the **original signature** for transparency and public verification.

8.  **Viewing Entries (App):**
    * The app queries the Clean Data Server's API using the user's Bitcoin address.
    * It fetches the verified journal entries associated with that address.
    * It displays these entries in a list format within the app.

## Features (Android App)

* **Bitcoin Key Pair Generation:** Creates new SECP256k1 key pairs and provides the private key (WIF) and P2PKH address.
* **Journal Entry Input:** Form for entering date, amount, type (Debit/Credit), description, and category.
* **Bitcoin Message Signing:** Uses `bitcoinj` to sign entry data using the standard Bitcoin message signing protocol.
* **RSA Payload Encryption:** Encrypts the signed payload using a configured RSA public key via JCA.
* **Secure Data Submission:** Sends the encrypted payload to a configured "Dumb Pipe" server endpoint via HTTPS POST.
* **Verified Entry Display:** Fetches and displays verified entries for the user's address from a configured "Clean Data Server" endpoint via HTTPS GET.
* **Basic UI:** Jetpack Compose based UI with Material 3 components, screen navigation (Input/Entries), and loading/error indicators.
* **Modern Styling:** UI elements inspired by shadcn/daisyUI aesthetics (clean lines, cards, clear typography).

## Technology Stack (App)

* **Language:** Kotlin
* **UI:** Jetpack Compose (Material 3, Foundation, Animation)
* **Architecture:** Single `MainActivity.kt` with ViewModel & StateFlow for state management.
* **Networking:** Ktor Client (CIO engine)
* **JSON Parsing:** Gson
* **Bitcoin Cryptography:** `org.bitcoinj:bitcoinj-core`
* **RSA Encryption:** Java Cryptography Architecture (JCA) - `javax.crypto`, `java.security`

## How to Use

1.  **Build & Run:** Compile and install the application on an Android device or emulator.
2.  **Generate Keys:**
    * Navigate to the "Add Entry" screen.
    * Tap the "Generate New Keys" button.
    * **Read the security warnings carefully.** The app will display the generated Bitcoin Address and the Private Key (WIF). **Safeguard the Private Key!** In this example, it's not saved persistently.
3.  **Create Entry:**
    * Once keys are generated, the input form is enabled.
    * Fill in the Date, Amount, select Debit/Credit, add a Description (and Category if Credit).
4.  **Submit Entry:**
    * Tap "Sign & Submit Entry".
    * The app performs the signing and encryption steps and sends the data to the configured `DUMB_PIPE_SERVER_URL`.
5.  **View Entries:**
    * Navigate to the "Entries" screen.
    * The app will automatically attempt to fetch entries for the currently generated address from the `CLEAN_DATA_SERVER_URL`.
    * *(Note: This requires the backend servers - Dumb Pipe, Offline, Clean Data - to be operational and processing entries).*

## Configuration

Before running, you **must** configure the following constants within `MainActivity.kt`:

* `DUMB_PIPE_SERVER_URL`: The HTTPS endpoint of your Dumb Pipe server.
* `CLEAN_DATA_SERVER_URL`: The HTTPS endpoint of your Clean Data server API.
* `OFFLINE_SERVER_PUBLIC_KEY_PEM`: The Base64 encoded RSA public key (PEM format, without `-----BEGIN...` headers/footers) belonging to the Offline Server.
* `BITCOIN_NETWORK_PARAMS`: Set to `MainNetParams.get()` for Bitcoin mainnet or `TestNet3Params.get()` for testnet.

## Security Considerations

* **🚨 PRIVATE KEY HANDLING (CRITICAL) 🚨:**
    * **This example app stores the private key (WIF) directly in the ViewModel's state, which is highly insecure and intended for demonstration purposes ONLY.**
    * **In a production application, NEVER store private keys this way.**
    * You **MUST** use the Android Keystore system to generate and store cryptographic keys securely in hardware-backed storage whenever possible. Alternatively, explore other secure storage solutions.
    * Exposing the private key compromises the security of all entries signed with it.

* **Server Security:** The overall system security depends on:
    * The **physical security and air-gapped nature** of the Offline Server.
    * The secure implementation of the Dumb Pipe and Clean Data servers (e.g., using HTTPS, proper access controls on the Clean Data server).
    * Secure transfer mechanisms between the Dumb Pipe -> Offline and Offline -> Clean Data servers.

* **RSA Key Distribution:** The Offline Server's RSA public key needs to be embedded securely within the app or distributed via a secure channel. If the public key is compromised or replaced, attackers could potentially trick the app into encrypting data for them (though they still couldn't fake Bitcoin signatures without the user's private key).

## Limitations

* **Example Code:** This is a single-file application for demonstration. Real-world apps require better project structure, dependency injection, more robust error handling, and unit/integration tests.
* **No Persistence:** The generated private key is lost when the app is closed. A real app needs secure persistent storage.
* **Backend Dependency:** The app requires the described backend infrastructure (Dumb Pipe, Offline Server, Clean Data Server) to be fully functional.
* **Basic UI/UX:** Features like date pickers, input validation improvements, and more sophisticated UI elements are not included.

## Dependencies

* `androidx.core:core-ktx`
* `androidx.lifecycle:lifecycle-runtime-ktx`
* `androidx.activity:activity-compose`
* `androidx.compose:compose-bom`
* `androidx.compose.ui:ui`
* `androidx.compose.ui:ui-graphics`
* `androidx.compose.ui:ui-tooling-preview`
* `androidx.compose.material3:material3`
* `androidx.compose.foundation:foundation`
* `androidx.compose.material:material-icons-extended`
* `androidx.compose.animation:animation`
* `androidx.compose.animation:animation-core`
* `androidx.lifecycle:lifecycle-viewmodel-compose`
* `org.jetbrains.kotlinx:kotlinx-coroutines-core`
* `io.ktor:ktor-client-core`
* `io.ktor:ktor-client-cio`
* `com.google.code.gson:gson`
* `org.bitcoinj:bitcoinj-core` (Ensure version compatibility, e.g., 0.16.2+)
