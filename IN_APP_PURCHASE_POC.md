# In-App Purchase Implementation POC for VPN App (Jetpack Compose)

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Google Play Billing Setup](#google-play-billing-setup)
3. [Android Implementation](#android-implementation)
4. [Backend Verification](#backend-verification)
5. [Security Best Practices](#security-best-practices)
6. [Testing Strategy](#testing-strategy)

---

## Architecture Overview

```
┌─────────────────┐
│   Android App   │
│ (Jetpack Compose)│
└────────┬────────┘
         │
         │ 1. Purchase Flow
         ▼
┌─────────────────┐
│ Google Play     │
│ Billing Library │
└────────┬────────┘
         │
         │ 2. Purchase Token
         ▼
┌─────────────────┐
│  Your Backend   │
│   (REST API)    │
└────────┬────────┘
         │
         │ 3. Verify with Google
         ▼
┌─────────────────┐
│ Google Play     │
│ Developer API   │
└─────────────────┘
```

---

## Google Play Billing Setup

### 1. Add Dependencies (build.gradle.kts)

**AndroidManifest.xml - Add Billing Permission:**
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- Required permission for Google Play Billing -->
    <uses-permission android:name="com.android.vending.BILLING" />

    <application>
        ...
    </application>
</manifest>
```

**build.gradle.kts:**
```kotlin
dependencies {
    // Billing Library (Updated to latest version)
    implementation("com.android.billingclient:billing-ktx:6.2.1")

    // Jetpack Compose
    implementation("androidx.compose.ui:ui:1.6.0")
    implementation("androidx.compose.material3:material3:1.2.0")

    // Lifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // Retrofit for backend communication
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")

    // Logging
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
}
```

---

## Android Implementation

### 2. Billing Manager (Core Logic)

```kotlin
// BillingManager.kt
package com.vpnapp.billing

import android.app.Activity
import android.content.Context
import com.android.billingclient.api.*
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

class BillingManager(
    private val context: Context,
    private val backendService: BackendService
) : PurchasesUpdatedListener {

    private var billingClient: BillingClient? = null
    private val coroutineScope = CoroutineScope(Dispatchers.Main)

    private val _billingState = MutableStateFlow<BillingState>(BillingState.Idle)
    val billingState: StateFlow<BillingState> = _billingState.asStateFlow()

    private val _products = MutableStateFlow<List<ProductDetails>>(emptyList())
    val products: StateFlow<List<ProductDetails>> = _products.asStateFlow()

    // Product IDs from Google Play Console
    companion object {
        const val PRODUCT_MONTHLY = "vpn_monthly_subscription"
        const val PRODUCT_YEARLY = "vpn_yearly_subscription"
        const val PRODUCT_LIFETIME = "vpn_lifetime_purchase"
    }

    fun initialize() {
        billingClient = BillingClient.newBuilder(context)
            .setListener(this)
            .enablePendingPurchases()
            .build()

        billingClient?.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(billingResult: BillingResult) {
                if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    _billingState.value = BillingState.Connected
                    queryProducts()
                    queryPurchases()
                } else {
                    _billingState.value = BillingState.Error(
                        "Billing setup failed: ${billingResult.debugMessage}"
                    )
                }
            }

            override fun onBillingServiceDisconnected() {
                _billingState.value = BillingState.Disconnected
                // Retry connection
                coroutineScope.launch {
                    kotlinx.coroutines.delay(3000)
                    initialize()
                }
            }
        })
    }

    private fun queryProducts() {
        coroutineScope.launch {
            val productList = listOf(
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(PRODUCT_MONTHLY)
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build(),
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(PRODUCT_YEARLY)
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build(),
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(PRODUCT_LIFETIME)
                    .setProductType(BillingClient.ProductType.INAPP)
                    .build()
            )

            val params = QueryProductDetailsParams.newBuilder()
                .setProductList(productList)
                .build()

            withContext(Dispatchers.IO) {
                billingClient?.queryProductDetailsAsync(params) { billingResult, productDetailsList ->
                    if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                        _products.value = productDetailsList
                    }
                }
            }
        }
    }

    fun launchPurchaseFlow(
        activity: Activity,
        productDetails: ProductDetails,
        basePlanId: String? = null // For multiple subscription plans
    ) {
        val productDetailsParamsList = listOf(
            BillingFlowParams.ProductDetailsParams.newBuilder()
                .setProductDetails(productDetails)
                .apply {
                    // For subscriptions, set the offer token with base plan support
                    productDetails.subscriptionOfferDetails?.let { offers ->
                        // If basePlanId is provided, find the matching offer
                        val targetOffer = if (basePlanId != null) {
                            offers.firstOrNull { it.basePlanId == basePlanId }
                        } else {
                            offers.firstOrNull()
                        }

                        targetOffer?.let {
                            setOfferToken(it.offerToken)
                        }
                    }
                }
                .build()
        )

        val billingFlowParams = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(productDetailsParamsList)
            .build()

        val billingResult = billingClient?.launchBillingFlow(activity, billingFlowParams)

        // Log the result for debugging
        if (billingResult?.responseCode != BillingClient.BillingResponseCode.OK) {
            _billingState.value = BillingState.Error(
                "Failed to launch billing flow: ${billingResult?.debugMessage}"
            )
        }
    }

    override fun onPurchasesUpdated(
        billingResult: BillingResult,
        purchases: List<Purchase>?
    ) {
        when (billingResult.responseCode) {
            BillingClient.BillingResponseCode.OK -> {
                purchases?.forEach { purchase ->
                    handlePurchase(purchase)
                }
            }
            BillingClient.BillingResponseCode.USER_CANCELED -> {
                _billingState.value = BillingState.Cancelled
            }
            else -> {
                _billingState.value = BillingState.Error(
                    "Purchase failed: ${billingResult.debugMessage}"
                )
            }
        }
    }

    private fun handlePurchase(purchase: Purchase) {
        coroutineScope.launch {
            _billingState.value = BillingState.Processing

            // Step 1: Verify with backend
            val verificationResult = verifyPurchaseWithBackend(purchase)

            if (verificationResult) {
                // Step 2: Acknowledge the purchase (required within 3 days)
                // IMPORTANT: Subscriptions auto-acknowledge. Only acknowledge one-time purchases
                if (!purchase.isAcknowledged) {
                    val productId = purchase.products.firstOrNull() ?: ""
                    val isSubscription = productId.contains("subscription", ignoreCase = true)

                    if (!isSubscription) {
                        // For one-time purchases (INAPP), acknowledge them
                        acknowledgePurchase(purchase)
                    }
                    // Subscriptions are auto-acknowledged by Google Play
                }
                _billingState.value = BillingState.PurchaseSuccess(purchase)
            } else {
                _billingState.value = BillingState.Error("Backend verification failed")
            }
        }
    }

    private suspend fun verifyPurchaseWithBackend(purchase: Purchase): Boolean {
        return withContext(Dispatchers.IO) {
            try {
                val request = PurchaseVerificationRequest(
                    purchaseToken = purchase.purchaseToken,
                    productId = purchase.products.firstOrNull() ?: "",
                    orderId = purchase.orderId ?: "",
                    packageName = purchase.packageName
                )

                val response = backendService.verifyPurchase(request)
                response.isSuccessful && response.body()?.isValid == true
            } catch (e: Exception) {
                e.printStackTrace()
                false
            }
        }
    }

    private suspend fun acknowledgePurchase(purchase: Purchase) {
        withContext(Dispatchers.IO) {
            val acknowledgePurchaseParams = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()

            billingClient?.acknowledgePurchase(acknowledgePurchaseParams) { billingResult ->
                if (billingResult.responseCode != BillingClient.BillingResponseCode.OK) {
                    // Handle acknowledgment failure
                }
            }
        }
    }

    private fun queryPurchases() {
        coroutineScope.launch {
            // Query subscriptions
            val subsResult = withContext(Dispatchers.IO) {
                billingClient?.queryPurchasesAsync(
                    QueryPurchasesParams.newBuilder()
                        .setProductType(BillingClient.ProductType.SUBS)
                        .build()
                )
            }

            // Query one-time purchases
            val inAppResult = withContext(Dispatchers.IO) {
                billingClient?.queryPurchasesAsync(
                    QueryPurchasesParams.newBuilder()
                        .setProductType(BillingClient.ProductType.INAPP)
                        .build()
                )
            }

            val allPurchases = mutableListOf<Purchase>()
            subsResult?.purchasesList?.let { allPurchases.addAll(it) }
            inAppResult?.purchasesList?.let { allPurchases.addAll(it) }

            allPurchases.forEach { purchase ->
                if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
                    handlePurchase(purchase)
                }
            }
        }
    }

    fun destroy() {
        billingClient?.endConnection()
    }
}

