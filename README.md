# Apollo SDK API Reference

**SDK Version:** 0.1.0
**Platform:** Unity (iOS & Android)

---

## Table of Contents

1. [Overview](#overview)
2. [Setup & Initialization](#setup--initialization)
3. [BFGUnitySDK](#bfgunitysdksdk-entry-point) — SDK entry point
4. [Adapter Interfaces](#adapter-interfaces)
   - [IAuthenticationAdapter](#iauthenticationadapter)
5. [Listener Interfaces](#listener-interfaces)
   - [IAuthenticationListener](#iauthenticationlistener)
   - [ITelemetryListener](#itelemetrylistener)
6. [Data Types](#data-types)
   - [CustomEventData](#customeventdata)
   - [PurchaseSuccessData](#purchasesuccessdata)
   - [PurchaseFailureData](#purchasefailuredata)
7. [Enums](#enums)
   - [ATTStatus](#attstatus)
   - [IdentityProviderName](#identityprovidername)
   - [PurchaseErrorReason](#purchaseerrorreason)
   - [PurchasePhase](#purchasephase)

---

## Overview

This reference covers the following Apollo SDK subsystems:

| Subsystem | Description |
|---|---|
| **Initialization** | Registers adapters/listeners, then calls `Initialize()` |
| **Authentication** | Implements `IAuthenticationAdapter` and `IAuthenticationListener` |
| **Telemetry** | Sends custom game-lifecycle events and purchase events; implements `ITelemetryListener` |
| **Purchasing** | Reports purchase success and failure via `BFGUnitySDK` telemetry helpers |
| **Consent / Privacy** | Applies ATT status (iOS) and third-party tracking consent (GDPR) |

This reference does not cover the Attribution adapter, Purchasing adapter, or Consent management types (`ConsentList`, `ConsentStatus`, `GDPRStatus`).

---

## Setup & Initialization

Initialize the SDK as follows:

```csharp
// 1. Register the authentication adapter implementation
BFGUnitySDK.RegisterAdapter<AuthenticationAdapter>();

// 2. Register listener implementations
BFGUnitySDK.RegisterListener<BasicAuthListener>();
BFGUnitySDK.RegisterListener<BasicTelemetryListener>();

// 3. Initialize — must follow all registrations
BFGUnitySDK.Initialize();

// 4. Supply the attribution ID (e.g. AppsFlyer device ID) — included in all GTS events as 'afid'
BFGUnitySDK.SetAttributionID(ATTRIBUTION_ID);
```

All `RegisterAdapter<T>` and `RegisterListener<T>` calls must occur before `Initialize()`.

---

## BFGUnitySDK — SDK Entry Point

```csharp
public static class BFGUnitySDK
```

No namespace; available globally.

---

### Initialization

#### `Initialize`

```csharp
public static void Initialize()
```

Starts the SDK. Wires all registered adapters and listeners and begins lifecycle tracking. Call once per session, after all registrations.

---

### Adapter & Listener Registration

#### `RegisterAdapter<T>`

```csharp
public static void RegisterAdapter<T>() where T : IAdapter, new()
```

Registers a concrete adapter. `T` must implement `IAdapter` and have a parameterless constructor. Call before `Initialize()`.

#### `RegisterListener<T>`

```csharp
public static void RegisterListener<T>() where T : IListener, new()
```

Registers a concrete listener. `T` must implement `IListener` and have a parameterless constructor. Call before `Initialize()`.

---

### Attribution ID

#### `SetAttributionID`

```csharp
public static void SetAttributionID(string attributionID)
```

Stores the attribution provider's device ID (e.g., the AppsFlyer ID). Once set, this value is included in all outbound GTS telemetry events as the `afid` field. The value is persisted across sessions via `PlayerPrefs`.

| Parameter | Type | Description |
|---|---|---|
| `attributionID` | `string` | The attribution provider's device identifier. |

> Call this as soon as the attribution ID is available — typically immediately after `Initialize()`, once your attribution adapter has obtained the ID from its underlying SDK. Any telemetry events dispatched before `SetAttributionID` is called will have an empty `afid`.

---

### Consent & Privacy

#### `ApplyAttConsentStatus`

```csharp
public static void ApplyAttConsentStatus(ATTStatus authorized)
```

Stores the iOS App Tracking Transparency (ATT) consent result. Call this after receiving the OS callback from the ATT prompt. The SDK reads this value when building telemetry payloads to determine whether the IDFA may be included.

| Parameter | Type | Description |
|---|---|---|
| `authorized` | [`ATTStatus`](#attstatus) | The authorization status returned by the OS after the ATT prompt. |

```csharp
// Example: native bridge delivers status as a string integer
if (int.TryParse(statusCode, out int value))
{
    ATTStatus status = (ATTStatus)value;
    BFGUnitySDK.ApplyAttConsentStatus(status);
}
```

> iOS-specific. No-op on Android.

#### `ApplyThirdPartyTrackingConsentStatus`

```csharp
public static void ApplyThirdPartyTrackingConsentStatus(bool authorized = true)
```

Stores the user's consent for third-party tracking. This value is written into all outbound GTS telemetry events as the `tpte` (third-party tracking enabled) field. Call when the GDPR/consent UI is dismissed.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `authorized` | `bool` | `true` | `true` if the user granted consent; `false` if denied or withdrawn. |

---

### Telemetry

#### `SendCustomEvent<T>`

```csharp
public static void SendCustomEvent<T>(string eventName, T customEventData)
    where T : BFG.Apollo.Telemetry.DataObjects.CustomEvent.CustomEventData, new()
```

Sends a custom-typed telemetry event to GTS. Define your payload by subclassing [`CustomEventData`](#customeventdata).

| Parameter | Type | Description |
|---|---|---|
| `eventName` | `string` | Name identifying the event type. |
| `customEventData` | `T` | An instance of your `CustomEventData` subclass carrying the event payload. |

| Type parameter | Constraint |
|---|---|
| `T` | Subclass of `CustomEventData`, `new()` |

#### `SetAppUserId`

```csharp
public static void SetAppUserId(string appUserId)
```

Sets a game-level user identifier attached to all subsequent telemetry events. Persisted across sessions.

| Parameter | Type | Description |
|---|---|---|
| `appUserId` | `string` | Your game's identifier for this user. |

#### `GetAppUserId`

```csharp
public static string GetAppUserId()
```

Returns the currently stored application user ID.

**Returns:** `string` — The app user ID, or an empty string if none has been set.

---

### Purchasing Telemetry

#### `SendPurchasingSuccessEvent`

```csharp
public static void SendPurchasingSuccessEvent(PurchaseSuccessData customEventData)
```

Reports a completed purchase (including restores) to the telemetry system.

| Parameter | Type | Description |
|---|---|---|
| `customEventData` | [`PurchaseSuccessData`](#purchasesuccessdata) | Transaction details. |

#### `SendPurchasingFailureEvent`

```csharp
public static void SendPurchasingFailureEvent(PurchaseFailureData customEventData)
```

Reports a failed purchase to the telemetry system.

| Parameter | Type | Description |
|---|---|---|
| `customEventData` | [`PurchaseFailureData`](#purchasefailuredata) | Failure details. |

---

## Adapter Interfaces

Adapters are interfaces your game implements to connect third-party services into the Apollo SDK.

---

### IAuthenticationAdapter

```csharp
public interface IAuthenticationAdapter : IAdapter
```

**Namespace:** `BFG.Apollo.Auth`
**Implemented by:** `AuthenticationAdapter`

This adapter connects a Firebase-backed authentication service to the Apollo SDK. It exposes a singleton via `AuthenticationAdapter.getInstance()` so other systems can read and set auth state directly.

| Member | Signature | Implementation Notes |
|---|---|---|
| `ProviderName` | `string ProviderName { get; }` | Returns `"FirebaseAuthentication"` |
| `UserID` | `string UserID { get; }` | Reads from `PlayerPrefs`; defaults to a constant default ID if not set |
| `Initialize` | `void Initialize(IAuthenticationListener authenticationListener)` | Stores the singleton reference; immediately calls `authenticationListener.OnAuthenticationInitialized()` |
| `Start` | `void Start()` | Empty — no deferred startup work needed |
| `IsAuthenticated` | `bool IsAuthenticated()` | Returns `authenticatedState`, which is backed by `PlayerPrefs` and survives restarts |
| `IsAnonymouslyAuthenticated` | `bool IsAnonymouslyAuthenticated()` | Returns `false` — anonymous auth is not used |
| `Login` | `void Login(IdentityProviderName identityProviderName)` | Empty — logins are not initiated through the SDK in this implementation |
| `Logout` | `void Logout()` | Empty — logouts are not initiated through the SDK in this implementation |

> **Note on persistence:** Both `UserID` and `IsAuthenticated()` read from `PlayerPrefs`, so their values survive app restarts without requiring a new login. They can be updated at runtime through `SetUserID(string)` and `SetAuthenticatedState(bool)` (non-interface helpers on `AuthenticationAdapter`).

---

## Listener Interfaces

Listeners receive callbacks from the SDK. Your implementations are registered before `Initialize()`.

---

### IAuthenticationListener

```csharp
public interface IAuthenticationListener : IListener
```

**Namespace:** `BFG.Apollo.Auth`
**Implemented by:** `BasicAuthListener`

| Method | Signature | Behavior |
|---|---|---|
| `OnAuthenticationInitialized` | `void OnAuthenticationInitialized()` | Logs `"Authentication initialization successful"` |
| `OnAuthenticationInitializeFailed` | `void OnAuthenticationInitializeFailed(string failureReason)` | Logs the failure reason |
| `OnLoginSuccess` | `void OnLoginSuccess()` | Logs `"Login Success"` |
| `OnLoginFailed` | `void OnLoginFailed(string failureReason)` | Logs a warning with the failure reason |
| `OnLogoutSuccess` | `void OnLogoutSuccess()` | Logs `"Logout Success"` |
| `OnLogoutFailed` | `void OnLogoutFailed(string failureReason)` | Logs the failure reason |

---

### ITelemetryListener

```csharp
public interface ITelemetryListener : IListener
```

**Namespace:** `BFG.Apollo.Telemetry`
**Implemented by:** `BasicTelemetryListener`

| Method | Signature | Behavior |
|---|---|---|
| `OnTelemetrySent` | `void OnTelemetrySent(bool success, string message)` | Logs `message` to the Unity console regardless of `success` |

---

## Data Types

---

### CustomEventData

```csharp
[Serializable]
public class CustomEventData
```

**Namespace:** `BFG.Apollo.Telemetry.DataObjects.CustomEvent`

Base class for all custom telemetry event payloads. Subclass this to add game-specific fields.

| Field | Type | Description |
|---|---|---|
| `eventName` | `string` | Name of the event. Set this on your subclass instance before passing to `SendCustomEvent`. |

**Example subclasses:**

```csharp
// Game lifecycle event
class GameLifecycleEvent : CustomEventData
{
    public string HeartsAvailable;  // Current lives count as a string
}

// Inventory change event
class InventoryChangeEvent : CustomEventData
{
    public string Item;    // Item name (e.g., "Sword")
    public string Status;  // Action (e.g., "Drop")
}
```

Type constraint for `SendCustomEvent<T>`: `T` must subclass `CustomEventData` and have a parameterless constructor.

---

### PurchaseSuccessData

```csharp
public class PurchaseSuccessData
```

**Namespace:** `BFG.Apollo.Purchasing`

Carries the details of a completed purchase. Pass a populated instance to `BFGUnitySDK.SendPurchasingSuccessEvent`.

| Field | Type | Description |
|---|---|---|
| `productId` | `string` | Store-specific product identifier. |
| `transactionID` | `string` | Unique transaction identifier from the store. |
| `transactionTimestamp` | `long` | Unix timestamp (seconds UTC) of the transaction. Use `0` for restored purchases when the original timestamp is unavailable. |
| `price` | `string` | Purchase price as a decimal string (e.g., `"2.99"`). Use `localizedPrice.ToString(CultureInfo.InvariantCulture)` for new purchases; use `localizedPriceString` for restored purchases. |
| `currency` | `string` | ISO 4217 currency code (e.g., `"USD"`). |
| `receipt` | `string` | Raw receipt data from the store. |
| `uniqueReceiptID` | `string` | A unique identifier derived from the receipt (e.g., a hash). Pass `null` for restored purchases if not available. |
| `restore` | `bool` | `true` if this is a restore; `false` for a new purchase. |
| `signature` | `string` | Purchase signature (Android). Pass `null` on iOS or when unavailable. |
| `description` | `string` | Localized product description from the store. |

**Example — new purchase:**

```csharp
BFGUnitySDK.SendPurchasingSuccessEvent(new PurchaseSuccessData
{
    currency             = cartItem.Product.metadata.isoCurrencyCode,
    description          = cartItem.Product.metadata.localizedDescription,
    price                = cartItem.Product.metadata.localizedPrice.ToString(CultureInfo.InvariantCulture),
    productId            = cartItem.Product.definition.storeSpecificId,
    receipt              = obj.Info.Receipt,
    uniqueReceiptID      = obj.Info.Receipt.GetHashCode().ToString(),
    restore              = false,
    signature            = null,
    transactionID        = obj.Info.TransactionID,
    transactionTimestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds()
});
```

**Example — restore:**

```csharp
BFGUnitySDK.SendPurchasingSuccessEvent(new PurchaseSuccessData
{
    currency             = cartItem.Product.metadata.isoCurrencyCode,
    description          = cartItem.Product.metadata.localizedDescription,
    price                = cartItem.Product.metadata.localizedPriceString,
    productId            = cartItem.Product.definition.storeSpecificId,
    receipt              = confirmedOrder.Info.Receipt,
    uniqueReceiptID      = null,
    restore              = true,
    signature            = null,
    transactionID        = confirmedOrder.Info.TransactionID,
    transactionTimestamp = 0
});
```

---

### PurchaseFailureData

```csharp
public class PurchaseFailureData
```

**Namespace:** `BFG.Apollo.Purchasing`

Carries the details of a failed purchase. Pass a populated instance to `BFGUnitySDK.SendPurchasingFailureEvent`.

| Field | Type | Description |
|---|---|---|
| `productId` | `string` | Store-specific product identifier of the attempted purchase. |
| `errorCode` | `int` | Raw numeric error code from the store or purchasing library. |
| `errorMessage` | `string` | Human-readable description of the error. |
| `errorReason` | [`PurchaseErrorReason`](#purchaseerrorreason) | Normalized failure reason. When using Unity IAP, cast the `FailureReason` integer directly: `(PurchaseErrorReason)(int)obj.FailureReason`. |
| `purchasePhase` | [`PurchasePhase`](#purchasephase) | The pipeline phase at which the failure occurred. |

**Example:**

```csharp
BFGUnitySDK.SendPurchasingFailureEvent(new PurchaseFailureData
{
    errorCode     = (int)obj.FailureReason,
    errorMessage  = obj.Details,
    productId     = cartItem.Product.definition.storeSpecificId,
    errorReason   = (PurchaseErrorReason)(int)obj.FailureReason,
    purchasePhase = PurchasePhase.Unknown
});
```

---

## Enums

---

### ATTStatus

```csharp
public enum ATTStatus
```

**Namespace:** `BFG.Apollo.Policy`

Mirrors iOS `ATTrackingManager.AuthorizationStatus`. Values are mapped 1:1 to the native integer values delivered by the iOS ATT callback.

| Value | Integer | Description |
|---|---|---|
| `NotDetermined` | `0` | The user has not yet been prompted. |
| `Restricted` | `1` | Access is restricted by device policy or parental controls. |
| `Denied` | `2` | The user denied permission. |
| `Authorized` | `3` | The user granted permission; the IDFA may be read. |

When the native iOS bridge delivers the status as a string integer, cast it as follows:

```csharp
ATTStatus status = (ATTStatus)value;
BFGUnitySDK.ApplyAttConsentStatus(status);
```

> `ATTStatus.Unknown` is an alias for `NotDetermined` (both equal `0`) and is used only internally by the SDK. Always use `NotDetermined` in app code.

---

### IdentityProviderName

```csharp
public enum IdentityProviderName
```

**Namespace:** `BFG.Apollo.Auth`

Used as the parameter type for `IAuthenticationAdapter.Login()`. If your implementation does not initiate identity-provider logins through the SDK, `Login()` may be a no-op.

| Value | Description |
|---|---|
| `Mock` | Test/mock provider. Do not ship. |
| `Apple` | Sign in with Apple. |
| `Google` | Sign in with Google. |
| `Facebook` | Sign in with Facebook. |

---

### PurchaseErrorReason

```csharp
public enum PurchaseErrorReason
```

**Namespace:** `BFG.Apollo.Purchasing`

Normalized reason for a purchase failure, set on `PurchaseFailureData.errorReason`. When using Unity IAP, cast the `FailureReason` integer directly: `(PurchaseErrorReason)(int)failureReason`.

| Value | Description |
|---|---|
| `PurchasingUnavailable` | Purchasing system unavailable on this device. |
| `ExistingPurchasePending` | A prior transaction for this product is still open. |
| `ProductUnavailable` | The product is not available in the store. |
| `SignatureInvalid` | Receipt signature validation failed. |
| `UserCancelled` | The user cancelled the purchase flow. |
| `PaymentDeclined` | The payment method was declined. |
| `DuplicateTransaction` | This transaction was already processed. |
| `NoConnection` | No network connectivity. |
| `Unknown` | Failure reason could not be determined. |

---

### PurchasePhase

```csharp
public enum PurchasePhase
```

**Namespace:** `BFG.Apollo.Purchasing`

Identifies the pipeline stage where a purchase failure occurred, set on `PurchaseFailureData.purchasePhase`. When using Unity IAP, which does not map directly to these phases, use `PurchasePhase.Unknown`.

| Value | Integer | Description |
|---|---|---|
| `Unknown` | `0` | Phase undetermined. Use when the failure source does not map to a specific phase. |
| `StartPhase` | `1` | Failure at purchase initiation. |
| `PreBuyPhase` | `2` | Failure during pre-purchase validation. |
| `HealthCheckPhase` | `3` | Reserved; not currently used. |
| `StoreResponsePhase` | `4` | Failure processing the store's response. |
| `ClientVerificationPhase` | `5` | Failure during client-side receipt verification. |
| `ServerVerificationPhase` | `6` | Failure during server-side receipt verification. |
