## Basic Usage

### Installation

Make sure you have jitpack enabled in your settings.gradle: 

```kotlin
repositories {
    maven { url 'https://jitpack.io' }
}
```

Add the following dependencies to your `build.gradle.kts` (always add them as `aar`, otherwise the library will not work):

```kotlin
dependencies {
    implementation("net.java.dev.jna:jna:5.14.0@aar")
    implementation("com.github.marmot-protocol:mdk-kotlin:0.8.0@aar")
}
```

**Note:** The library version is automatically synchronized with the Rust crate version from `Cargo.toml` during the build process. The version is embedded in `gradle.properties` and published to the separate `mdk-kotlin` repository. Check the repository releases or `gradle.properties` for the current version.

### Import and Initialize

```kotlin
import build.marmot.mdk.*

// Create an MDK instance backed by an encrypted SQLite database.
// The encryption key is managed automatically via the platform keyring
// (Android Keystore on Android).
val dbPath = context.filesDir.resolve("mdk.db").absolutePath
val mdk = newMdk(
    dbPath = dbPath,
    serviceId = "com.example.myapp",   // stable application identifier
    dbKeyId = "mdk.db.key.default",    // stable identifier for this DB's key
    config = null,                      // optional MdkConfig; null uses defaults
)
```

If you want to manage the encryption key yourself instead of using the
platform keyring, use `newMdkWithKey(dbPath, encryptionKey, config)`
(the key must be exactly 32 bytes).

`newMdk` initializes the platform keyring store on first use. If you need to
prime it ahead of time (or in tests), you can call the top-level
`initKeyringStore()` once at startup.

### Create and Publish Key Package

```kotlin
val publicKey = "your_hex_public_key"
val relays = listOf("wss://relay.example.com", "wss://another-relay.com")

val result = mdk.createKeyPackageForEvent(
    publicKey = publicKey,
    relays = relays
)

// result.keyPackage  — Base64-encoded key package content
// result.tags        — tags for the kind:30443 event (includes the `d` tag)
// result.tagsLegacy  — tags for the legacy kind:443 event (omits the `d` tag)
// result.hashRef     — serialized hash_ref bytes (for lifecycle tracking)
// result.dTag        — the `d` tag value; store this so you can pass it back
//                      via KeyPackageOptions::existing_d_tag when rotating
```

To rotate an existing key package while keeping the same `(kind, pubkey, d)`
address (so relays replace the previous event), use
`createKeyPackageForEventWithOptions(publicKey, relays, options: KeyPackageOptions)`
and set `options.existingDTag` to the `dTag` you stored from the prior call.

#### Build and Publish the Key Package Event

`KeyPackageResult` already contains the Base64 payload (`keyPackage`) and the
tags that need to go on the Nostr event. You only need to wrap it in your
preferred Nostr event type, sign it, and push it to the relays you want to
advertise on. Use `result.tags` with kind `30443` (the addressable form), or
`result.tagsLegacy` with kind `443` (the legacy form):

```kotlin
data class UnsignedEvent(
    val pubkey: String,
    val created_at: Long,
    val kind: Int,
    val tags: List<List<String>>,
    val content: String
)

val keyPackageResult = mdk.createKeyPackageForEvent(
    publicKey = myPublicKey,
    relays = listOf("wss://relay.example.com")
)

val unsigned = UnsignedEvent(
    pubkey = myPublicKey,
    created_at = System.currentTimeMillis() / 1000,
    kind = 30443,
    tags = keyPackageResult.tags,
    content = keyPackageResult.keyPackage
)

val signedEventJson = nostrSigner.signAndSerialize(unsigned)
relays.forEach { relay -> nostrClient.publish(relay, signedEventJson) }
```

Use whatever signer/client you already have; the key point is that the MDK
gives you the correct content and tags for the key package, which you then
embed in a standard Nostr event.

### Parse Key Packages

```kotlin
// When you receive a key package event from Nostr (kind 30443 or 443)
val eventJson = """
{
    "id": "...",
    "kind": 30443,
    "content": "base64_key_package...",
    ...
}
""".trimIndent()

// Returns a string identifying the parsed key package.
val keyPackageId = mdk.parseKeyPackage(eventJson = eventJson)
```