sealed class BillingState {
    object Idle : BillingState()
    object Connected : BillingState()
    object Disconnected : BillingState()
    object Processing : BillingState()
    object Cancelled : BillingState()
    data class PurchaseSuccess(val purchase: Purchase) : BillingState()
    data class Error(val message: String) : BillingState()
}
```

### 3. Backend Service Interface

```kotlin
// BackendService.kt
package com.vpnapp.billing

import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.GET
import retrofit2.http.Header
import retrofit2.http.POST

interface BackendService {

    @POST("/api/v1/billing/verify")
    suspend fun verifyPurchase(
        @Body request: PurchaseVerificationRequest
    ): Response<PurchaseVerificationResponse>

    @GET("/api/v1/user/subscription")
    suspend fun getSubscriptionStatus(
        @Header("Authorization") token: String
    ): Response<SubscriptionStatus>
}

data class PurchaseVerificationRequest(
    val purchaseToken: String,
    val productId: String,
    val orderId: String,
    val packageName: String
)

data class PurchaseVerificationResponse(
    val isValid: Boolean,
    val expiryDate: Long?,
    val subscriptionType: String
)

data class SubscriptionStatus(
    val isActive: Boolean,
    val expiryDate: Long?,
    val subscriptionType: String
)
```

### 4. ViewModel for UI

```kotlin
// BillingViewModel.kt
package com.vpnapp.billing

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.android.billingclient.api.ProductDetails
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn

