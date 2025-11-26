# 🔐 CryptoVis: The AES-256 Visualizer
[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support%20my%20work-FFDD00?style=flat&logo=buy-me-a-coffee&logoColor=white)](https://buymeacoffee.com/quizmybrai7)

<div align="center">

![CryptoVis Banner](https://img.shields.io/badge/AES--256--GCM-Visualized-00ff41?style=for-the-badge&logo=shield&logoColor=white)
![Web Crypto API](https://img.shields.io/badge/Web%20Crypto%20API-Native%20Security-00d4ff?style=for-the-badge)
![Zero Dependencies](https://img.shields.io/badge/Dependencies-Zero-bf00ff?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-f0ff00?style=for-the-badge)

**An interactive, educational visualization of AES-256 encryption with a cyberpunk aesthetic.**

[Live Demo](#demo) • [Features](#features) • [How It Works](#how-it-works) • [Security Deep Dive](#security-deep-dive) • [Contributing](#contributing)

</div>

---

## ✨ Features

- **🔒 Real Encryption**: Uses the native [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) for actual AES-256-GCM encryption
- **👁️ Visual Process**: Watch every step of encryption happen in real-time
- **🎓 Educational**: Learn about Salt, IVs, PBKDF2, and Galois Field math
- **🌐 100% Client-Side**: No data ever leaves your browser
- **🎨 Hacker Aesthetic**: Cyberpunk/Matrix-inspired dark mode UI
- **📱 Responsive**: Works on desktop and mobile devices
- **⚡ Zero Dependencies**: Single HTML file, no build process required

---

## 🔍 How It Works

### The Encryption Pipeline

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Password   │────▶│    Salt     │────▶│   PBKDF2    │────▶│  AES Key    │
│  (User)     │     │  (Random)   │     │ (100K iter) │     │ (256-bit)   │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                   │
┌─────────────┐     ┌─────────────┐     ┌─────────────┐            │
│  Plaintext  │────▶│     IV      │────▶│ AES-256-GCM │◀───────────┘
│  (Message)  │     │  (Random)   │     │ (Encrypt)   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │ Ciphertext  │
                                        │ + Auth Tag  │
                                        └─────────────┘
```

### Output Format

The final ciphertext is structured as:

```
[Salt: 16 bytes][IV: 12 bytes][Ciphertext + Auth Tag: variable]
```

This allows the decryption function to extract everything it needs from a single Base64 string.

---

## 🛡️ Security Deep Dive

### Why Use Salt? 🧂

**The Problem**: Without salt, the same password always produces the same encryption key.

Attackers can precompute hash tables (called "rainbow tables") for millions of common passwords. If your password is "password123", they already have its hash ready.

**The Solution**: Add random bytes to the password before hashing.

```
Without Salt:
  "password123" → SHA256 → abc123...  (Same every time!)

With Salt:
  "password123" + [random salt] → SHA256 → xyz789...  (Different every time!)
```

**Real Impact**: Even if two users have the same password, their encrypted data looks completely different. Rainbow tables become useless because attackers would need a separate table for every possible salt value (2^128 possibilities with our 16-byte salt).

```javascript
// CryptoVis generates a new salt for every encryption:
const salt = crypto.getRandomValues(new Uint8Array(16));
```

---

### Why Use an IV (Initialization Vector)? 🎲

**The Problem**: Encrypting the same message twice with the same key produces identical ciphertext.

This is called "deterministic encryption" and it leaks information. An attacker could detect when you send the same message, even without decrypting it.

**The Solution**: Use a random IV that changes with every encryption.

```
Without IV:
  Key + "Attack at dawn" → [same ciphertext]
  Key + "Attack at dawn" → [same ciphertext]  ← Pattern detected!

With IV:
  Key + IV₁ + "Attack at dawn" → [ciphertext A]
  Key + IV₂ + "Attack at dawn" → [ciphertext B]  ← Completely different!
```

**Critical Warning for GCM Mode**: Never reuse an IV with the same key! In AES-GCM, IV reuse doesn't just reveal patterns—it completely breaks the encryption, allowing attackers to recover the authentication key and forge messages.

```javascript
// CryptoVis generates a unique IV for every encryption:
const iv = crypto.getRandomValues(new Uint8Array(12));  // 96 bits for GCM
```

---

### Why PBKDF2 with 100,000 Iterations? ⏱️

**The Problem**: Passwords are weak. A GPU can test billions of SHA-256 hashes per second.

**The Solution**: Make each password guess expensive with PBKDF2 (Password-Based Key Derivation Function 2).

```
Direct Hash (fast, bad):
  Password → 1 SHA-256 → Key
  Attack speed: ~10 billion guesses/second

PBKDF2 (slow, good):
  Password → 100,000 HMAC-SHA256 iterations → Key
  Attack speed: ~100,000 guesses/second
```

**The Math**: At 100,000 iterations, testing a 4-character password space (62^4 ≈ 14.7 million combinations) takes:

| Method | Time to Crack |
|--------|---------------|
| Direct SHA-256 | < 0.01 seconds |
| PBKDF2 (100K) | ~150 seconds |

For a 10-character password, PBKDF2 extends the attack time from minutes to millions of years.

```javascript
// CryptoVis uses PBKDF2 with NIST-recommended parameters:
const key = await crypto.subtle.deriveKey(
    {
        name: 'PBKDF2',
        salt: salt,
        iterations: 100000,  // Minimum recommended by OWASP
        hash: 'SHA-256'
    },
    passwordKey,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
);
```

---

### Why AES-256-GCM? 🔐

AES-GCM (Galois/Counter Mode) provides **authenticated encryption**, meaning it guarantees both:

1. **Confidentiality**: Attackers can't read the message
2. **Integrity**: Attackers can't modify the message without detection

The 128-bit authentication tag acts as a cryptographic checksum. If even a single bit of the ciphertext is changed, decryption fails.

```javascript
const ciphertext = await crypto.subtle.encrypt(
    {
        name: 'AES-GCM',
        iv: iv,
        tagLength: 128  // 128-bit authentication tag
    },
    key,
    plaintext
);
```

---

## 🧮 Crypto Math Samples

CryptoVis includes two interactive demonstrations of the mathematical operations inside AES:

### 1. Galois Field Multiplication (GF(2⁸))

AES operates in a finite field where bytes are treated as polynomials. Multiplication in GF(2⁸) is used in the MixColumns step.

```javascript
// Multiply two bytes in GF(2^8) modulo the AES polynomial x^8 + x^4 + x^3 + x + 1
function gfMultiply(a, b) {
    let result = 0;
    while (b) {
        if (b & 1) result ^= a;           // XOR if low bit set
        const highBit = a & 0x80;
        a <<= 1;
        if (highBit) a ^= 0x11B;          // Reduce modulo AES polynomial
        b >>= 1;
    }
    return result & 0xFF;
}
```

### 2. S-Box Substitution

The S-Box provides non-linearity (confusion) by computing the multiplicative inverse in GF(2⁸) followed by an affine transformation.

```javascript
function computeSBox(byte) {
    // Step 1: Multiplicative inverse in GF(2^8)
    let inverse = byte !== 0 ? gfPow(byte, 254) : 0;  // a^(-1) = a^254
    
    // Step 2: Affine transformation
    let result = inverse;
    for (let i = 0; i < 4; i++) {
        inverse = ((inverse << 1) | (inverse >> 7)) & 0xFF;
        result ^= inverse;
    }
    return result ^ 0x63;
}
```

---

## 📁 Project Structure

```
cryptovis/
├── index.html      # Everything in one file!
├── README.md       # You're reading it
└── LICENSE         # MIT License
```

---

## 🤝 Contributing

Contributions are welcome! Some ideas:

- [ ] Add more cipher modes (CBC, CTR) with visualizations
- [ ] Implement RSA key exchange demo
- [ ] Add hash function visualizations (SHA-256)
- [ ] Translate educational content
- [ ] Add more crypto math demonstrations
- [ ] Create a guided tutorial mode

---

## 📚 Further Reading

- [NIST AES Specification (FIPS 197)](https://csrc.nist.gov/publications/detail/fips/197/final)
- [Web Crypto API Documentation](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [Galois/Counter Mode (GCM) RFC 5116](https://datatracker.ietf.org/doc/html/rfc5116)

---

## ⚠️ Disclaimer

This is an **educational tool**. While it uses real cryptographic primitives correctly, please note:

- For production use, consider established libraries like [libsodium](https://libsodium.org/)
- Browser crypto relies on the browser's security model
- Always follow security best practices for key management

---

## 📄 License

MIT License - feel free to use, modify, and share!

---

<div align="center">

**Made with 💚 for the security community**

If this helped you understand cryptography better, consider giving it a ⭐

</div>
## ☕ Support the Project

If you found this tool useful, you can buy me a coffee to keep me awake while I code the next feature!

<a href="https://buymeacoffee.com/quizmybrai7">
  <img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" width="180" />
</a>