### Delete Key Packages from Storage

Once a KeyPackage has been consumed (e.g. used to invite you to a group) it
should be removed from MLS storage. You can delete it either by passing the
original Nostr event JSON or by passing the `hashRef` bytes you got back from
`createKeyPackageForEvent`:

```kotlin
// Delete by full key package event JSON
mdk.deleteKeyPackageFromStorage(keyPackageEventJson = eventJson)

// Or delete by the hashRef returned at creation time
mdk.deleteKeyPackageFromStorageByHashRef(hashRef = keyPackageResult.hashRef)
```

### Create a Group

```kotlin
val creatorPublicKey = "your_hex_public_key"
val memberKeyPackageEvents = listOf("{...}", "{...}") // JSON strings of key package events
val name = "My Group"
val description = "A secure group chat"
val relays = listOf("wss://relay.example.com")
val admins = listOf("your_hex_public_key")

val result = mdk.createGroup(
    creatorPublicKey = creatorPublicKey,
    memberKeyPackageEventsJson = memberKeyPackageEvents,
    name = name,
    description = description,
    relays = relays,
    admins = admins
)

// result.group contains the created group
// result.welcomeRumorsJson contains welcome messages for initial members
```

### Get Groups

```kotlin
val groups = mdk.getGroups()

groups.forEach { group ->
    println("Group: ${group.name}")
    println("ID: ${group.mlsGroupId}")
    println("State: ${group.state}")
    // To get member count, use: mdk.getMembers(mlsGroupId = group.mlsGroupId).size
}
```

### Get a Specific Group

```kotlin
val group = mdk.getGroup(mlsGroupId = "hex_group_id")
if (group != null) {
    println("Found group: ${group.name}")
} else {
    println("Group not found")
}
```

### Get Members

```kotlin
val members = mdk.getMembers(mlsGroupId = "hex_group_id")
println("Group has ${members.size} members")
members.forEach { member ->
    println("  - $member")
}
```

### Get Group Relays

```kotlin
val relays = mdk.getRelays(mlsGroupId = "hex_group_id")
relays.forEach { println(it) }
```

### Add Members to a Group

```kotlin
val mlsGroupId = "hex_group_id"
val keyPackageEvents = listOf("{...}", "{...}") // JSON strings of key package events

val result = mdk.addMembers(
    mlsGroupId = mlsGroupId,
    keyPackageEventsJson = keyPackageEvents
)

// result.evolutionEventJson contains the group update event
// result.welcomeRumorsJson contains welcome messages for new members
```

### Remove Members from a Group

```kotlin
val mlsGroupId = "hex_group_id"
val memberPublicKeys = listOf("hex_pubkey1", "hex_pubkey2")

val result = mdk.removeMembers(
    mlsGroupId = mlsGroupId,
    memberPublicKeys = memberPublicKeys
)
```

### Update Group Data

```kotlin
val mlsGroupId = "hex_group_id"

// Only set the fields you want to change; leave the others as null.
val update = GroupDataUpdate(
    name = "Updated Group Name",
    description = "New description",
    imageHash = null,
    imageKey = null,
    imageNonce = null,
    relays = listOf("wss://new-relay.com"),
    admins = null,
)

val result = mdk.updateGroupData(
    mlsGroupId = mlsGroupId,
    update = update,
)
// result is an UpdateGroupResult — see "Add Members to a Group" above.
```

### Leave a Group

Creates a proposal that, once committed by an admin, removes you from the
group. Non-admin members typically do this; admins should first call
`selfDemote` (see below).

```kotlin
val result = mdk.leaveGroup(mlsGroupId = "hex_group_id")
// result is an UpdateGroupResult — publish result.evolutionEventJson.
```

### Self-Demote (Admin)

Per MIP-03, an admin must demote themselves before calling `leaveGroup`. If you
are the last admin, designate a successor via `updateGroupData` first.