class BillingViewModel(
    private val billingManager: BillingManager
) : ViewModel() {

    val billingState: StateFlow<BillingState> = billingManager.billingState
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = BillingState.Idle
        )

    val products: StateFlow<List<ProductDetails>> = billingManager.products
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    init {
        billingManager.initialize()
    }

    override fun onCleared() {
        super.onCleared()
        billingManager.destroy()
    }
}
```

### 5. Jetpack Compose UI

```kotlin
// SubscriptionScreen.kt
package com.vpnapp.ui

import android.app.Activity
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.android.billingclient.api.ProductDetails
import com.vpnapp.billing.BillingManager
import com.vpnapp.billing.BillingState
import com.vpnapp.billing.BillingViewModel

@Composable
fun SubscriptionScreen(
    viewModel: BillingViewModel,
    billingManager: BillingManager
) {
    val billingState by viewModel.billingState.collectAsStateWithLifecycle()
    val products by viewModel.products.collectAsStateWithLifecycle()
    val activity = LocalContext.current as Activity

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Choose Your Plan") }
            )
        }
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            when (billingState) {
                is BillingState.Processing -> {
                    CircularProgressIndicator(
                        modifier = Modifier.align(Alignment.Center)
                    )
                }
                is BillingState.Error -> {
                    ErrorMessage(
                        message = (billingState as BillingState.Error).message,
                        modifier = Modifier.align(Alignment.Center)
                    )
                }
                is BillingState.PurchaseSuccess -> {
                    SuccessMessage(
                        modifier = Modifier.align(Alignment.Center)
                    )
                }
                else -> {
                    ProductList(
                        products = products,
                        onProductClick = { product ->
                            billingManager.launchPurchaseFlow(activity, product)
                        }
                    )
                }
            }
        }
    }
}

@Composable
fun ProductList(
    products: List<ProductDetails>,
    onProductClick: (ProductDetails) -> Unit
) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(products) { product ->
            ProductCard(
                product = product,
                onClick = { onProductClick(product) }
            )
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProductCard(
    product: ProductDetails,
    onClick: () -> Unit
) {
    Card(
        onClick = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = product.name,
                style = MaterialTheme.typography.titleLarge
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = product.description,
                style = MaterialTheme.typography.bodyMedium
            )
            Spacer(modifier = Modifier.height(8.dp))

            // Display pricing
            product.subscriptionOfferDetails?.firstOrNull()?.pricingPhases?.pricingPhaseList?.firstOrNull()?.let { phase ->
                Text(
                    text = "${phase.formattedPrice} / ${phase.billingPeriod}",
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.primary
                )
            } ?: product.oneTimePurchaseOfferDetails?.let { offer ->
                Text(
                    text = offer.formattedPrice,
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.primary
                )
            }
        }
    }
}

@Composable
fun ErrorMessage(message: String, modifier: Modifier = Modifier) {
    Column(
        modifier = modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Error",
            style = MaterialTheme.typography.titleLarge,
            color = MaterialTheme.colorScheme.error
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(text = message)
    }
}

@Composable
fun SuccessMessage(modifier: Modifier = Modifier) {
    Column(
        modifier = modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "Purchase Successful!",
            style = MaterialTheme.typography.titleLarge,
            color = MaterialTheme.colorScheme.primary
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(text = "Your subscription is now active")
    }
}
```

---

## Backend Verification

### 6. Backend Implementation (Node.js + Express Example)

```javascript
// server.js
const express = require('express');
const { google } = require('googleapis');
const app = express();

app.use(express.json());

// Load service account credentials
const serviceAccount = require('./service-account-key.json');

// Initialize Google Play Developer API
const auth = new google.auth.GoogleAuth({
  credentials: serviceAccount,
  scopes: ['https://www.googleapis.com/auth/androidpublisher']
});

const androidPublisher = google.androidpublisher({
  version: 'v3',
  auth
});

const PACKAGE_NAME = 'com.yourcompany.vpnapp';

// Verify subscription purchase
app.post('/api/v1/billing/verify', async (req, res) => {
  try {
    const { purchaseToken, productId, orderId, packageName } = req.body;

    // Validate input
    if (!purchaseToken || !productId || packageName !== PACKAGE_NAME) {
      return res.status(400).json({
        isValid: false,
        error: 'Invalid request parameters'
      });
    }

    // Determine if it's a subscription or one-time purchase
    const isSubscription = productId.includes('subscription');

    let purchaseDetails;

    if (isSubscription) {
      // Verify subscription
      const response = await androidPublisher.purchases.subscriptions.get({
        packageName: PACKAGE_NAME,
        subscriptionId: productId,
        token: purchaseToken
      });

      purchaseDetails = response.data;

      // Check if subscription is valid
      const expiryTimeMillis = parseInt(purchaseDetails.expiryTimeMillis);
      const isValid =
        purchaseDetails.paymentState === 1 && // Payment received
        expiryTimeMillis > Date.now() && // Not expired
        purchaseDetails.cancelReason === undefined; // Not cancelled

      // Store in database
      await storeSubscription({
        userId: req.user?.id, // Assuming you have user auth
        orderId,
        productId,
        purchaseToken,
        expiryDate: new Date(expiryTimeMillis),
        autoRenewing: purchaseDetails.autoRenewing,
        isValid
      });

      return res.json({
        isValid,
        expiryDate: expiryTimeMillis,
        subscriptionType: productId,
        autoRenewing: purchaseDetails.autoRenewing
      });

    } else {
      // Verify one-time purchase
      const response = await androidPublisher.purchases.products.get({
        packageName: PACKAGE_NAME,
        productId: productId,
        token: purchaseToken
      });

      purchaseDetails = response.data;

      const isValid =
        purchaseDetails.purchaseState === 0 && // Purchased
        purchaseDetails.consumptionState === 0; // Not consumed

      // Store in database
      await storePurchase({
        userId: req.user?.id,
        orderId,
        productId,
        purchaseToken,
        isValid
      });

      return res.json({
        isValid,
        expiryDate: null, // Lifetime
        subscriptionType: productId
      });
    }

  } catch (error) {
    console.error('Verification error:', error);
    return res.status(500).json({
      isValid: false,
      error: 'Verification failed'
    });
  }
});

