# User verification

Words MUST, SHOULD and MAY are to be interpreted according to [rfc2119](https://datatracker.ietf.org/doc/html/rfc2119)

Everything below is described in the context of some particular MST account M.

## Why do we need verification
Matrix specification states:
> Before Alice sends Bob encrypted data, or trusts data received from him, she may want to verify that she is actually communicating with him, rather than a man-in-the-middle.This verification process requires an out-of-band channel: there is no way to do it within Matrix without trusting the administrators of the homeservers.

General flow of verification process: 
```
    +-------------+                    +-----------+
    | AliceDevice |                    | BobDevice |
    +-------------+                    +-----------+
          |                                 |
          | m.key.verification.start        |
          |-------------------------------->|
          |                                 |
          |       m.key.verification.accept |
          |<--------------------------------|
          |                                 |
          | m.key.verification.key          |
          |-------------------------------->|
          |                                 |
          |          m.key.verification.key |
          |<--------------------------------|
          |                                 |
          | m.key.verification.mac          |
          |-------------------------------->|
          |                                 |
          |          m.key.verification.mac |
          |<--------------------------------|
          |                                 |
```

Device verification may reach one of several conclusions:
- Alice may “accept” the device. This means that she is satisfied that the device belongs to Bob.
- Alice may “reject” the device. She will do this if she knows or suspects that Bob does not control that device.
- Alice may choose to skip the device verification process. She is not able to verify that the device actually belongs to Bob, but has no reason to suspect otherwise.

### Emojis / Decimals (Short Authentication String)
Clients SHOULD show the emoji with the descriptions from the [table](https://spec.matrix.org/v1.3/client-server-api/#sas-method-emoji), or appropriate translation of those descriptions. Client authors SHOULD collaborate to create a common set of translations for all languages.

Generate 5 bytes using [HKDF](https://spec.matrix.org/v1.3/client-server-api/#hkdf-calculation) then take sequences of 13 bits to convert to decimal numbers (resulting in 3 numbers between 0 and 9191 inclusive each). Add 1000 to each calculated number.

The bitwise operations to get the numbers given the 5 bytes B0, B1, B2, B3, B4 would be:

- First: (B0 ≪ 5|B1 ≫ 3) + 1000
- Second: ((B1&0x7) ≪ 10|B2 ≪ 2|B3 ≫ 6) + 1000
- Third: ((B3&0x3F) ≪ 7|B4 ≫ 1) + 1000

It is expected to to display the numbers with some separator in between, e.g. 1337-4242-9001.

The entire SAS verification flow in short is as follows:

1. (Optional) Alice sends Bob a `m.key.verification.request` along with the methods she supports, including `m.sas.v1`.
2. (Optional) Bob receives that request. His device asks him if he wants to verify with Alice. He hits accept.
3. (Optional) Bob sends a `m.key.verification.ready` along with the methods he supports, including `m.sas.v1`.
4. Alice receives Bob's methods, and based on that, and the ones she knows how to do herself, she sends a `m.key.verification.start` for `m.sas.v1`. This will contain a bunch of parameters for how the SAS will specifically work. Alice's device notes down the canonical json of the request for later commitment verification.
5. Bob could also send an `m.key.verification.start` for `m.sas.v1`. As Alice user ID is lexicographically smaller it is discarded, though.
6. Bob received the `m.key.verification.start`, and generates his own ephemeral keypair with `var sas = olm.SAS();`. He then calculates the commitment using the canonical json of the m.key.verification.start request, before sending a `m.key.verification.accept` back to Alice, along with specific parameters to use for this SAS verificaton.
7. Alice receives the `m.key.verification.accept` and stores the commitment to verify it later. She now creates her own ephemeral key pair with `var sas = olm.SAS();` and sends Bob the public key, via m.key.verification.key.
8. Bob receives the public key from Alice and sends his own public key. Bob's device displays the emoji / numbers for verification now.
9. Alice receives Bob's public key, and can finally verify the commitment she saved earlier. If all matches, Alice's device will now display the emoji / numbers for verification.
10. If the emoji / numbers both match up, they will send each other an `m.key.verification.mac` with the MAC'd information of the keys that should be verified.
11. Finally, if all checks out, they send each other a `m.key.verification.done` and the verification process is concluded.

### Cross Signing
Rather than requiring Alice to verify each of Bob’s devices with each of her own devices and vice versa, the cross-signing feature allows users to sign their device keys such that Alice and Bob only need to verify once. With cross-signing, each user has a set of cross-signing keys that are used to sign their own device keys and other users’ keys, and can be used to trust device keys that were not verified directly.

Each user has three ed25519 key pairs for cross-signing:
- a master key (MSK) that serves as the user’s identity in cross-signing and signs their other cross-signing keys;
- a user-signing key (USK) – only visible to the user that it belongs to –that signs other users’ master keys; and
- a self-signing key (SSK) that signs the user’s own device keys.

Verification with security phrase:
```typescript
  async verifyWithPhrase(securityPhrase: string) {
    try {
      const mx = this.matrixClient;
      const defaultSSKey = mx.getAccountData('m.secret_storage.default_key').getContent().key;
      const sSKeyInfo = mx.getAccountData(`m.secret_storage.key.${defaultSSKey}`).getContent<ISecretStorageKeyInfo>();
      const { salt, iterations } = sSKeyInfo.passphrase || {};
      const privateKey = await deriveKey(securityPhrase, salt, iterations);
      const isCorrect = await mx.checkSecretStorageKey(privateKey, sSKeyInfo);
      if (isCorrect) {
        await mx.checkOwnCrossSigningTrust();
      }
      return isCorrect;
    } catch () {
      throw new Error('Verification with security phrase failed');
    }
  }
```

Verification with security key:
```typescript
  async verifyWithKey(securityKey: string) {
    try {
      const mx = this.matrixClient;
      const defaultSSKey = mx.getAccountData('m.secret_storage.default_key').getContent().key;
      const sSKeyInfo = mx.getAccountData(`m.secret_storage.key.${defaultSSKey}`).getContent<ISecretStorageKeyInfo>();
      const privateKey = mx.keyBackupKeyFromRecoveryKey(securityKey);
      const isCorrect = await mx.checkSecretStorageKey(privateKey, sSKeyInfo);
      if (isCorrect) {
        await mx.checkOwnCrossSigningTrust();
      }
      return isCorrect;
    } catch () {
      throw new Error('Verification with security key failed');
    }
  }
```

### QR codes
Verifying by QR codes provides a quick way to verify when one of the parties has a device capable of scanning a QR code. The QR code encodes both parties' master signing keys as well as a random shared secret that is used to allow bi-directional verification from a single scan.

The QR codes to be displayed and scanned using this format will encode binary strings in the general form:

- the ASCII string `MATRIX`
- one byte indicating the QR code version (must be `0x02`)
- one byte indicating the QR code verification mode. Should be one of the following values:
`0x00` verifying another user with cross-signing
`0x01` self-verifying in which the current device does trust the master key
`0x02` self-verifying in which the current device does not yet trust the master key
- the event ID or transaction_id of the associated verification request event, encoded as:
two bytes in network byte order (big-endian) indicating the length in bytes of the ID as a UTF-8 string
the ID as a UTF-8 string
- the first key, as 32 bytes. The key to use depends on the mode field:
if `0x00` or `0x01`, then the current user’s own master cross-signing public key
if `0x02`, then the current device’s device key
- the second key, as 32 bytes. The key to use depends on the mode field:
if `0x00`, then what the device thinks the other user’s master cross-signing key is
if `0x01`, then what the device thinks the other device’s device key is
if `0x02`, then what the device thinks the user’s master cross-signing key is
- a random shared secret, as a byte string. It is suggested to use a secret that is about 8 bytes long. Note: as we do not share the length of the secret, and it is not a fixed size, clients will just use the remainder of binary string as the shared secret.

For example, if Alice displays a QR code encoding the following binary string:
```
      "MATRIX"    |ver|mode| len   | event ID
4D 41 54 52 49 58  02  00   00 2D   21 41 42 43 44 ...
| user's cross-signing key    | other user's cross-signing key | shared secret
00 01 02 03 04 05 06 07 ...   10 11 12 13 14 15 16 17 ...      20 21 22 23 24 25 26 27
```
this indicates that Alice is verifying another user (say Bob), in response to the request from event “$ABCD…”, her cross-signing key is `0001020304050607`... (which is “AAECAwQFBg…” in base64), she thinks that Bob’s cross-signing key is `1011121314151617`... (which is “EBESExQVFh…” in base64), and the shared secret is `2021222324252627` (which is “ICEiIyQlJic” in base64).
