# XMTP Browser SDK Implementation Guidelines

## Authentication Best Practices

### Client Initialization

- Initialize XMTP clients exclusively through the `Client.create` static method with an appropriate signer
- Select the right signer for your authentication method:
  - `createEOASigner` for standard wallet authentication
  - `createEphemeralSigner` for temporary sessions
  - `createSCWSigner` for smart contract wallets
- Implement comprehensive error handling around client initialization
- Add timeout handling for signature requests to prevent UI freezes
- Store authentication state securely in localStorage with proper encryption
- Always verify user registration status with `client.isRegistered()` before operations

```typescript
// Recommended client initialization pattern
const client = await Client.create(signer, {
  env: environment,
  dbEncryptionKey: encryptionKey ? hexToUint8Array(encryptionKey) : undefined,
  loggingLevel: loggingLevel,
  codecs: [...requiredCodecs],
});
```

### Signer Implementation

- Create appropriate signers based on wallet type:
  - Use `createEOASigner` for EOA wallets (MetaMask, WalletConnect)
  - Use `createEphemeralSigner` for temporary sessions or testing
  - Use `createSCWSigner` for smart contract wallets
- Ensure signers implement the complete `Signer` interface
- Include `getChainId` method for smart contract wallet signers
- Verify wallet connection before signer creation
- Implement signature caching to minimize user prompt fatigue

```typescript
// EOA signer implementation
const signer = createEOASigner(walletClient.account.address, walletClient);

// Smart Contract Wallet signer implementation
const signer = createSCWSigner(
  walletClient.account.address,
  signMessageAsync,
  BigInt(chainId)
);

// Ephemeral signer implementation
const signer = createEphemeralSigner(privateKey);
```

### Connection Management

- Always call `client.close()` when disconnecting to properly clean up resources
- Handle reconnection scenarios with appropriate state management
- Implement loading states during client initialization
- Store authentication parameters securely with encryption
- Use timestamps to identify and prevent stale initialization states

## Data Synchronization Strategies

### Conversation Synchronization

- Call `sync()` before accessing conversation data to ensure freshness
- Implement streaming patterns for real-time updates with `conversation.stream()`
- Manage stream lifecycles properly based on component mount/unmount
- Use `useEffect` cleanup functions to close streams when components unmount
- Track stream state with refs to prevent duplicate initialization

```typescript
// Recommended streaming pattern with ref tracking
const streamStartedRef = useRef(false);

useEffect(() => {
  if (!client || !conversation || streamActive || streamStartedRef.current) return;
  
  const startStream = async () => {
    try {
      // Set flag before state to prevent race conditions
      streamStartedRef.current = true;
      setStreamActive(true);
      
      const stream = await conversation.stream();
      
      const streamMessages = async () => {
        for await (const message of stream) {
          if (message) {
            setMessages((prev) => [...prev, message]);
          }
        }
      };
      
      streamMessages();
      
      return () => {
        if (stream && typeof stream.return === 'function') {
          stream.return(undefined);
        }
        setStreamActive(false);
        streamStartedRef.current = false;
      };
    } catch (error) {
      setStreamActive(false);
      streamStartedRef.current = false;
    }
  };

  startStream();
  
  return () => {
    streamStartedRef.current = false;
    setStreamActive(false);
  };
}, [client, conversation, streamActive]);
```

### Message Retrieval

- Use `conversation.messages()` with appropriate pagination options
- Always call `conversation.sync()` before fetching messages
- Implement loading states during synchronization operations
- Use streaming for real-time message updates in conversational UIs
- Handle network errors gracefully with appropriate retry logic

### Network Resilience

- Implement comprehensive error handling for network disruptions
- Add reconnection logic after network failures
- Provide clear status feedback to users during synchronization
- Monitor backend service health and handle outages gracefully

## Group Conversation Management

### Group Creation and Administration

- Follow the recommended API patterns for group operations
- Synchronize conversation state after membership changes
- Verify group existence and active status before operations
- Implement proper error handling for all group operations

```typescript
// Group management best practice
const initializeGroup = async () => {
  if (!client || !client.inboxId) return;
  
  try {
    // Sync to ensure latest data
    await client.conversations.sync();
    const conversations = await client.conversations.list();
    
    // Find group in conversations
    const group = conversations.find(conv => conv.id === groupId) as Group | undefined;
    
    if (group && group.isActive) {
      // Group is valid and active
      setCurrentGroup(group);
      
      // Fetch members for permission checks
      const members = await group.members();
      const currentUser = members.find(m => m.inboxId === client.inboxId);
      
      if (currentUser) {
        // Set permissions based on member status
        setIsAdmin(group.isAdmin(client.inboxId));
        setIsSuperAdmin(group.isSuperAdmin(client.inboxId));
      }
    } else {
      // Handle inactive or missing group
      setGroupError("Group not found or inactive");
    }
  } catch (error) {
    console.error("Group initialization error:", error);
    setGroupError("Failed to load group");
  }
};
```

### Group Message Management

