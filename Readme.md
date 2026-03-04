## Basic Usage

### Installation

![marmot](https://blossom.primal.net/392245b37b513b93947a930bb18a75d170e5ddb0e455ec3d073994e6c445d1e7.png)

Make sure you have jitpack enabled in your settings.gradle: 

```kotlin
repositories {
    maven { url 'https://jitpack.io' }
}
```

Add the following dependencies (always add them as aar! it will not work otherwise): 

```kotlin
dependencies {
    implementation("net.java.dev.jna:jna:5.14.0@aar")
    implementation("com.github.marmot-protocol:mdk-kotlin:0.6.0@aar") 
}
```

**Note:** The library version is automatically synchronized with the Rust crate version from `Cargo.toml` during the build process. The version is embedded in `gradle.properties` and published to the separate `mdk-kotlin` repository. Check the repository releases or `gradle.properties` for the current version.

to your build.gradle.kts

### Import and Initialize

![marmot](https://blossom.primal.net/fbf39578e7bed162419c9c249a7ee18cbbaa68de8c8815359ca13428e7d74135.png)

```kotlin
import build.marmot.mdk.*

// Create an MDK instance with a SQLite database path
val dbPath = context.filesDir.resolve("mdk.db").absolutePath
val mdk = newMdk(dbPath)
```

### Create and Publish Key Package

![marmot](https://blossom.primal.net/c1801b3637aaaf374ef729ae327e3566e1520784c54837adde89cc38f9912dd3.png)

```kotlin
val publicKey = "your_hex_public_key"
val relays = listOf("wss://relay.example.com", "wss://another-relay.com")

val result = mdk.createKeyPackageForEvent(
    publicKey = publicKey,
    relays = relays
)

// result.keyPackage contains the hex-encoded key package
// result.tags contains Nostr event tags (List<List<String>>)
// Publish as a Nostr event (kind 443) to your relays
```

#### Build and Publish a Kind 443 Event

`KeyPackageResult` already contains the payload (`keyPackage`) and all tags that
need to go on the Nostr event. You only need to wrap it in your preferred Nostr
event type, sign it, and push it to the relays you want to advertise on:

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
    kind = 443,
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

![marmot](https://blossom.primal.net/3a93f1f2b773d1236edc50d3bec5d1160d77ad46847d67418913483102d36f5d.png)

```kotlin
// When you receive a key package event from Nostr
val eventJson = """
{
    "id": "...",
    "kind": 443,
    "content": "hex_key_package...",
    ...
}
""".trimIndent()

mdk.parseKeyPackage(eventJson = eventJson)
```

### Create a Group

![marmot](https://blossom.primal.net/cedc0680b3a8f9b00cd527555ae770815f3a96f18775ce16d445457e7a4f6ae6.png)

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

### Update Group Metadata

```kotlin
val mlsGroupId = "hex_group_id"
val newName = "Updated Group Name"
val newDescription = "New description"
val newRelays = listOf("wss://new-relay.com")

val result = mdk.updateGroupMetadata(
    mlsGroupId = mlsGroupId,
    name = newName,
    description = newDescription,
    relays = newRelays
)
```

### Accept Welcome Messages

![marmot](https://blossom.primal.net/ec8bcff479764bd9f9df8d3aec106b348a653c35f4c4c6711745071f04f8aff2.png)

```kotlin
// Get pending welcomes
val welcomes = mdk.getPendingWelcomes()

welcomes.forEach { welcome ->
    println("Invited to: ${welcome.groupName}")
    println("By: ${welcome.welcomer}")
    
    // Accept the welcome
    mdk.acceptWelcome(welcomeJson = welcome.eventJson)
}
```

### Decline Welcome Messages

```kotlin
val welcome = welcomes.first()
mdk.declineWelcome(welcomeJson = welcome.eventJson)
```

### Create and Send Messages

![marmot](https://blossom.primal.net/83d56f90973787858c085ab51cf2c4454134a2a6cdf98d996c464ea00f23baad.png)

```kotlin
val mlsGroupId = "hex_group_id"
val senderPublicKey = "your_hex_public_key"
val content = "Hello, group!"
val kind: UShort = 9u // Message kind

val eventJson = mdk.createMessage(
    mlsGroupId = mlsGroupId,
    senderPublicKey = senderPublicKey,
    content = content,
    kind = kind
)

// eventJson is a JSON string of the encrypted Nostr event
// Publish this to your Nostr relays
```

### Get Messages

```kotlin
val messages = mdk.getMessages(mlsGroupId = "hex_group_id")

messages.forEach { message ->
    println("From: ${message.senderPubkey}")
    println("Event JSON: ${message.eventJson}")
    println("Kind: ${message.kind}")
    // Note: To extract decrypted content, parse the eventJson and extract the content field
}
```

### Get a Specific Message

```kotlin
val message = mdk.getMessage(eventId = "hex_event_id")
if (message != null) {
    println("Message event JSON: ${message.eventJson}")
    // Note: To extract decrypted content, parse the eventJson and extract the content field
}
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
    is MessageProcessingResult.NewMessage -> {
        println("New message event JSON: ${result.newMessage.eventJson}")
        // Note: To extract decrypted content, parse the eventJson and extract the content field
    }
    is MessageProcessingResult.Duplicate -> {
        println("Message already processed")
    }
    is MessageProcessingResult.Error -> {
        println("Error processing: ${result.error}")
    }
}
```

### Process Multiple Messages

```kotlin
val eventJsons = listOf("{...}", "{...}", "{...}")
val results = mdk.processMessages(eventJsons = eventJsons)

results.forEach { result ->
    if (result is MessageProcessingResult.NewMessage) {
        println("Processed message event JSON: ${result.newMessage.eventJson}")
        // Note: To extract decrypted content, parse the eventJson and extract the content field
    }
}
```

## Error Handling

![marmot](https://blossom.primal.net/5d3892de07d242caa190e655e09c6501c450e15968a3c430df7c2c4752b461ab.png)

All MDK operations can throw `MdkUniffiError`:

```kotlin
try {
    val groups = mdk.getGroups()
    // Use groups...
} catch (e: MdkUniffiError.Storage) {
    println("Storage error: ${e.message}")
} catch (e: MdkUniffiError.Mdk) {
    println("MDK error: ${e.message}")
} catch (e: MdkUniffiError.InvalidInput) {
    println("Invalid input: ${e.message}")
}
```

## Data Types

![marmot](https://blossom.primal.net/5f94fd8d54356223dfb54117ed20a978696737df822bcff22baa8df1c6662a69.png)

### Group

```kotlin
data class Group(
    val mlsGroupId: String,              // Hex-encoded MLS group ID
    val nostrGroupId: String,            // Hex-encoded Nostr group ID
    val name: String,
    val description: String,
    val imageHash: List<UByte>?,        // Optional group image hash
    val imageKey: List<UByte>?,         // Optional group image encryption key
    val imageNonce: List<UByte>?,       // Optional group image encryption nonce
    val adminPubkeys: List<String>,     // Admin public keys (hex-encoded)
    val lastMessageId: String?,         // Last message event ID (hex-encoded)
    val lastMessageAt: ULong?,          // Timestamp of last message (Unix timestamp)
    val epoch: ULong,                   // Current epoch number
    val state: String                   // Group state (e.g., "active", "archived")
)
```

### Message

```kotlin
data class Message(
    val id: String,                     // Message ID (hex-encoded event ID)
    val mlsGroupId: String,             // Hex-encoded MLS group ID
    val nostrGroupId: String,            // Hex-encoded Nostr group ID
    val eventId: String,                // Event ID (hex-encoded)
    val senderPubkey: String,           // Sender public key (hex-encoded)
    val eventJson: String,              // JSON representation of the event
    val processedAt: ULong,             // Timestamp when message was processed (Unix timestamp)
    val kind: UShort,                   // Message kind
    val state: String                   // Message state (e.g., "processed", "pending")
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
    val groupImageHash: List<UByte>?,   // Optional group image hash
    val groupImageKey: List<UByte>?,    // Optional group image encryption key
    val groupImageNonce: List<UByte>?,  // Optional group image encryption nonce
    val groupAdminPubkeys: List<String>, // List of admin public keys (hex-encoded)
    val groupRelays: List<String>,      // List of relay URLs for the group
    val welcomer: String,               // Welcomer public key (hex-encoded)
    val memberCount: UInt,              // Current member count
    val state: String,                  // Welcome state (e.g., "pending", "accepted", "declined")
    val wrapperEventId: String         // Wrapper event ID (hex-encoded)
)
```

### KeyPackageResult

```kotlin
data class KeyPackageResult(
    val keyPackage: String,         // Hex-encoded key package
    val tags: List<List<String>>    // Nostr event tags
)
```

## Thread Safety

![marmot](https://blossom.primal.net/aef8e66dd3557d1eba4f05d3278b0f269468a32a9182272d3d12f77b1854524c.png)

A given `Mdk` instance must be confined to a single thread and must not be shared across threads. If you need to use MDK from multiple threads, create separate isolated `Mdk` instances per thread. Note that multi-threaded usage with separate instances is not a supported concurrency model.

## Android Integration

![marmot](https://blossom.primal.net/8279fbcbd6f8eddd768a8a204eb984b9407a1a8893a9bf99e4933ba50a9376d4.png)

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
    private val mdk = newMdk(context.filesDir.resolve("mdk.db").absolutePath)
    
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
            kind = 9u
        )
    }
    
    // Clean up dispatcher when done
    fun close() {
        (mdkDispatcher as? ExecutorCoroutineDispatcher)?.close()
    }
}
```

## Example: Complete Workflow

![marmot](https://blossom.primal.net/49a2e700df6aadc7ee4a3b2a6b51e06f3de3ca3b6add5cf7ce3d8f39b5180d3e.png)

```kotlin
import build.marmot.mdk.*

// 1. Initialize
val dbPath = "/path/to/mdk.db"
val mdk = newMdk(dbPath)

// 2. Create and publish key package
val keyPackage = mdk.createKeyPackageForEvent(
    publicKey = myPublicKey,
    relays = listOf("wss://relay.example.com")
)
// Publish keyPackage.keyPackage as Nostr event kind 443

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
    kind = 9u
)
// Publish messageEvent to Nostr relays

// 5. Retrieve messages
val messages = mdk.getMessages(mlsGroupId = groupResult.group.mlsGroupId)
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
                        kind = 9u
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