```kotlin
val result = mdk.selfDemote(mlsGroupId = "hex_group_id")
// Publish result.evolutionEventJson.
```

### Self-Update

Refresh your own MLS leaf node. Use this to rotate keys, complete a post-join
self-update (MIP-02), or recover after a stale rotation.

```kotlin
val result = mdk.selfUpdate(mlsGroupId = "hex_group_id")
```

To find groups that currently need a self-update (post-join or older than the
given threshold in seconds):

```kotlin
val needsUpdate = mdk.groupsNeedingSelfUpdate(thresholdSecs = 86_400u)
needsUpdate.forEach { mdk.selfUpdate(mlsGroupId = it) }
```

### Delete a Group

Removes all local state for a group. This is a local operation only; it does
not send anything to other members.

```kotlin
mdk.deleteGroup(mlsGroupId = "hex_group_id")
```

### Sync Group Metadata from MLS

Re-reads the latest committed group metadata from the underlying MLS state
into the MDK storage layer. Use this if the cached `Group` row has drifted
from MLS truth.

```kotlin
mdk.syncGroupMetadataFromMls(mlsGroupId = "hex_group_id")
```

### Recover From a Failed Commit

When a commit you produced couldn't be published (e.g. all relays rejected
the evolution event after retries), roll the group back to its pre-commit
state:

```kotlin
mdk.clearPendingCommit(mlsGroupId = "hex_group_id")
```

The opposite operation, finalizing a commit that you've already published
and confirmed, is `mergePendingCommit`:

```kotlin
mdk.mergePendingCommit(mlsGroupId = "hex_group_id")
```

### Accept Welcome Messages

```kotlin
// Get pending welcomes (pass limit / offset, or null for no paging)
val welcomes = mdk.getPendingWelcomes(limit = null, offset = null)

welcomes.forEach { welcome ->
    println("Invited to: ${welcome.groupName}")
    println("By: ${welcome.welcomer}")

    // Accept the welcome (pass the Welcome itself, not its JSON)
    mdk.acceptWelcome(welcome)
}
```

### Decline Welcome Messages

```kotlin
val welcome = welcomes.first()
mdk.declineWelcome(welcome)
```

### Process a Raw Welcome

If you receive a welcome rumor outside of `processMessage` (e.g. unwrapped
yourself), turn it into a stored `Welcome` so you can `acceptWelcome` it later:

```kotlin
val welcome = mdk.processWelcome(
    wrapperEventId = "hex_wrapper_event_id",  // gift-wrap (kind 1059) event ID
    rumorEventJson = unwrappedRumorJson,
)
```

### Get a Welcome by Event ID

```kotlin
val welcome = mdk.getWelcome(eventId = "hex_event_id")
```

### JSON-Based Welcome Accept/Decline

The preferred API is `acceptWelcome(welcome)` / `declineWelcome(welcome)`, but
JSON-string variants exist for callers that don't have a `Welcome` object
handy:

```kotlin
mdk.acceptWelcomeJson(welcomeJson = welcomeEventJson)
mdk.declineWelcomeJson(welcomeJson = welcomeEventJson)
```

### Create and Send Messages

```kotlin
val mlsGroupId = "hex_group_id"
val senderPublicKey = "your_hex_public_key"
val content = "Hello, group!"
val kind: UShort = 9u // Message kind

val eventJson = mdk.createMessage(
    mlsGroupId = mlsGroupId,
    senderPublicKey = senderPublicKey,
    content = content,
    kind = kind,
    tags = null,        // optional inner-event tags
    eventTags = null,   // optional outer event tags
)

// eventJson is a JSON string of the encrypted Nostr event
// Publish this to your Nostr relays
```

### Get Messages

```kotlin
val messages = mdk.getMessages(
    mlsGroupId = "hex_group_id",
    limit = null,        // optional UInt
    offset = null,       // optional UInt
    sortOrder = null,    // optional "asc" / "desc"
)

messages.forEach { message ->
    println("From: ${message.senderPubkey}")
    println("Event JSON: ${message.eventJson}")
    println("Kind: ${message.kind}")
    // Note: To extract decrypted content, parse the eventJson and extract the content field
}
```