// Get user subscription status
app.get('/api/v1/user/subscription', async (req, res) => {
  try {
    // Get user from auth token
    const userId = req.user?.id;

    // Query database for active subscription
    const subscription = await getActiveSubscription(userId);

    if (!subscription) {
      return res.json({
        isActive: false,
        expiryDate: null,
        subscriptionType: null
      });
    }

    // Verify subscription is still valid with Google
    const response = await androidPublisher.purchases.subscriptions.get({
      packageName: PACKAGE_NAME,
      subscriptionId: subscription.productId,
      token: subscription.purchaseToken
    });

    const details = response.data;
    const expiryTimeMillis = parseInt(details.expiryTimeMillis);
    const isActive =
      details.paymentState === 1 &&
      expiryTimeMillis > Date.now() &&
      details.cancelReason === undefined;

    return res.json({
      isActive,
      expiryDate: expiryTimeMillis,
      subscriptionType: subscription.productId
    });

  } catch (error) {
    console.error('Status check error:', error);
    return res.status(500).json({
      isActive: false,
      error: 'Status check failed'
    });
  }
});

// Database functions (implement with your DB of choice)
async function storeSubscription(data) {
  // Store in PostgreSQL, MongoDB, etc.
  // Example: await db.subscriptions.create(data);
}

async function storePurchase(data) {
  // Store one-time purchase
  // Example: await db.purchases.create(data);
}

async function getActiveSubscription(userId) {
  // Query active subscription for user
  // Example: return await db.subscriptions.findOne({ userId, isValid: true });
  return null;
}

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 7. Python Backend Example (FastAPI)

```python
# main.py
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from google.oauth2 import service_account
from googleapiclient.discovery import build
from datetime import datetime
import os

app = FastAPI()

# Load service account credentials
CREDENTIALS_FILE = 'service-account-key.json'
SCOPES = ['https://www.googleapis.com/auth/androidpublisher']
PACKAGE_NAME = 'com.yourcompany.vpnapp'

credentials = service_account.Credentials.from_service_account_file(
    CREDENTIALS_FILE, scopes=SCOPES
)

android_publisher = build('androidpublisher', 'v3', credentials=credentials)

class PurchaseVerificationRequest(BaseModel):
    purchaseToken: str
    productId: str
    orderId: str
    packageName: str

class PurchaseVerificationResponse(BaseModel):
    isValid: bool
    expiryDate: int | None
    subscriptionType: str
    autoRenewing: bool | None = None

@app.post("/api/v1/billing/verify", response_model=PurchaseVerificationResponse)
async def verify_purchase(request: PurchaseVerificationRequest):
    try:
        if request.packageName != PACKAGE_NAME:
            raise HTTPException(status_code=400, detail="Invalid package name")

        is_subscription = 'subscription' in request.productId

        if is_subscription:
            # Verify subscription
            result = android_publisher.purchases().subscriptions().get(
                packageName=PACKAGE_NAME,
                subscriptionId=request.productId,
                token=request.purchaseToken
            ).execute()

            expiry_time_millis = int(result.get('expiryTimeMillis', 0))
            is_valid = (
                result.get('paymentState') == 1 and
                expiry_time_millis > int(datetime.now().timestamp() * 1000) and
                result.get('cancelReason') is None
            )

            # Store in database
            await store_subscription({
                'orderId': request.orderId,
                'productId': request.productId,
                'purchaseToken': request.purchaseToken,
                'expiryDate': datetime.fromtimestamp(expiry_time_millis / 1000),
                'autoRenewing': result.get('autoRenewing'),
                'isValid': is_valid
            })

            return PurchaseVerificationResponse(
                isValid=is_valid,
                expiryDate=expiry_time_millis,
                subscriptionType=request.productId,
                autoRenewing=result.get('autoRenewing')
            )
        else:
            # Verify one-time purchase
            result = android_publisher.purchases().products().get(
                packageName=PACKAGE_NAME,
                productId=request.productId,
                token=request.purchaseToken
            ).execute()

            is_valid = (
                result.get('purchaseState') == 0 and
                result.get('consumptionState') == 0
            )

            await store_purchase({
                'orderId': request.orderId,
                'productId': request.productId,
                'purchaseToken': request.purchaseToken,
                'isValid': is_valid
            })

            return PurchaseVerificationResponse(
                isValid=is_valid,
                expiryDate=None,
                subscriptionType=request.productId
            )

    except Exception as e:
        print(f"Verification error: {e}")
        raise HTTPException(status_code=500, detail="Verification failed")

async def store_subscription(data):
    # Implement database storage
    pass

async def store_purchase(data):
    # Implement database storage
    pass
```