- Store group conversation in application context for efficient access
- Create specialized UI components for group messages
- Track message states to provide appropriate user feedback
- Use efficient state management for message lists

## Bot Integration

- Initialize bot conversations with proper wallet identifiers
- Implement streaming for real-time bot message handling
- Manage bot conversation lifecycle with proper cleanup
- Handle bot interaction errors gracefully

```typescript
// Bot conversation initialization
const initializeBotConversation = async () => {
  if (!client) return;
  
  try {
    setLoading(true);
    
    const botConversation = await client.conversations.newDmWithIdentifier({
      identifier: BOT_ADDRESS,
      identifierKind: "Ethereum"
    });
    
    // Sync to get any existing messages
    await botConversation.sync();
    const existingMessages = await botConversation.messages();
    setMessages(existingMessages);
    
    // Start streaming for real-time updates
    const stream = await botConversation.stream();
    
    const streamMessages = async () => {
      for await (const message of stream) {
        if (message) {
          setMessages((prev) => [...prev, message]);
        }
      }
    };
    
    streamMessages();
    setBotConversation(botConversation);
    setLoading(false);
    
    return () => {
      if (stream && typeof stream.return === 'function') {
        stream.return(undefined);
      }
    };
  } catch (error) {
    console.error("Bot conversation error:", error);
    setLoading(false);
    setError("Failed to connect to bot");
  }
};
```

## Local Storage Management

### Database Security

- Always use `dbEncryptionKey` with client initialization
- Generate and securely store encryption keys
- Utilize `hexToUint8Array` for encryption key conversion
- Consider platform-specific secure storage options when available

```typescript
// Secure database encryption implementation
const generateEncryptionKey = () => {
  const randomBytes = new Uint8Array(32);
  crypto.getRandomValues(randomBytes);
  return Array.from(randomBytes)
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
};

// Initialize client with encryption
const encryptionKey = localStorage.getItem(XMTP_ENCRYPTION_KEY) || generateEncryptionKey();
localStorage.setItem(XMTP_ENCRYPTION_KEY, encryptionKey);

const client = await Client.create(signer, {
  env: environment,
  dbEncryptionKey: hexToUint8Array(encryptionKey),
  // other options
});
```

### Persistent Settings

- Use consistent localStorage key naming with proper namespacing
- Encrypt sensitive information before storage
- Follow a predictable naming pattern for all keys
- Include timestamps with initialization flags
- Clean up storage during logout operations

```typescript
// Recommended localStorage key pattern
const STORAGE_PREFIX = "xmtp:";
const KEYS = {
  CONNECTION_TYPE: `${STORAGE_PREFIX}connectionType`,
  EPHEMERAL_KEY: `${STORAGE_PREFIX}ephemeralKey`,
  ENCRYPTION_KEY: `${STORAGE_PREFIX}encryptionKey`,
  ENVIRONMENT: `${STORAGE_PREFIX}environment`,
  INITIALIZING: `${STORAGE_PREFIX}initializing`,
  INIT_TIMESTAMP: `${STORAGE_PREFIX}initTimestamp`,
};

// Storage helper functions
const storeValue = (key, value) => {
  localStorage.setItem(key, value);
};

const retrieveValue = (key) => {
  return localStorage.getItem(key);
};

const clearStorage = () => {
  Object.values(KEYS).forEach(key => localStorage.removeItem(key));
};
```

### Configuration Options

- Provide user preferences for environment selection
- Store and restore client configuration options
- Create UI components for managing essential settings
- Implement proper defaults for initial configuration

## Wallet Integration

### Environment Detection

- Detect and adapt to different wallet environments
- Support multiple wallet types (EOA, smart contract wallets)
- Adjust UI for mobile browsers and in-app wallets
- Test connectivity and permissions before operations

```typescript
// Enhanced wallet environment detection
const detectWalletEnvironment = () => {
  const info = {
    isMobile: false,
    isInApp: false,
    walletType: "unknown",
    availableProviders: [],
    isSCW: false
  };

  if (typeof window !== 'undefined') {
    // Detect mobile environment
    info.isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
    
    // Detect in-app browser
    info.isInApp = /FBAN|FBAV|Instagram|Twitter|WeChat|Line/i.test(navigator.userAgent);
    
    // Check for providers
    if (window.ethereum) {
      // Check wallet type
      if (window.ethereum.isCoinbaseWallet) {
        info.walletType = "coinbase";
        info.availableProviders.push("CoinbaseWallet");
      } else if (window.ethereum.isMetaMask) {
        info.walletType = "metamask";
        info.availableProviders.push("MetaMask");
      } else if (window.ethereum.isWalletConnect) {
        info.walletType = "walletconnect";
        info.availableProviders.push("WalletConnect");
      }
      
      // Check for smart contract wallet
      if (window.ethereum.isSmartContractWallet) {
        info.isSCW = true;
      }
    }
  }
  
  return info;
};
```

### Wagmi Configuration