### Get a Specific Message

```kotlin
val message = mdk.getMessage(
    mlsGroupId = "hex_group_id",
    eventId = "hex_event_id",
)
if (message != null) {
    println("Message event JSON: ${message.eventJson}")
    // Note: To extract decrypted content, parse the eventJson and extract the content field
}
```

### Get the Last Message

Returns the most recent message under the requested ordering. The cached
`Group.lastMessageId` always reflects `"created_at_first"`, so if your UI
sorts by `"processed_at_first"` use this method instead for a consistent
"last message" value.

```kotlin
val last = mdk.getLastMessage(
    mlsGroupId = "hex_group_id",
    sortOrder = "created_at_first",   // or "processed_at_first"
)
```

### Delete All Messages for a Group

Removes all locally-stored messages for the group and returns the number of
deleted rows. This is local-only and does not affect remote relays or other
members.

```kotlin
val deleted = mdk.deleteMessagesForGroup(mlsGroupId = "hex_group_id")
println("Deleted $deleted messages")
```

### Process Incoming Messages

```kotlin
// When you receive a message event from Nostr
val eventJson = """
{
    "id": "...",
    "kind": 1059,
    "content": "encrypted_content...",
    ...
}
""".trimIndent()

val result = mdk.processMessage(eventJson = eventJson)

when (result) {
    is ProcessMessageResult.ApplicationMessage -> {
        // A regular chat-style message.
        println("New message event JSON: ${result.message.eventJson}")
        // Note: To extract decrypted content, parse the eventJson and extract the content field
    }
    is ProcessMessageResult.Proposal -> {
        // A proposal that was auto-committed by an admin receiver.
        // result.result is an UpdateGroupResult to publish.
        println("Proposal merged for group ${result.result.mlsGroupId}")
    }
    is ProcessMessageResult.PendingProposal -> {
        // A proposal stored but not committed (receiver is not admin).
        println("Pending proposal for group ${result.mlsGroupId}")
    }
    is ProcessMessageResult.ExternalJoinProposal -> {
        println("External join proposal for group ${result.mlsGroupId}")
    }
    is ProcessMessageResult.Commit -> {
        println("Commit applied to group ${result.mlsGroupId}")
    }
    is ProcessMessageResult.Unprocessable -> {
        println("Unprocessable message for group ${result.mlsGroupId}")
    }
    is ProcessMessageResult.IgnoredProposal -> {
        println("Ignored proposal for ${result.mlsGroupId}: ${result.reason}")
    }
    ProcessMessageResult.PreviouslyFailed -> {
        // Message was previously marked as failed and cannot be reprocessed.
        println("Previously failed message")
    }
}
```

### Process with MLS Context

Like `processMessage`, but the returned value also carries transient MLS
context such as the sender's leaf index, useful for UI display or
verification.

```kotlin
val result = mdk.processMessageWithContext(eventJson = eventJson)
// result is a ProcessMessageWithContextResult — same variants as
// ProcessMessageResult, with additional context fields attached.
```

## Encrypted Media (Messages)

MDK can encrypt attachments with the group's MLS epoch key, produce an IMETA
tag that lets recipients decrypt the file, and unwind the same process on the
receive side. Upload the encrypted bytes to any Blossom server.

### Encrypt and Attach Media

```kotlin
val upload = mdk.encryptMediaForUpload(
    mlsGroupId = mlsGroupId,
    data = fileBytes,
    mimeType = "image/jpeg",
    filename = "photo.jpg",
)

// Upload upload.encryptedData to your Blossom server, then build the IMETA tag:
val uploadedUrl = blossomClient.upload(upload.encryptedData)
val imetaTag = mdk.createMediaImetaTag(
    mlsGroupId = mlsGroupId,
    upload = upload,
    uploadedUrl = uploadedUrl,
)

// Pass imetaTag through createMessage's `tags` parameter to attach it.
```

For custom EXIF stripping, preview-hash, or size limits, use
`encryptMediaForUploadWithOptions(..., options: MediaProcessingOptionsInput)`.