---

## Security Best Practices

### 8. Security Checklist

#### ✅ Android Side
- [ ] Never store sensitive billing data in SharedPreferences
- [ ] Always verify purchases on the backend
- [ ] Use ProGuard/R8 to obfuscate billing code
- [ ] Implement certificate pinning for backend communication
- [ ] Use encrypted storage for user tokens
- [ ] Validate all purchase data before sending to backend

#### ✅ Backend Side
- [ ] Always verify purchases with Google Play Developer API
- [ ] Implement rate limiting on verification endpoints
- [ ] Use secure authentication (JWT, OAuth2)
- [ ] Store service account credentials securely (env variables, secrets manager)
- [ ] Log all verification attempts for audit
- [ ] Implement webhook handling for Real-time Developer Notifications (RTDN)
- [ ] Use HTTPS only
- [ ] Validate purchase tokens haven't been used before (prevent replay attacks)

#### ✅ Database Security
- [ ] Hash/encrypt sensitive purchase tokens
- [ ] Index on userId and productId for fast lookups
- [ ] Implement backup strategy
- [ ] Set up monitoring and alerts for unusual activity

### 9. ProGuard Rules

```proguard
# proguard-rules.pro

# Keep billing classes
-keep class com.android.billingclient.api.** { *; }
-keep class com.vpnapp.billing.** { *; }

# Keep Purchase class for verification
-keepclassmembers class com.android.billingclient.api.Purchase {
    public <init>(...);
    *** getPurchaseToken();
    *** getOrderId();
    *** getPackageName();
}
```

---

## Testing Strategy

### 10. Testing on Android

#### Test with License Testers
1. Go to Google Play Console → Setup → License Testing
2. Add test Gmail accounts
3. Use these accounts to make test purchases (no real charges)

#### Test Products
```kotlin
// Add test product IDs in debug builds
object BillingProducts {
    val MONTHLY = if (BuildConfig.DEBUG) {
        "android.test.purchased"  // Always succeeds
    } else {
        "vpn_monthly_subscription"
    }

    val CANCELLED = if (BuildConfig.DEBUG) {
        "android.test.canceled"  // Always cancelled
    } else {
        "vpn_yearly_subscription"
    }
}
```

#### Unit Tests
```kotlin
// BillingManagerTest.kt
@Test
fun `test purchase verification with valid token`() = runTest {
    val mockBackend = mockk<BackendService>()
    val mockPurchase = mockk<Purchase>()

    coEvery {
        mockBackend.verifyPurchase(any())
    } returns Response.success(
        PurchaseVerificationResponse(
            isValid = true,
            expiryDate = System.currentTimeMillis() + 86400000,
            subscriptionType = "monthly"
        )
    )

    // Test verification logic
}
```

### 11. Testing Backend

#### Mock Google API Responses
```javascript
// test/billing.test.js
const request = require('supertest');
const app = require('../server');

jest.mock('googleapis');

describe('Billing Verification', () => {
  test('should verify valid subscription', async () => {
    // Mock Google API response
    const mockGet = jest.fn().mockResolvedValue({
      data: {
        paymentState: 1,
        expiryTimeMillis: Date.now() + 86400000,
        autoRenewing: true
      }
    });

    const response = await request(app)
      .post('/api/v1/billing/verify')
      .send({
        purchaseToken: 'test_token',
        productId: 'vpn_monthly_subscription',
        orderId: 'ORDER_123',
        packageName: 'com.yourcompany.vpnapp'
      });

    expect(response.status).toBe(200);
    expect(response.body.isValid).toBe(true);
  });
});
```

---

## Real-time Developer Notifications (RTDN)

### 12. Webhook Implementation

