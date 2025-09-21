# pyVinty By mxo23 - Public Functions Reference

This document provides complete documentation for all public functions available in pyVinty, including their parameters and return data.

## Table of Contents

1. [CloudScraper Session Configuration](#cloudscraper-session-configuration)
   - [Basic Setup](#basic-setup)
   - [Using Pre-stored Tokens from Database](#using-pre-stored-tokens-from-database)
   - [Cookie and Token Management](#cookie-and-token-management)
2. [Utility Functions (tokens.py)](#utility-functions-tokenspy)
   - [User Information Retrieval - Multiple Approaches](#user-information-retrieval---multiple-approaches)
3. [Services](#services)
   - [MessagingService](#messagingservice)
   - [PaymentService](#paymentservice)
   - [NotificationService](#notificationservice)
   - [AuthService](#authservice)
4. [DataDomeHelper](#datadomehelper)
5. [Best Practices Summary](#best-practices-summary)
   - [Database Integration Patterns](#database-integration-patterns)

---

## CloudScraper Session Configuration

Before using pyVinty, configure a session with CloudScraper.

### Basic Setup

```python
import asyncio
from pyvinty import (
    VintedClient, VintedDomain, 
    refresh_tokens, get_public_session,
    DataDomeHelper
)

async def setup_session():
    # 1. User agent REQUIRED
    user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36"
    
    # 2. Initialize client (proxy optional)
    client = VintedClient(user_agent=user_agent, domain=VintedDomain.FRANCE)
    
    # 2b. Proxy configuration (OPTIONAL - not required)
    # Without proxy (default behavior)
    # client = VintedClient(user_agent=user_agent, domain=VintedDomain.FRANCE)
    
    # With simple HTTP proxy
    # client = VintedClient(
    #     user_agent=user_agent, 
    #     domain=VintedDomain.FRANCE,
    #     proxy="http://proxy.example.com:8080"
    # )
    
    # With authenticated proxy
    # client = VintedClient(
    #     user_agent=user_agent,
    #     domain=VintedDomain.FRANCE,
    #     proxy="http://username:password@proxy.example.com:8080"
    # )
    
    # 3. Get fresh DataDome cookie
    datadome_cookie = await DataDomeHelper.get_fresh_cookie(target_url="https://www.vinted.fr", auto_close=True)
    if datadome_cookie:
        client.set_cookies({"datadome": datadome_cookie['value']})
    
    # 4. Get public session cookies
    public_cookies = await get_public_session(client.config)
    client.set_cookies(public_cookies)
    
    # 5. For authenticated operations, refresh tokens
    refresh_token = "your_refresh_token_here"
    updated_cookies = await refresh_tokens(client, refresh_token)
    client.set_cookies(updated_cookies)
    
    return client

# Usage
client = await setup_session()

# Examples of user information retrieval
from pyvinty import get_current_user_raw, get_current_user_info

# Method 1: Direct stateless functions
user_data = await get_current_user_raw(client)
user = await get_current_user_info(client)

# Method 2: Via services (backward compatibility)
user_data = await client.messaging.get_current_user()
user_info = await client.messaging.get_current_user_info()
```

### Using Pre-stored Tokens from Database

For applications that store authentication tokens in a database (like bot systems), pyVinty provides a direct injection method to use existing tokens without going through the full authentication flow.

#### Manual Token Injection Setup

```python
import asyncio
from pyvinty import VintedClient, VintedDomain

async def setup_with_stored_tokens():
    # 1. Initialize client with user agent
    user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36"
    client = VintedClient(user_agent=user_agent, domain=VintedDomain.FRANCE)
    
    # 2. Retrieve tokens from your database/storage
    # These would come from your database, config file, or storage system
    stored_access_token = "eyJraWQiOiJFNTdZZH..."      # JWT access token
    stored_refresh_token = "eyJraWQiOiJFNTdZZH..."     # JWT refresh token  
    stored_session_token = "bzZBVGNnc0xqSHhh..."      # Vinted session token
    stored_anon_id = "0f5ce556-9646-4f0a-a67d-..."    # Anonymous ID
    stored_datadome_cookie = "pxScq_xMVqeqUtWg..."    # DataDome cookie
    
    # 3. Inject all tokens at once using set_tokens()
    client.set_tokens(
        access_token=stored_access_token,
        refresh_token=stored_refresh_token,
        session_token=stored_session_token,
        anon_id=stored_anon_id,
        datadome_cookie=stored_datadome_cookie
    )
    
    return client

# Usage
client = await setup_with_stored_tokens()

# All services now work with pre-configured tokens
user_data = await client.payments.get_current_user()
conversations = await client.messaging.get_conversations()
```

#### Individual Token Injection

You can also inject tokens individually if needed:

```python
client = VintedClient(user_agent="...", domain=VintedDomain.FRANCE)

# Set tokens individually
client.set_cookies({"datadome": stored_datadome_cookie})
client.set_cookies({"_vinted_fr_session": stored_session_token})
client.set_cookies({"access_token": stored_access_token})
# ... etc
```

#### Database Integration Example

```python
import asyncio
from your_database import get_user_tokens  # Your database module
from pyvinty import VintedClient, VintedDomain

async def create_client_from_db(user_id: int):
    """Create a VintedClient using tokens stored in database"""
    
    # Retrieve tokens from database
    tokens = await get_user_tokens(user_id)
    
    # Initialize client
    client = VintedClient(
        user_agent=tokens["user_agent"], 
        domain=VintedDomain.FRANCE
    )
    
    # Inject stored tokens
    client.set_tokens(
        access_token=tokens["access_token"],
        refresh_token=tokens["refresh_token"],
        session_token=tokens["session_token"],
        anon_id=tokens["anon_id"],
        datadome_cookie=tokens["datadome_cookie"]
    )
    
    return client

# Usage in your application
user_client = await create_client_from_db(user_id=12345)
user_info = await user_client.payments.get_current_user()
```

#### Important Notes for Database Storage

- **Token Expiry**: Access tokens expire regularly (usually 2 hours), refresh tokens less frequently
- **Validation**: Always validate tokens before use by calling `get_current_user()` first
- **Refresh Logic**: Implement automatic refresh when access tokens expire
- **Security**: Store tokens securely in your database with proper encryption

#### Automatic Token Refresh with Database

```python
from pyvinty import VintedClient, VintedDomain, refresh_tokens
from your_database import get_user_tokens, save_user_tokens

async def create_auto_refresh_client(user_id: int):
    """Create a client with automatic token refresh and database persistence"""
    
    tokens = await get_user_tokens(user_id)
    client = VintedClient(user_agent=tokens["user_agent"], domain=VintedDomain.FRANCE)
    
    # Inject stored tokens
    client.set_tokens(
        access_token=tokens["access_token"],
        refresh_token=tokens["refresh_token"],
        session_token=tokens["session_token"],
        anon_id=tokens["anon_id"],
        datadome_cookie=tokens["datadome_cookie"]
    )
    
    # Test if tokens are still valid
    try:
        user_info = await client.payments.get_current_user()
        print(f"Tokens valid for user: {user_info.get('user', {}).get('login')}")
        return client
        
    except Exception as e:
        print(f"Access token expired, refreshing... Error: {e}")
        
        try:
            # Refresh tokens using stored refresh token
            updated_cookies = await refresh_tokens(client, tokens["refresh_token"])
            client.set_cookies(updated_cookies)
            
            # Extract new tokens and save to database
            new_tokens = {
                **tokens,  # Keep existing tokens
                "access_token": updated_cookies.get("access_token"),
                "refresh_token": updated_cookies.get("refresh_token", tokens["refresh_token"]),
                # Update other cookies if provided
            }
            
            await save_user_tokens(user_id, new_tokens)
            print("Tokens refreshed and saved to database")
            return client
            
        except Exception as refresh_error:
            print(f"Failed to refresh tokens: {refresh_error}")
            print("User needs to re-authenticate")
            raise

# Usage with error handling
try:
    client = await create_auto_refresh_client(user_id=12345)
    # Use client for operations...
except Exception:
    # Handle re-authentication needed
    pass
```

#### Error Handling for Stored Tokens

```python
async def safe_api_call(client, operation_func, *args, **kwargs):
    """Safely execute API calls with token validation"""
    
    try:
        return await operation_func(*args, **kwargs)
        
    except Exception as e:
        if "401" in str(e) or "403" in str(e):
            print("Authentication error - tokens may be expired")
            print("Please refresh tokens or re-authenticate")
        elif "datadome" in str(e).lower():
            print("DataDome challenge detected - need fresh DataDome cookie")
        else:
            print(f"API error: {e}")
        raise

# Usage
try:
    conversations = await safe_api_call(
        client, 
        client.messaging.get_conversations,
        page=1, per_page=10
    )
except Exception:
    # Handle authentication or other errors
    pass
```

### Cookie and Token Management

**User Vinted cookies and tokens are automatically managed by the pyVinty client.**

- No need to pass cookies as parameters to each method
- The client automatically stores all necessary cookies
- All methods automatically use the client's cookies
- Single configuration at startup with `client.set_cookies()`

```python
# INCORRECT - No need to pass cookies
# result = await client.messaging.get_conversations(cookies=my_cookies)

# CORRECT - Cookies are automatically used
result = await client.messaging.get_conversations()
```

### Types of Automatically Managed Cookies

1. **Public session cookies** - Via `get_public_session()`
2. **Authentication cookies** - Via `refresh_tokens()`  
3. **DataDome cookie** - Via `DataDomeHelper.get_fresh_cookie()`
4. **Navigation cookies** - Automatically generated by requests

**Once configured with `client.set_cookies()`, all method calls automatically use these cookies.**

### DataDome: Manual vs Automatic

**DataDomeHelper is primarily MANUAL:**

- Not automatic in services (except specific cases)
- User must call `DataDomeHelper.get_fresh_cookie()` manually
- Single exception: AuthService with `interactive_captcha=True`

```python
# DataDome is NOT managed automatically
result = await client.messaging.get_conversations()  # No auto DataDome

# User must configure DataDome manually
dd_cookie = await DataDomeHelper.get_fresh_cookie(target_url="https://www.vinted.fr")
if dd_cookie:
    client.set_cookies({"datadome": dd_cookie['value']})

# Exception: AuthService can manage DataDome automatically
auth_result = await client.auth.authenticate("user", "pass", interactive_captcha=True)
```

### Key Points
- **User agent required**: Requests will fail without a user agent
- **DataDome MANUAL**: Must be configured by user (except AuthService)
- **Public session**: Required for certain operations
- **Authenticated tokens**: Required for private user operations

---

## VintedClient Configuration Parameters

### Constructor Parameters

```python
client = VintedClient(
    user_agent="Mozilla/5.0 (...)",     # REQUIRED
    domain=VintedDomain.FRANCE,         # Optional (default: FRANCE)
    proxy="http://proxy:8080",          # Optional (default: None)
    timeout=30,                         # Optional (default: 30s)
    max_retries=3,                      # Optional (default: 3)
    rate_limit_delay=0.5,               # Optional (default: 0.5s)
    config=None                         # Or use VintedConfig
)
```

### Proxy Parameter (Optional)

**Proxies are NOT required** - without a configured proxy, requests go directly.

**Supported formats:**
```python
# No proxy (default)
client = VintedClient(user_agent="...")

# Simple HTTP proxy  
client = VintedClient(user_agent="...", proxy="http://proxy.example.com:8080")

# Proxy with authentication
client = VintedClient(user_agent="...", proxy="http://user:pass@proxy.example.com:8080")

# HTTPS proxy
client = VintedClient(user_agent="...", proxy="https://secure-proxy.example.com:3128")
```

**Behavior:**
- If `proxy=None` (default) → Direct requests
- If `proxy` configured → All requests go through proxy
- Compatible with all Vinted domains
- HTTP Basic authentication supported

---

## Utility Functions (tokens.py)

### `refresh_tokens(client, refresh_token)`

Refreshes user authentication tokens.

**Parameters:**
- `client` (VintedClient): Configured client instance
- `refresh_token` (str): Refresh token obtained during authentication

**Returns:**
- `Dict[str, str]`: Dictionary of updated cookies

**Usage:**
```python
updated_cookies = await refresh_tokens(client, "eyJraWQiOiJFNTdZZH...")
client.set_cookies(updated_cookies)
```

**Important:**
- Must be called regularly to maintain authentication
- Refresh token must be valid and not expired
- Returned cookies must be applied to client

### `get_public_session(config)`

Obtains an unauthenticated public session.

**Parameters:**
- `config` (VintedConfig): Client configuration (via client.config)

**Returns:**
- `Dict[str, str]`: Dictionary of public session cookies

**Usage:**
```python
public_cookies = await get_public_session(client.config)
client.set_cookies(public_cookies)
```

### `get_current_user_raw(client_or_config, http_client=None)`

Get current user information (raw API response).

**Parameters:**
- `client_or_config`: VintedClient instance OR VintedConfig
- `http_client` (HTTPClient, optional): Required if first parameter is VintedConfig

**Returns:**
- `Dict[str, Any]`: Raw API response with user data

**Usage with VintedClient:**
```python
# Method 1: With full client (recommended for end users)
from pyvinty import get_current_user_raw
user_data = await get_current_user_raw(client)
user = user_data.get("user", {})
user_id = user.get("id")
```

**Usage with Config + HTTPClient:**
```python
# Method 2: With separate components (used internally by services)
user_data = await get_current_user_raw(config, http_client)
```

### `get_current_user_info(client_or_config, http_client=None)`

Get current user information (standardized VintedUser object).

**Parameters:**
- `client_or_config`: VintedClient instance OR VintedConfig
- `http_client` (HTTPClient, optional): Required if first parameter is VintedConfig

**Returns:**
- `VintedUser`: Comprehensive user information object

**Usage with VintedClient:**
```python
# Method 1: With full client (recommended for end users)
from pyvinty import get_current_user_info
user = await get_current_user_info(client)
print(f"User: {user.login} (ID: {user.id})")
```

**Usage with Config + HTTPClient:**
```python
# Method 2: With separate components (used internally by services)
user = await get_current_user_info(config, http_client)
```

**Alternative via Services:**
```python
# Method 3: Via messaging service (backward compatibility)
user_data = await client.messaging.get_current_user()
user_info = await client.messaging.get_current_user_info()
```

### User Information Retrieval - Multiple Approaches

pyVinty offers multiple ways to retrieve user information, respecting both stateless and stateful patterns:

#### 1. Direct Stateless Functions (Recommended)
```python
from pyvinty import get_current_user_raw, get_current_user_info

# Raw API response
user_data = await get_current_user_raw(client)
user_id = user_data.get("user", {}).get("id")

# Structured VintedUser object  
user = await get_current_user_info(client)
print(f"User: {user.login} (ID: {user.id})")
```

**Benefits:**
- Explicit control over API calls
- No hidden caching or state
- Follows library's stateless philosophy
- Always fresh data from API

#### 2. Via Service Methods (Backward Compatibility)
```python
# Through messaging service
user_data = await client.messaging.get_current_user()
user_info = await client.messaging.get_current_user_info()

# Through payments service  
user_data = await client.payments.get_current_user()
```

**Benefits:**
- Familiar service-oriented approach
- Internally uses stateless functions
- Maintains existing code compatibility

#### 3. Internal Service Usage (Advanced)
```python
# Used internally by services - not recommended for end users
from pyvinty.utils.tokens import get_current_user_raw
user_data = await get_current_user_raw(config, http_client)
```

**When to use each approach:**
- **Method 1**: New code, explicit control needed
- **Method 2**: Existing code, service context
- **Method 3**: Internal service development only

---

## Services

### Automatic Service Authentication

**All services automatically use the client's configured authentication:**

- No cookie/token parameters needed in methods
- Single configuration with `client.set_cookies()`
- Transparent authentication management

```python
# Initial configuration (once)
client = VintedClient(user_agent="...", domain=VintedDomain.FRANCE)
updated_cookies = await refresh_tokens(client, refresh_token)
client.set_cookies(updated_cookies)

# All methods automatically use authentication
conversations = await client.messaging.get_conversations()
notifications = await client.notifications.get_favorite_notifications()
balance = await client.payments.get_user_balance(user_id)
# No need to pass cookies to each call!
```

---

## MessagingService

Access: `client.messaging`

### `get_conversations(page=1, per_page=20)`

Retrieves conversation list with preprocessed data.

**Parameters:**
- `page` (int, optional): Page number (default: 1)
- `per_page` (int, optional): Items per page (default: 20)

**Returns:**
```python
{
    "conversations": [
        {
            "id": int,                   # Conversation ID
            "login": str,                # Other user's login  
            "last_message": str,         # Last message content
            "updated_at": str,           # Last activity timestamp
            "item_count": int,           # Number of items
            "read": bool,                # Read status
            "unread": bool,              # Inverse of read
            "raw_data": dict             # Raw API data
        }
    ],
    "pagination": {
        "current_page": int,
        "total_pages": int,
        "total_count": int,
        "per_page": int
    },
    "raw_response": dict                 # Complete API response
}
```

**Usage:**
```python
conversations = await client.messaging.get_conversations(page=1, per_page=10)
for conv in conversations["conversations"]:
    print(f"Conversation with {conv['login']}: {conv['last_message']}")
```

### `get_conversation_details(conversation_id)`

Retrieves conversation details with preprocessed messages.

**Parameters:**
- `conversation_id` (int): Conversation ID

**Returns:**
```python
{
    "conversation": {
        "id": int,
        "other_user_login": str,         # Other user's login
        "item_count": int,
        "raw_data": dict
    },
    "messages": [
        {
            "id": int,                   # Message ID
            "body": str,                 # Message content
            "created_at": str,           # Creation timestamp
            "sender_login": str,         # Sender login
            "is_from_current_user": bool,# Message from current user
            "entity_type": str,          # Message type
            "raw_data": dict
        }
    ],
    "raw_response": dict
}
```

**Usage:**
```python
details = await client.messaging.get_conversation_details(12345)
for message in details["messages"]:
    sender = "Me" if message["is_from_current_user"] else message["sender_login"]
    print(f"{sender}: {message['body']}")
```

### `send_message(conversation_id, message, photo_ids=None)`

Sends a message in a conversation.

**Parameters:**
- `conversation_id` (int): Conversation ID
- `message` (str): Message content
- `photo_ids` (List[int], optional): Photo IDs to attach

**Returns:**
- `Dict[str, Any]`: API response of sent message

**Usage:**
```python
response = await client.messaging.send_message(12345, "Hello!")
# With photos
response = await client.messaging.send_message(12345, "Here are photos", photo_ids=[1001, 1002])
```

### `accept_offer(transaction_id, offer_request_id)`

Accepts a price offer.

**Parameters:**
- `transaction_id` (int): Transaction ID
- `offer_request_id` (int): Offer request ID

**Returns:**
- `Dict[str, Any]`: API response of acceptance

### `reject_offer(transaction_id, offer_request_id)`

Rejects a price offer.

**Parameters:**
- `transaction_id` (int): Transaction ID  
- `offer_request_id` (int): Offer request ID

**Returns:**
- `Dict[str, Any]`: API response of rejection

### `make_offer(conversation_id, amount)`

Makes a price offer on an item.

**Parameters:**
- `conversation_id` (int): Conversation ID
- `amount` (float): Offer amount

**Returns:**
- `Dict[str, Any]`: API response of offer

### `find_pending_offers(conversation_details)`

Finds pending offers in a conversation.

**Parameters:**
- `conversation_details` (dict): Conversation details (from get_conversation_details())

**Returns:**
- `List[Dict[str, Any]]`: List of pending offers

### `get_conversations_list(page=1, per_page=10, current_user_id=None)`

**ALTERNATIVE METHOD** - Returns structured ConversationsResponse object.

**Parameters:**
- `page` (int): Page number
- `per_page` (int): Items per page
- `current_user_id` (int, optional): Current user ID (auto-detected if None)

**Returns:**
- `ConversationsResponse`: Object with structured conversations and pagination

### `get_conversation_messages_list(conversation_id, current_user_id=None)`

**ALTERNATIVE METHOD** - Returns structured MessagesResponse object.

**Parameters:**
- `conversation_id` (int): Conversation ID
- `current_user_id` (int, optional): Current user ID (auto-detected if None)

**Returns:**
- `MessagesResponse`: Object with structured messages and metadata

---

## PaymentService

Access: `client.payments`

### `create_transaction(item_id, seller_id)`

Creates a purchase transaction with preprocessed data.

**Parameters:**
- `item_id` (int): Item ID to purchase
- `seller_id` (int): Seller ID

**Returns:**
```python
{
    "transaction": {
        "item_id": int,              # Item ID
        "item_title": str,           # Item title
        "item_url": str,             # Item URL
        "conversation_id": int       # Created conversation ID
    },
    "pricing": {
        "item_price": {
            "amount": float,         # Item price
            "currency": str          # Currency (EUR, USD...)
        },
        "service_fee": {
            "amount": float,         # Service fee
            "currency": str
        },
        "total": {
            "amount": float,         # Total price
            "currency": str
        }
    },
    "shipping_options": {
        "pickup_points": [],         # Available pickup points
        "home_delivery": [],         # Home delivery options
        "selected_option": dict      # Selected option
    },
    "payment_methods": {
        "available_cards": [],       # Available cards
        "balance_available": bool,   # Vinted balance available
        "selected_payment_method": dict
    },
    # Compatibility fields
    "transaction_id": int,
    "checkout_id": str,
    "shipping_id": str,
    "conversation_id": int,
    "photo_url": str,
    "title": str
}
```

**Usage:**
```python
transaction = await client.payments.create_transaction(item_id=123456, seller_id=789)
print(f"Item: {transaction['transaction']['item_title']}")
print(f"Price: {transaction['pricing']['total']['amount']} {transaction['pricing']['total']['currency']}")
```

### `select_delivery_method(transaction_data, shipping_method, latitude=None, longitude=None)`

Selects delivery method with preprocessed data.

**Parameters:**
- `transaction_data` (dict): Transaction data from create_transaction()
- `shipping_method` (int): 1 for pickup point, 2 for home delivery
- `latitude` (float, optional): Latitude (required for pickup point)
- `longitude` (float, optional): Longitude (required for pickup point)

**Returns:**
```python
{
    "delivery_info": str,           # Description of chosen delivery
    "checksum": str,                # Checksum for next step
    "pricing": {                    # Updated pricing with delivery
        "item_price": {...},
        "service_fee": {...},
        "total": {...}
    },
    "shipping_options": {...},      # Updated shipping options
    "payment_methods": {...},       # Updated payment methods
    "checkout_id": str,
    "shipping_method": int,
    "price": str                    # Legacy format "amount currency"
}
```

### `prepare_payment(checkout_id, payment_method, card_id=None)`

Prepares payment method.

**Parameters:**
- `checkout_id` (str): Checkout ID
- `payment_method` (int): 1 for card, 2 for Vinted balance
- `card_id` (int, optional): Card ID (required if payment_method=1)

**Returns:**
- `str`: New checksum to finalize payment

### `complete_payment(checkout_id, checksum)`

Finalizes payment with preprocessed data.

**Parameters:**
- `checkout_id` (str): Checkout ID
- `checksum` (str): Checksum from prepare_payment()

**Returns:**
```python
{
    "status": str,                   # "success", "pending", "failed"
    "transaction_id": str,           # Transaction ID
    "redirect_url": str,             # 3D Secure URL if needed
    "requires_action": bool          # True if user action required
}
```

**Usage:**
```python
result = await client.payments.complete_payment(checkout_id, checksum)
if result["status"] == "success":
    print("Payment successful!")
elif result["requires_action"]:
    print(f"3D Secure required: {result['redirect_url']}")
```

### `get_coordinates_from_address(address)`

Geocodes an address to obtain GPS coordinates.

**Parameters:**
- `address` (Dict[str, Any]): Address dictionary with the following fields:
  - `line1` (str): First address line (number + street)
  - `postal_code` (str): Postal code
  - `city` (str): City
  - `country_iso_code` (str): Country code (FR, BE, etc.)

**Returns:**
```python
# Success
{
    "latitude": float,               # GPS latitude
    "longitude": float               # GPS longitude
}

# Failure
None
```

**Usage:**
```python
address = {
    "line1": "123 rue de la Paix",
    "postal_code": "75001",
    "city": "Paris", 
    "country_iso_code": "FR"
}

coords = await client.payments.get_coordinates_from_address(address)
if coords:
    print(f"Coordinates: {coords['latitude']}, {coords['longitude']}")
    # Use for select_delivery_method with shipping_method=1
    delivery = await client.payments.select_delivery_method(
        transaction_data, 
        shipping_method=1, 
        latitude=coords['latitude'], 
        longitude=coords['longitude']
    )
else:
    print("Geocoding failed - address not found")
```

**Important information:**
- 15-second timeout per request
- Returns `None` if address not found or error occurs
- Primarily used to find nearby pickup points
- Coordinates can then be used with `select_delivery_method()`

### Other PaymentService Methods

- `get_user_balance(user_id)`: Retrieves user's Vinted balance
- `get_current_user()`: Get current user information (uses stateless tokens)

### Service Architecture Note

**All services now use stateless token functions internally:**

- `MessagingService.get_current_user()` → calls `get_current_user_raw(config, http_client)`
- `PaymentService.get_current_user()` → calls `get_current_user_raw(config, http_client)`
- No hidden caching or state management
- Every call is fresh from the API
- User maintains full control over when API requests are made

This ensures the **stateless philosophy** is maintained throughout the library while providing backward compatibility.

---

## NotificationService

Access: `client.notifications`

### `get_notifications(page=1, per_page=20)`

Retrieves notifications with preprocessed data.

**Parameters:**
- `page` (int, optional): Page number (default: 1)
- `per_page` (int, optional): Items per page (default: 20)

**Returns:**
```python
{
    "notifications": [
        {
            "id": int,                   # Notification ID
            "entry_type": int,           # Notification type
            "timestamp": str,            # Date/time
            "read_status": bool,         # Read status
            "body": str,                 # Notification content
            "photo_url": str,            # Associated photo URL
            "raw_data": dict             # Raw API data
        }
    ],
    "pagination": {
        "current_page": int,
        "per_page": int, 
        "total_count": int,
        "total_pages": int,
        "has_next_page": bool
    },
    "raw_response": dict
}
```

### `get_favorite_notifications(page=1, per_page=20)`

Retrieves only favorite notifications with automatic parsing.

**Parameters:**
- `page` (int, optional): Page number (default: 1) 
- `per_page` (int, optional): Items per page (default: 20)

**Returns:**
```python
[
    {
        "timestamp": str,            # Date/time
        "username": str,             # User who favorited
        "item_name": str,            # Item name
        "photo_url": str,            # Item photo URL
        "body": str,                 # Complete message
        "read_status": bool,         # Read status
        "parsed_successfully": bool, # True if parsing succeeded
        "raw_data": dict             # Raw data
    }
]
```

**Usage:**
```python
favorites = await client.notifications.get_favorite_notifications()
for fav in favorites:
    print(f"{fav['username']} liked '{fav['item_name']}'")
```

### `get_notification_types(page=1, per_page=20)`

Groups notifications by type.

**Returns:**
```python
{
    20: [notifications],             # Type 20 = favorites
    15: [notifications],             # Type 15 = messages
    # ... other types
}
```

### `get_unread_count()`

Counts unread notifications.

**Returns:**
```python
{
    "unread_count": int,             # Unread count
    "total_count": int,              # Total notifications
    "checked_count": int,            # Checked notifications
    "pagination": {...}              # Pagination info
}
```

### Other NotificationService Methods

- `mark_notification_as_read(notification_id)`: Mark as read
- `mark_all_notifications_as_read()`: Mark all as read

---

## AuthService

Access: `client.auth`

### `authenticate(username, password, interactive_captcha=False)`

Authenticates user with username and password.

**Parameters:**
- `username` (str): Vinted username/email
- `password` (str): User password
- `interactive_captcha` (bool): Automatic DataDome challenge handling

**Returns:**
```python
{
    "status": "success_no_2fa" | "requires_2fa" | "failed",
    "response_body": {...},  # Complete response data
    "cookies": {...},        # Session cookies
    "message": "...",        # Error message if failed
    "control_code": "...",   # Control code if 2FA required
    "raw_body": {...}        # Raw response on error
}
```

**Usage:**
```python
result = await client.auth.authenticate("your_username", "your_password")

# With interactive captcha handling
result = await client.auth.authenticate("user", "pass", interactive_captcha=True)
```

### `complete_2fa(username, control_code, verification_code, initial_cookies)`

Completes two-factor authentication.

**Parameters:**
- `username` (str): Vinted username/email
- `control_code` (str): Control code from initial authentication
- `verification_code` (str): User's 2FA verification code
- `initial_cookies` (Dict[str, str]): Cookies from initial authentication

**Returns:**
```python
{
    "status": "success" | "failed",
    "response_body": {...},  # Complete response data
    "cookies": {...},        # Final session cookies
    "message": "...",        # Error message if failed
    "raw_body": {...}        # Raw response on error
}
```

**Usage:**
```python
# After receiving "requires_2fa" from initial authentication
result = await client.auth.complete_2fa(
    username="your_username",
    control_code=auth_result["control_code"],
    verification_code="123456",  # Code received via SMS/email
    initial_cookies=auth_result["cookies"]
)
```

**Important information:**
- Authentication follows a 2-step process if 2FA is enabled
- Returned cookies must be used for subsequent sessions
- Automatic DataDome challenge handling with `interactive_captcha=True`
- All errors are logged with complete details
- Methods use data preprocessing architecture

### `login(email, password)`

Alternative authentication method by email/password.

**Parameters:**
- `email` (str): Email address
- `password` (str): Password

**Returns:**
- `Dict[str, Any]`: Authentication response with tokens

### `verify_2fa(code, login_response)`

Verifies 2FA code.

**Parameters:**
- `code` (str): Received 2FA code
- `login_response` (dict): Response from login()

**Returns:**
- `Dict[str, Any]`: Finalized authentication tokens

---

## DataDomeHelper

### Usage: MANUAL by Default

**DataDomeHelper is NOT automatic in services - user must use it manually.**

### `get_fresh_cookie(target_url="https://www.vinted.fr", auto_close=True)`

Generates a fresh DataDome cookie to avoid anti-bot blocking.

**Parameters:**
- `target_url` (str, optional): Target Vinted URL (default: "https://www.vinted.fr")
- `auto_close` (bool, optional): Automatically close browser (default: True)

**Returns:**
```python
{
    "name": "datadome",
    "value": str,                    # DataDome cookie value
    "domain": str,                   # Domain (.vinted.fr)
    "path": str                      # Path (/)
}
# Or None if failed
```

**Typical usage:**
```python
# 1. Manual retrieval before sensitive operations
dd_cookie = await DataDomeHelper.get_fresh_cookie(target_url="https://www.vinted.fr")
if dd_cookie:
    client.set_cookies({"datadome": dd_cookie['value']})
    print(f"DataDome configured: {dd_cookie['value'][:30]}...")

# 2. Services then automatically use this cookie
conversations = await client.messaging.get_conversations()  # Uses configured DataDome
payments = await client.payments.get_user_balance(user_id)  # Same
```

### Automatic Usage Case

**Single exception: AuthService with `interactive_captcha=True`**

```python
# DataDome managed automatically by AuthService if challenge detected
auth_result = await client.auth.authenticate(
    username="user", 
    password="pass", 
    interactive_captcha=True
)
```

### Recommendations

1. **Configure DataDome** before sensitive operations (auth, payments, messages...)
2. **Reuse the same cookie** for the entire session
3. **Renew periodically** if frequent blocking occurs
4. **Monitor logs** to detect DataDome challenges

---

## Important Notes

### Error Handling

All methods can raise exceptions:
- `APIError`: Vinted API error
- `AuthenticationError`: Authentication problem
- `VintedError`: General pyVinty error

### Async Management

All methods are asynchronous and must be called with `await`:
```python
async def main():
    client = VintedClient(...)
    result = await client.messaging.get_conversations()
    await client.close()

asyncio.run(main())
```

### Best Practices Summary

#### User Information Retrieval - Recommended Patterns

**For New Code (Recommended):**
```python
from pyvinty import get_current_user_info, get_current_user_raw

# Structured data with full type hints
user = await get_current_user_info(client)
print(f"User: {user.login} (ID: {user.id})")

# Raw API data when you need flexibility  
user_data = await get_current_user_raw(client)
```

**For Existing Code (Backward Compatible):**
```python
# Via services - uses stateless functions internally
user_data = await client.messaging.get_current_user()
user_info = await client.messaging.get_current_user_info()
```

#### Database Integration Patterns

**For Applications with Token Storage:**
```python
# Direct token injection from database
client = VintedClient(user_agent="...", domain=VintedDomain.FRANCE)
client.set_tokens(
    access_token=db_tokens["access_token"],
    refresh_token=db_tokens["refresh_token"],
    session_token=db_tokens["session_token"],
    anon_id=db_tokens["anon_id"],
    datadome_cookie=db_tokens["datadome_cookie"]
)

# Validate and use immediately
user_data = await client.payments.get_current_user()
```

**For Bot Systems with Multiple Users:**
```python
async def get_user_client(user_id: int) -> VintedClient:
    tokens = await database.get_user_tokens(user_id)
    client = VintedClient(user_agent=tokens["user_agent"])
    client.set_tokens(**tokens)
    return client

# Usage per user
user_client = await get_user_client(12345)
conversations = await user_client.messaging.get_conversations()
```

#### Architecture Benefits
- **Stateless**: No hidden caching, full user control
- **Explicit**: Every API call is intentional and visible
- **Fresh Data**: Always current information from API
- **Backward Compatible**: Existing service methods still work
- **Flexible**: Use full client or separate components
- **Testable**: Easy to mock and test individual functions
- **Database Friendly**: Direct token injection for stored credentials

#### Token Management Best Practices
1. **Always validate tokens** before API operations with `get_current_user()`
2. **Implement refresh logic** for expired access tokens
3. **Store securely** in database with proper encryption
4. **Handle errors gracefully** with 401/403 detection
5. **Update database** after successful token refresh
6. **Use set_tokens()** for bulk token injection from storage

This design ensures you maintain complete control over API requests while providing multiple usage patterns for different scenarios, including seamless database integration for bot systems and multi-user applications.

The `raw_data` and `raw_response` fields are available in all responses for advanced use cases.