### Parse and Decrypt Media

```kotlin
// imetaTag was extracted from the received message's tags
val reference = mdk.parseMediaImetaTag(
    mlsGroupId = mlsGroupId,
    imetaTag = imetaTag,
)

val encryptedBytes = blossomClient.download(reference.url)
val plaintext = mdk.decryptMediaFromDownload(
    mlsGroupId = mlsGroupId,
    encryptedData = encryptedBytes,
    reference = reference,
)
```

## Group Image Helpers

Group images (set via `GroupDataUpdate.imageHash` / `imageKey` / `imageNonce`)
use a separate, top-level helper API. These are package-level functions, not
methods on `Mdk`.

```kotlin
// Encrypt and prepare a group image for upload
val prepared = prepareGroupImageForUpload(
    imageData = imageBytes,
    mimeType = "image/png",
)
// prepared contains the encrypted bytes plus the hash/key/nonce to store
// in the Group via updateGroupData.

// On the receiving side, decrypt with the values stored on the Group
val original = decryptGroupImage(
    encryptedData = downloadedBytes,
    expectedHash = group.imageHash,
    imageKey = group.imageKey!!,
    imageNonce = group.imageNonce!!,
)

// Derive the deterministic upload keypair (Blossom auth) from the image key
val pubkey = deriveUploadKeypair(imageKey = group.imageKey!!, version = 1u)
```

For custom processing, use `prepareGroupImageForUploadWithOptions(..., options)`.

## Group Capabilities and Diagnostics

These methods expose lower-level MLS state. Most apps won't need them, but
they are useful for capability upgrades, debugging, and admin tooling.

### Capability Upgrades

```kotlin
// Inspect upgrade readiness for each mirrored proposal type
val status = mdk.groupCapabilityUpgradeStatus(groupIdHex = "hex_group_id")

// Admin: add the upgradeable proposal types to the group's required capabilities
val result = mdk.upgradeGroupCapabilities(
    groupIdHex = "hex_group_id",
    proposalsToAdd = listOf(MdkProposalType.SELF_REMOVE),
)
```

### Required Proposals

```kotlin
// The proposal types every member is required to support in this group
val required = mdk.groupRequiredProposals(groupIdHex = "hex_group_id")
```

### Per-Member Capabilities

```kotlin
val caps = mdk.groupMemberCapabilities(groupIdHex = "hex_group_id")
// caps is ordered by MLS leaf index.
```

### Leaf Indices and Tree Info

```kotlin
val myLeaf = mdk.ownLeafIndex(groupIdHex = "hex_group_id")

// (leafIndex, hex_pubkey) entries for every active leaf — removed-member holes are omitted
val leafMap = mdk.groupLeafMap(groupIdHex = "hex_group_id")

// Full ratchet-tree snapshot (for debugging or external verification)
val treeInfo = mdk.getRatchetTreeInfo(groupIdHex = "hex_group_id")
```

### Pending Proposal Inspection

Before a commit lands, you can inspect what would change:

```kotlin
val toAdd     = mdk.pendingAddedMembersPubkeys(groupIdHex = "hex_group_id")
val toRemove  = mdk.pendingRemovedMembersPubkeys(groupIdHex = "hex_group_id")
val combined  = mdk.pendingMemberChanges(groupIdHex = "hex_group_id")
```

## Error Handling

All MDK operations can throw `MdkUniffiException`:

```kotlin
try {
    val groups = mdk.getGroups()
    // Use groups...
} catch (e: MdkUniffiException.Storage) {
    println("Storage error: ${e.message}")
} catch (e: MdkUniffiException.Mdk) {
    println("MDK error: ${e.message}")
} catch (e: MdkUniffiException.InvalidInput) {
    println("Invalid input: ${e.message}")
}
```

## Data Types

### Group