```javascript
// webhook.js
app.post('/api/v1/billing/webhook', async (req, res) => {
  try {
    // Verify the message is from Google
    const message = req.body.message;
    const data = message.data;
    const decodedData = JSON.parse(Buffer.from(data, 'base64').toString());

    const notificationType = decodedData.notificationType;
    const subscriptionNotification = decodedData.subscriptionNotification;

    switch (notificationType) {
      case 1: // SUBSCRIPTION_RECOVERED
        await handleSubscriptionRecovered(subscriptionNotification);
        break;
      case 2: // SUBSCRIPTION_RENEWED
        await handleSubscriptionRenewed(subscriptionNotification);
        break;
      case 3: // SUBSCRIPTION_CANCELED
        await handleSubscriptionCanceled(subscriptionNotification);
        break;
      case 4: // SUBSCRIPTION_PURCHASED
        await handleSubscriptionPurchased(subscriptionNotification);
        break;
      case 5: // SUBSCRIPTION_ON_HOLD
        await handleSubscriptionOnHold(subscriptionNotification);
        break;
      case 6: // SUBSCRIPTION_IN_GRACE_PERIOD
        await handleSubscriptionGracePeriod(subscriptionNotification);
        break;
      case 7: // SUBSCRIPTION_RESTARTED
        await handleSubscriptionRestarted(subscriptionNotification);
        break;
      case 8: // SUBSCRIPTION_PRICE_CHANGE_CONFIRMED
        await handlePriceChangeConfirmed(subscriptionNotification);
        break;
      case 9: // SUBSCRIPTION_DEFERRED
        await handleSubscriptionDeferred(subscriptionNotification);
        break;
      case 10: // SUBSCRIPTION_PAUSED
        await handleSubscriptionPaused(subscriptionNotification);
        break;
      case 11: // SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED
        await handlePauseScheduleChanged(subscriptionNotification);
        break;
      case 12: // SUBSCRIPTION_REVOKED
        await handleSubscriptionRevoked(subscriptionNotification);
        break;
      case 13: // SUBSCRIPTION_EXPIRED
        await handleSubscriptionExpired(subscriptionNotification);
        break;
    }

    res.status(200).send('OK');
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).send('Error');
  }
});

async function handleSubscriptionRenewed(notification) {
  const { purchaseToken, subscriptionId } = notification;

  // Update database
  await updateSubscriptionRenewal(purchaseToken, subscriptionId);

  // Notify user via push notification
  await sendPushNotification(
    await getUserByToken(purchaseToken),
    'Subscription Renewed',
    'Your VPN subscription has been renewed successfully!'
  );
}

async function handleSubscriptionCanceled(notification) {
  const { purchaseToken, subscriptionId } = notification;

  // Update database
  await markSubscriptionCanceled(purchaseToken, subscriptionId);

  // Notify user
  await sendPushNotification(
    await getUserByToken(purchaseToken),
    'Subscription Canceled',
    'Your subscription has been canceled and will expire soon.'
  );
}
```

---

## Database Schema

### 13. Recommended Schema (PostgreSQL)

```sql
-- users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- subscriptions table
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    order_id VARCHAR(255) UNIQUE NOT NULL,
    product_id VARCHAR(100) NOT NULL,
    purchase_token TEXT UNIQUE NOT NULL,
    package_name VARCHAR(255) NOT NULL,
    purchase_time TIMESTAMP NOT NULL,
    expiry_date TIMESTAMP,
    auto_renewing BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    notification_type INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_id (user_id),
    INDEX idx_purchase_token (purchase_token),
    INDEX idx_expiry_date (expiry_date),
    INDEX idx_is_active (is_active)
);

-- purchases table (one-time purchases)
CREATE TABLE purchases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    order_id VARCHAR(255) UNIQUE NOT NULL,
    product_id VARCHAR(100) NOT NULL,
    purchase_token TEXT UNIQUE NOT NULL,
    package_name VARCHAR(255) NOT NULL,
    purchase_time TIMESTAMP NOT NULL,
    is_valid BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_id (user_id),
    INDEX idx_purchase_token (purchase_token)
);

-- audit log for all verification attempts
CREATE TABLE billing_audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    action VARCHAR(50) NOT NULL,
    purchase_token TEXT,
    product_id VARCHAR(100),
    success BOOLEAN,
    error_message TEXT,
    ip_address VARCHAR(45),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_created_at (created_at),
    INDEX idx_user_id (user_id)
);
```

---

## Deployment Checklist

### 14. Pre-Launch Checklist

#### Google Play Console Setup
- [ ] Create in-app products and subscriptions
- [ ] Set pricing for all countries
- [ ] Configure subscription benefits and features
- [ ] Set up Real-time Developer Notifications
- [ ] Add license testers
- [ ] Enable Google Play Billing API

#### Service Account Setup
- [ ] Create service account in Google Cloud Console
- [ ] Grant permissions: "View financial data, orders, and cancellation survey responses"
- [ ] Download JSON key file
- [ ] Store securely on backend (DO NOT commit to git)

#### Backend Setup
- [ ] Deploy backend with HTTPS
- [ ] Set up environment variables
- [ ] Configure database
- [ ] Set up monitoring and logging
- [ ] Test webhook endpoint
- [ ] Implement rate limiting

#### Android App
- [ ] Test all purchase flows
- [ ] Test restoration of purchases
- [ ] Test error handling
- [ ] Enable ProGuard for release build
- [ ] Test on multiple devices
- [ ] Verify backend integration

---

## Monitoring and Analytics

### 15. Key Metrics to Track

```kotlin
// Analytics events to track
object BillingAnalytics {
    const val PURCHASE_INITIATED = "purchase_initiated"
    const val PURCHASE_SUCCESS = "purchase_success"
    const val PURCHASE_FAILED = "purchase_failed"
    const val PURCHASE_CANCELLED = "purchase_cancelled"
    const val VERIFICATION_SUCCESS = "verification_success"
    const val VERIFICATION_FAILED = "verification_failed"
    const val SUBSCRIPTION_EXPIRED = "subscription_expired"
    const val SUBSCRIPTION_RENEWED = "subscription_renewed"
}

// Log events
fun logPurchaseEvent(event: String, params: Map<String, Any>) {
    // Firebase Analytics, Mixpanel, etc.
    analytics.logEvent(event, params)
}
```