- Implement consistent cookie-based storage for wagmi
- Configure appropriate chain settings
- Provide seamless session persistence
- Include comprehensive disconnection utilities

```typescript
// Optimized wagmi configuration
export const wagmiConfig = createConfig({
  storage: createStorage({
    storage: cookieStorage,
    key: "xmtp-wagmi-storage",
  }),
  ssr: true,
  chains: [mainnet, base, optimism],
  transports: {
    [mainnet.id]: http("https://mainnet.llamarpc.com"),
    [base.id]: http("https://base.llamarpc.com"),
    [optimism.id]: http("https://optimism.llamarpc.com"),
  },
  connectors: [
    injected({
      shimDisconnect: true,
    }),
    coinbaseWallet({
      appName: "Your App Name",
    }),
  ],
});
```

## Error Handling Guidelines

- Implement error boundaries around all XMTP operations
- Provide specific error handling for different error types:
  - Authentication failures
  - Network disruptions
  - Permission issues
  - Database errors
- Display user-friendly error messages
- Implement contextual logging based on environment
- Add timeout handling for potentially long operations

## Performance Optimization

- Implement efficient syncing strategies
- Use pagination with appropriate page sizes
- Close streams properly when components unmount
- Prevent UI blocking during intensive operations
- Cache signatures to minimize wallet prompts
- Optimize state updates for message lists

## Identity and Key Management

### Working with Identifiers

The XMTP SDK uses several identifier types:

- **Ethereum Address**: `0x` followed by 40 hex characters 
  - Example: `0xfb55CB623f2aB58Da17D8696501054a2ACeD1944`
  - Use for wallet identification

- **Inbox ID**: 64 hex characters
  - Example: `1180478fde9f6dfd4559c25f99f1a3f1505e1ad36b9c3a4dd3d5afb68c419179`
  - Primary XMTP conversation identifier

- **Installation ID**: 64 hex characters
  - Example: `a83166f3ab057f28d634cc04df5587356063dba11bf7d6bcc08b21a8802f4028`
  - Identifies specific client installations

```typescript
// Working with member identifiers
const getEthereumAddress = async (member) => {
  const ethIdentifier = member.accountIdentifiers.find(
    (id) => id.identifierKind === IdentifierKind.Ethereum
  );
  
  return ethIdentifier ? ethIdentifier.identifier : null;
};

// Finding members by inbox ID
const findMemberByInboxId = async (conversation, targetInboxId) => {
  const members = await conversation.members();
  return members.find(
    (member) => member.inboxId.toLowerCase() === targetInboxId.toLowerCase()
  );
};
```

## Conversation Management

### Direct Messages

```typescript
// Creating and managing DMs
const startNewConversation = async (recipientAddress) => {
  // Create conversation with Ethereum address
  const conversation = await client.conversations.newDmWithIdentifier({
    identifier: recipientAddress,
    identifierKind: IdentifierKind.Ethereum,
  });
  
  // Send initial message
  await conversation.send("Hello! I'd like to connect.");
  
  // Start streaming updates
  const stream = await conversation.stream();
  
  const streamMessages = async () => {
    for await (const message of stream) {
      if (message) {
        // Handle new message
        processNewMessage(message);
      }
    }
  };
  
  streamMessages();
  
  return conversation;
};
```

### Group Management

```typescript
// Creating and configuring groups
const createProjectGroup = async (memberAddresses) => {
  // Convert addresses to inbox IDs (implementation depends on your app)
  const memberInboxIds = await Promise.all(
    memberAddresses.map(address => getInboxIdFromAddress(address))
  );
  
  // Create the group with configuration
  const group = await client.conversations.newGroup(memberInboxIds, {
    groupName: "Project Collaboration",
    groupDescription: "A group for our project work",
    groupImageUrlSquare: "https://example.com/group-image.jpg",
  });
  
  // Set initial permissions
  if (adminInboxId) {
    await group.addAdmin(adminInboxId);
  }
  
  return group;
};
```

## Message Retrieval Patterns

### Streaming (Real-time)

```typescript
// Real-time message streaming pattern
const streamAllConversationMessages = async () => {
  const stream = await client.conversations.streamAllMessages();
  
  const processMessageStream = async () => {
    for await (const message of stream) {
      const { conversationId, content, senderInboxId, sentAt } = message;
      
      // Update UI with new message
      dispatchMessageAction({
        type: 'message_received',
        payload: {
          conversationId,
          message: {
            content,
            senderInboxId,
            sentAt
          }
        }
      });
    }
  };
  
  processMessageStream();
  
  // Return cleanup function
  return () => {
    if (stream && typeof stream.return === 'function') {
      stream.return(undefined);
    }
  };
};
```

### Polling (Batch Retrieval)

```typescript
// Batch message retrieval pattern
const fetchRecentMessages = async (conversation, limit = 50) => {
  // Ensure latest data
  await conversation.sync();
  
  // Get messages with pagination
  const messages = await conversation.messages({
    limit,
    direction: 'descending'
  });
  
  return messages.reverse(); // Return in chronological order
};
```