```kotlin
data class Group(
    val mlsGroupId: String,              // Hex-encoded MLS group ID
    val nostrGroupId: String,            // Hex-encoded Nostr group ID
    val name: String,
    val description: String,
    val imageHash: ByteArray?,           // Optional group image hash
    val imageKey: ByteArray?,            // Optional group image encryption key
    val imageNonce: ByteArray?,          // Optional group image encryption nonce
    val adminPubkeys: List<String>,      // Admin public keys (hex-encoded)
    val lastMessageId: String?,          // Last message event ID (hex-encoded)
    val lastMessageAt: ULong?,           // Sender's created_at of last message (Unix timestamp)
    val lastMessageProcessedAt: ULong?,  // When this client received the last message (Unix timestamp)
    val epoch: ULong,                    // Current epoch number
    val state: String,                   // Group state (e.g., "active", "archived")
    val selfUpdateState: String,         // "required" or "completed_at:<unix_timestamp>"
)
```

### Message

```kotlin
data class Message(
    val id: String,                     // Message ID (hex-encoded event ID)
    val mlsGroupId: String,             // Hex-encoded MLS group ID
    val nostrGroupId: String,           // Hex-encoded Nostr group ID
    val eventId: String,                // Event ID (hex-encoded)
    val senderPubkey: String,           // Sender public key (hex-encoded)
    val eventJson: String,              // JSON representation of the event
    val createdAt: ULong,               // Sender's created_at (Unix timestamp; may skew vs processedAt)
    val processedAt: ULong,             // When this client processed/received it (Unix timestamp)
    val kind: UShort,                   // Message kind
    val state: String,                  // Message state (e.g., "processed", "pending")
)
```

### Welcome

```kotlin
data class Welcome(
    val id: String,                     // Welcome ID (hex-encoded event ID)
    val eventJson: String,              // JSON representation of the welcome event
    val mlsGroupId: String,             // Hex-encoded MLS group ID
    val nostrGroupId: String,           // Hex-encoded Nostr group ID
    val groupName: String,
    val groupDescription: String,
    val groupImageHash: ByteArray?,     // Optional group image hash
    val groupImageKey: ByteArray?,      // Optional group image encryption key
    val groupImageNonce: ByteArray?,    // Optional group image encryption nonce
    val groupAdminPubkeys: List<String>, // List of admin public keys (hex-encoded)
    val groupRelays: List<String>,      // List of relay URLs for the group
    val welcomer: String,               // Welcomer public key (hex-encoded)
    val memberCount: UInt,              // Current member count
    val state: String,                  // Welcome state (e.g., "pending", "accepted", "declined")
    val wrapperEventId: String,         // Wrapper event ID (hex-encoded)
)
```

### KeyPackageResult

```kotlin
data class KeyPackageResult(
    val keyPackage: String,                // Base64-encoded key package content
    val tags: List<List<String>>,          // Tags for the kind:30443 event (includes the `d` tag)
    val tagsLegacy: List<List<String>>,    // Tags for the legacy kind:443 event (omits the `d` tag)
    val hashRef: ByteArray,                // Serialized hash_ref bytes (for lifecycle tracking)
    val dTag: String,                      // The `d` tag value for this KeyPackage slot
)
```

## Thread Safety

A given `Mdk` instance must be confined to a single thread and must not be shared across threads. If you need to use MDK from multiple threads, create separate isolated `Mdk` instances per thread. Note that multi-threaded usage with separate instances is not a supported concurrency model.

## Android Integration

### Native Libraries

The package includes native libraries for:
- `arm64-v8a` (64-bit ARM)
- `armeabi-v7a` (32-bit ARM)

Place the `.so` files in your `src/main/jniLibs/` directory structure, or use the packaged AAR which includes them automatically.

### Coroutines Example

Since `Mdk` instances must be confined to a single thread, all MDK operations should be serialized using a single-threaded dispatcher:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.ExecutorCoroutineDispatcher
import java.util.concurrent.Executors

class MdkManager(private val context: Context) {
    // Single-threaded dispatcher to ensure all MDK operations run on the same thread
    private val mdkDispatcher = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
    private val mdk = newMdk(
        dbPath = context.filesDir.resolve("mdk.db").absolutePath,
        serviceId = "com.example.myapp",
        dbKeyId = "mdk.db.key.default",
        config = null,
    )
    