---

## Additional Best Practices (Based on Latest Google Guidelines)

### 16. Connection Retry Logic

**CRITICAL**: Always implement proper connection retry with exponential backoff:

```kotlin
class BillingManager(
    private val context: Context,
    private val backendService: BackendService
) : PurchasesUpdatedListener {

    private var retryAttempts = 0
    private val maxRetries = 3
    private var retryJob: Job? = null

    private fun retryBillingConnection() {
        if (retryAttempts < maxRetries) {
            retryAttempts++
            val delayMillis = (2.0.pow(retryAttempts) * 1000).toLong() // Exponential backoff

            retryJob?.cancel()
            retryJob = coroutineScope.launch {
                delay(delayMillis)
                initialize()
            }
        } else {
            _billingState.value = BillingState.Error("Failed to connect after $maxRetries attempts")
        }
    }

    override fun onBillingServiceDisconnected() {
        _billingState.value = BillingState.Disconnected
        retryBillingConnection()
    }
}
```

### 17. Pending Purchase Handling

**CRITICAL**: Handle pending purchases (Payment methods requiring additional steps):

```kotlin
private fun handlePurchase(purchase: Purchase) {
    coroutineScope.launch {
        when (purchase.purchaseState) {
            Purchase.PurchaseState.PURCHASED -> {
                _billingState.value = BillingState.Processing
                val verificationResult = verifyPurchaseWithBackend(purchase)

                if (verificationResult) {
                    if (!purchase.isAcknowledged) {
                        val productId = purchase.products.firstOrNull() ?: ""
                        val isSubscription = productId.contains("subscription", ignoreCase = true)

                        if (!isSubscription) {
                            acknowledgePurchase(purchase)
                        }
                    }
                    _billingState.value = BillingState.PurchaseSuccess(purchase)
                } else {
                    _billingState.value = BillingState.Error("Backend verification failed")
                }
            }

            Purchase.PurchaseState.PENDING -> {
                // Payment is pending (e.g., cash payment methods)
                _billingState.value = BillingState.Pending(
                    "Payment is being processed. You'll get access once payment is confirmed."
                )
                // Store pending purchase in database for later verification
                storePendingPurchase(purchase)
            }

            Purchase.PurchaseState.UNSPECIFIED_STATE -> {
                _billingState.value = BillingState.Error("Purchase state unspecified")
            }
        }
    }
}

sealed class BillingState {
    object Idle : BillingState()
    object Connected : BillingState()
    object Disconnected : BillingState()
    object Processing : BillingState()
    object Cancelled : BillingState()
    data class Pending(val message: String) : BillingState()
    data class PurchaseSuccess(val purchase: Purchase) : BillingState()
    data class Error(val message: String) : BillingState()
}
```

### 18. Grace Period and Account Hold Handling

Add support for subscription grace periods and account holds:

```kotlin
// In backend verification response, add these fields:
data class PurchaseVerificationResponse(
    val isValid: Boolean,
    val expiryDate: Long?,
    val subscriptionType: String,
    val autoRenewing: Boolean? = null,
    val paymentState: Int? = null, // 0 = Pending, 1 = Received, 2 = Free trial, 3 = Pending deferred
    val isGracePeriod: Boolean = false, // User in grace period
    val isAccountHold: Boolean = false // Payment issue, account on hold
)

// Show appropriate UI based on subscription state
@Composable
fun SubscriptionStatusBanner(status: PurchaseVerificationResponse) {
    when {
        status.isAccountHold -> {
            AlertBanner(
                message = "Payment issue detected. Please update your payment method to continue service.",
                severity = Severity.ERROR
            )
        }
        status.isGracePeriod -> {
            AlertBanner(
                message = "Payment failed. You have until ${formatDate(status.expiryDate)} to update payment.",
                severity = Severity.WARNING
            )
        }
        !status.autoRenewing ?: false -> {
            AlertBanner(
                message = "Your subscription will expire on ${formatDate(status.expiryDate)}",
                severity = Severity.INFO
            )
        }
    }
}
```

### 19. Offer Tags and Promo Codes Support

```kotlin
fun launchPurchaseFlowWithOffer(
    activity: Activity,
    productDetails: ProductDetails,
    basePlanId: String,
    offerTag: String? = null // For promotional offers
) {
    val offers = productDetails.subscriptionOfferDetails ?: return

    val targetOffer = if (offerTag != null) {
        // Find offer by tag (e.g., "intro-offer", "promo-2024")
        offers.firstOrNull {
            it.basePlanId == basePlanId &&
            it.offerTags.contains(offerTag)
        }
    } else {
        offers.firstOrNull { it.basePlanId == basePlanId }
    } ?: return

    val productDetailsParamsList = listOf(
        BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(productDetails)
            .setOfferToken(targetOffer.offerToken)
            .build()
    )

    val billingFlowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(productDetailsParamsList)
        .build()

    billingClient?.launchBillingFlow(activity, billingFlowParams)
}
```

