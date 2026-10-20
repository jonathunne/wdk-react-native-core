## Legacy Secure Storage Compatibility

`secureStorage` requires an `identifier` on every call - it's what namespaces keychain
entries per wallet. That wasn't always true: the retired
`@tetherto/wdk-react-native-secure-storage` package treated `identifier` as optional, and
omitting it stored the value under the bare base key (`wallet_encryption_key`,
`wallet_encrypted_seed`, `wallet_encrypted_entropy`) - no hash, no suffix.

Apps that shipped on that package may have real users with wallets stored under that bare
key, from before per-user identifiers existed in their app (a "single wallet, no login"
mode). Making `identifier` mandatory here can't silently drop that lookup path without
orphaning those wallets.

### `DEFAULT_IDENTIFIER`

```ts
import { createSecureStorage, DEFAULT_IDENTIFIER } from '@tetherto/wdk-react-native-core'

const secureStorage = createSecureStorage()
```

Pass this in place of a real identifier to reach that exact legacy slot - internally it
bypasses the SHA-256 hash and resolves straight to the bare base key, reproducing the old
package's no-identifier lookup byte-for-byte.

- Only use it to read/migrate wallets that predate your app's per-user identifiers.
- Never assign it as a real per-user identifier - any user whose identifier collided with
  it would read and write through the same legacy slot as every other default-identifier
  wallet.
- New wallets should always be created with a real, unique identifier.

### Examples

Check for a wallet stored before your app had per-user identifiers:

```ts
const hasLegacyWallet = await secureStorage.hasWallet(DEFAULT_IDENTIFIER)
```

Read it the same way you'd read any other wallet, just with the sentinel in place of a real
identifier:

```ts
const encryptionKey = await secureStorage.getEncryptionKey(DEFAULT_IDENTIFIER)
const encryptedSeed = await secureStorage.getEncryptedSeed(DEFAULT_IDENTIFIER)
```

One-time migration to a real per-user identifier, once you have one - write under the new
identifier, then delete the legacy slot so it can't be read from again:

```ts
async function migrateLegacyWallet(realIdentifier: string): Promise<void> {
  const legacy = await secureStorage.getAllEncrypted(DEFAULT_IDENTIFIER)
  if (!legacy.encryptedSeed) return // nothing to migrate

  await secureStorage.setEncryptedSeed(legacy.encryptedSeed, realIdentifier)
  if (legacy.encryptedEntropy) {
    await secureStorage.setEncryptedEntropy(legacy.encryptedEntropy, realIdentifier)
  }
  if (legacy.encryptionKey) {
    await secureStorage.setEncryptionKey(legacy.encryptionKey, realIdentifier)
  }

  await secureStorage.deleteWallet(DEFAULT_IDENTIFIER)
}
```

A caller with an already-optional identifier just needs to coalesce it at the call site
instead of forwarding `undefined`:

```ts
async function hasLocalWallet(identifier?: string): Promise<boolean> {
  return secureStorage.hasWallet(identifier ?? DEFAULT_IDENTIFIER)
}
```