    suspend fun getGroupsAsync(): List<Group> = withContext(mdkDispatcher) {
        mdk.getGroups()
    }
    
    suspend fun createMessageAsync(
        groupId: String,
        senderPublicKey: String,
        content: String
    ): String = withContext(mdkDispatcher) {
        mdk.createMessage(
            mlsGroupId = groupId,
            senderPublicKey = senderPublicKey,
            content = content,
            kind = 9u,
            tags = null,
            eventTags = null,
        )
    }
    
    // Clean up dispatcher when done
    fun close() {
        (mdkDispatcher as? ExecutorCoroutineDispatcher)?.close()
    }
}
```

## Example: Complete Workflow

```kotlin
import build.marmot.mdk.*

// 1. Initialize
val dbPath = "/path/to/mdk.db"
val mdk = newMdk(
    dbPath = dbPath,
    serviceId = "com.example.myapp",
    dbKeyId = "mdk.db.key.default",
    config = null,
)

// 2. Create and publish key package
val keyPackage = mdk.createKeyPackageForEvent(
    publicKey = myPublicKey,
    relays = listOf("wss://relay.example.com")
)
// Publish keyPackage.keyPackage as a Nostr event of kind 30443
// (use keyPackage.tags), or kind 443 (use keyPackage.tagsLegacy).

// 3. Create a group
val groupResult = mdk.createGroup(
    creatorPublicKey = myPublicKey,
    memberKeyPackageEventsJson = listOf(memberKeyPackageEventJson),
    name = "My Group",
    description = "A test group",
    relays = listOf("wss://relay.example.com"),
    admins = listOf(myPublicKey)
)

// 4. Send a message
val messageEvent = mdk.createMessage(
    mlsGroupId = groupResult.group.mlsGroupId,
    senderPublicKey = myPublicKey,
    content = "Hello!",
    kind = 9u,
    tags = null,
    eventTags = null,
)
// Publish messageEvent to Nostr relays

// 5. Retrieve messages
val messages = mdk.getMessages(
    mlsGroupId = groupResult.group.mlsGroupId,
    limit = null,
    offset = null,
    sortOrder = null,
)
messages.forEach { message ->
    println("${message.senderPubkey}: ${message.eventJson}")
    // Note: To extract decrypted content, parse the eventJson and extract the content field
}
```

## Integration with Android ViewModel

Since `Mdk` instances must be confined to a single thread, all MDK operations should be serialized using a single-threaded dispatcher:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import kotlinx.coroutines.ExecutorCoroutineDispatcher
import java.util.concurrent.Executors

class GroupViewModel(
    private val mdk: Mdk,
    private val senderPublicKey: String
) : ViewModel() {
    // Single-threaded dispatcher to ensure all MDK operations run on the same thread
    private val mdkDispatcher = Executors.newSingleThreadExecutor().asCoroutineDispatcher()
    
    private val _groups = MutableStateFlow<List<Group>>(emptyList())
    val groups: StateFlow<List<Group>> = _groups
    
    init {
        loadGroups()
    }
    
    private fun loadGroups() {
        viewModelScope.launch {
            try {
                _groups.value = withContext(mdkDispatcher) {
                    mdk.getGroups()
                }
            } catch (e: Exception) {
                // Handle error
            }
        }
    }
    
    fun sendMessage(groupId: String, content: String) {
        viewModelScope.launch {
            try {
                val eventJson = withContext(mdkDispatcher) {
                    mdk.createMessage(
                        mlsGroupId = groupId,
                        senderPublicKey = senderPublicKey,
                        content = content,
                        kind = 9u,
                        tags = null,
                        eventTags = null,
                    )
                }
                // Publish to Nostr
            } catch (e: Exception) {
                // Handle error
            }
        }
    }
    
    override fun onCleared() {
        super.onCleared()
        // Clean up dispatcher when ViewModel is cleared
        (mdkDispatcher as? ExecutorCoroutineDispatcher)?.close()
    }
}
```