### 20. Subscription Upgrade/Downgrade Flow

```kotlin
fun upgradeSubscription(
    activity: Activity,
    newProductDetails: ProductDetails,
    newBasePlanId: String,
    oldPurchaseToken: String
) {
    val offers = newProductDetails.subscriptionOfferDetails ?: return
    val targetOffer = offers.firstOrNull { it.basePlanId == newBasePlanId } ?: return

    val productDetailsParamsList = listOf(
        BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(newProductDetails)
            .setOfferToken(targetOffer.offerToken)
            .build()
    )

    val billingFlowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(productDetailsParamsList)
        .setSubscriptionUpdateParams(
            BillingFlowParams.SubscriptionUpdateParams.newBuilder()
                .setOldPurchaseToken(oldPurchaseToken)
                .setSubscriptionReplacementMode(
                    BillingFlowParams.SubscriptionUpdateParams.ReplacementMode.CHARGE_PRORATED_PRICE
                    // Other modes:
                    // WITHOUT_PRORATION - Charge full price
                    // WITH_TIME_PRORATION - Prorate based on time
                    // CHARGE_FULL_PRICE - Immediately charge full price
                )
                .build()
        )
        .build()

    billingClient?.launchBillingFlow(activity, billingFlowParams)
}
```

---

## Critical Security Addition: Play Integrity API

### 21. Integrate Play Integrity API (Prevent Fraud)

**IMPORTANT**: Google now recommends using Play Integrity API to verify the authenticity of requests:

```kotlin
// Add dependency
dependencies {
    implementation("com.google.android.play:integrity:1.3.0")
}

// Get integrity token
class PlayIntegrityHelper(private val context: Context) {

    suspend fun getIntegrityToken(): String? = withContext(Dispatchers.IO) {
        try {
            val integrityManager = IntegrityManagerFactory.create(context)

            // Create a nonce (random data) to prevent replay attacks
            val nonce = generateNonce()

            val integrityTokenResponse = integrityManager
                .requestIntegrityToken(
                    IntegrityTokenRequest.builder()
                        .setNonce(nonce)
                        .build()
                )
                .await()

            integrityTokenResponse.token()
        } catch (e: Exception) {
            e.printStackTrace()
            null
        }
    }

    private fun generateNonce(): String {
        // Generate random nonce and hash it
        val random = SecureRandom()
        val bytes = ByteArray(32)
        random.nextBytes(bytes)
        return Base64.encodeToString(bytes, Base64.NO_WRAP)
    }
}

// Include integrity token in purchase verification
data class PurchaseVerificationRequest(
    val purchaseToken: String,
    val productId: String,
    val orderId: String,
    val packageName: String,
    val integrityToken: String? = null // Add this
)

// Update backend to verify integrity token
// Backend verification (Node.js example)
const { GoogleAuth } = require('google-auth-library');

async function verifyIntegrityToken(integrityToken, packageName) {
    try {
        const auth = new GoogleAuth({
            credentials: serviceAccount,
            scopes: ['https://www.googleapis.com/auth/playintegrity']
        });

        const client = await auth.getClient();
        const url = `https://playintegrity.googleapis.com/v1/${packageName}/decodeIntegrityToken`;

        const response = await client.request({
            url,
            method: 'POST',
            data: {
                integrity_token: integrityToken
            }
        });

        const tokenPayload = response.data.tokenPayloadExternal;

        // Verify app integrity
        if (tokenPayload.appIntegrity.appRecognitionVerdict !== 'PLAY_RECOGNIZED') {
            return { valid: false, reason: 'App not recognized by Play Store' };
        }

        // Verify device integrity
        if (!tokenPayload.deviceIntegrity.deviceRecognitionVerdict.includes('MEETS_DEVICE_INTEGRITY')) {
            return { valid: false, reason: 'Device integrity check failed' };
        }

        return { valid: true };
    } catch (error) {
        console.error('Integrity verification error:', error);
        return { valid: false, reason: 'Integrity verification failed' };
    }
}
```

---

## Conclusion

This POC provides a complete end-to-end implementation for in-app purchases with backend verification for your VPN app. Key takeaways:

1. **Always verify purchases on the backend** - Never trust client-side verification
2. **Use Real-time Developer Notifications** - Stay updated on subscription changes
3. **Implement proper error handling** - Network issues, verification failures, etc.
4. **Test thoroughly** - Use license testers before production
5. **Monitor and log everything** - Track all purchase events for debugging and analytics
6. **Keep security in mind** - Encrypt tokens, use HTTPS, implement rate limiting

### Next Steps
1. Set up Google Play Console products
2. Implement the Android billing code
3. Set up backend with Google Play Developer API
4. Test with license testers
5. Deploy and monitor

### Additional Resources
- [Google Play Billing Library Documentation](https://developer.android.com/google/play/billing)
- [Google Play Developer API](https://developers.google.com/android-publisher)
- [Real-time Developer Notifications](https://developer.android.com/google/play/billing/rtdn-reference)
