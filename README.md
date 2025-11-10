# Google Play Subscriptions POC with RTDN Support

A complete end-to-end proof of concept implementation for Google Play In-App Subscriptions with Real-Time Developer Notifications (RTDN) backend support.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Guide](#setup-guide)
  - [1. Google Play Console Setup](#1-google-play-console-setup)
  - [2. Google Cloud Platform Setup](#2-google-cloud-platform-setup)
  - [3. Backend Setup](#3-backend-setup)
  - [4. Android App Setup](#4-android-app-setup)
- [API Documentation](#api-documentation)
- [RTDN Implementation](#rtdn-implementation)
- [Testing](#testing)
- [Flow Diagrams](#flow-diagrams)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Overview

This POC demonstrates how to:
- Implement Google Play Billing Library in Android apps
- Verify subscription purchases on the backend
- Handle Real-Time Developer Notifications (RTDN) via Google Cloud Pub/Sub
- Manage subscription lifecycle (purchase, renewal, cancellation, etc.)
- Acknowledge purchases server-side

---

## Architecture

```
┌─────────────────┐
│  Android App    │
│  (Play Billing) │
└────────┬────────┘
         │
         │ 1. Purchase Token
         ▼
┌─────────────────────────────────┐
│  Backend Server (Node.js)       │
│  ┌───────────────────────────┐  │
│  │ Subscription Verification │  │
│  │ Google Play API          │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
         ▲
         │ 2. RTDN Notifications
         │
┌─────────────────────────────────┐
│  Google Cloud Pub/Sub           │
│  (Real-Time Developer           │
│   Notifications)                │
└─────────────────────────────────┘
```

---

## Features

### Backend Features
- ✅ Verify subscription purchases with Google Play API
- ✅ Acknowledge purchases server-side
- ✅ Cancel subscriptions programmatically
- ✅ Get real-time subscription status
- ✅ Handle RTDN webhooks from Google Cloud Pub/Sub
- ✅ Process 13 different subscription notification types
- ✅ RESTful API endpoints

### RTDN Notification Types Supported
1. SUBSCRIPTION_RECOVERED
2. SUBSCRIPTION_RENEWED
3. SUBSCRIPTION_CANCELED
4. SUBSCRIPTION_PURCHASED
5. SUBSCRIPTION_ON_HOLD
6. SUBSCRIPTION_IN_GRACE_PERIOD
7. SUBSCRIPTION_RESTARTED
8. SUBSCRIPTION_PRICE_CHANGE_CONFIRMED
9. SUBSCRIPTION_DEFERRED
10. SUBSCRIPTION_PAUSED
11. SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED
12. SUBSCRIPTION_REVOKED
13. SUBSCRIPTION_EXPIRED

---

## Prerequisites

### Required Accounts
- Google Play Console account (enrolled as a developer)
- Google Cloud Platform account
- Android development environment

### Required Tools
- Node.js (v16 or higher)
- npm or yarn
- Android Studio
- A physical Android device or emulator for testing

### Required Knowledge
- Basic understanding of REST APIs
- Android development with Kotlin/Java
- Node.js/Express basics
- Google Cloud Platform basics

---

## Project Structure

```
in_app_purchase/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   └── googlePlay.js          # Google Play API configuration
│   │   ├── controllers/
│   │   │   ├── subscriptionController.js  # Subscription endpoints
│   │   │   └── rtdnController.js          # RTDN webhook handlers
│   │   ├── services/
│   │   │   ├── subscriptionService.js     # Subscription business logic
│   │   │   └── rtdnService.js             # RTDN processing logic
│   │   ├── routes/
│   │   │   ├── subscriptionRoutes.js      # Subscription routes
│   │   │   └── rtdnRoutes.js              # RTDN routes
│   │   ├── middleware/
│   │   │   └── errorHandler.js            # Error handling
│   │   └── server.js                      # Express server
│   ├── package.json
│   └── .env.example
│
├── android-app/
│   └── (Android app implementation example)
│
└── README.md
```

---

## Setup Guide

### 1. Google Play Console Setup

#### Step 1.1: Create or Select Your App
1. Go to [Google Play Console](https://play.google.com/console)
2. Select your app or create a new one
3. Complete the app setup (store listing, content rating, etc.)

#### Step 1.2: Create Subscription Products
1. Navigate to **Monetize → Products → Subscriptions**
2. Click **Create subscription**
3. Fill in the details:
   - **Product ID**: `premium_monthly` (example)
   - **Name**: Premium Monthly
   - **Description**: Monthly premium subscription
   - **Billing period**: 1 month
   - **Price**: Set your price
4. Click **Save**
5. Repeat for other subscription tiers (e.g., `premium_yearly`)

#### Step 1.3: Add License Testers
1. Go to **Settings → License testing**
2. Add test Gmail accounts
3. Choose response: **License will ALWAYS respond that the user has purchased the item**

---

### 2. Google Cloud Platform Setup

#### Step 2.1: Enable Google Play Developer API
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Navigate to **APIs & Services → Library**
4. Search for "Google Play Android Developer API"
5. Click **Enable**

#### Step 2.2: Create Service Account
1. Go to **IAM & Admin → Service Accounts**
2. Click **Create Service Account**
3. Name: `play-billing-service`
4. Click **Create and Continue**
5. Skip granting roles (we'll set permissions in Play Console)
6. Click **Done**

#### Step 2.3: Generate Service Account Key
1. Click on the created service account
2. Go to **Keys** tab
3. Click **Add Key → Create new key**
4. Choose **JSON** format
5. Click **Create**
6. Save the downloaded JSON file as `service-account-key.json`

#### Step 2.4: Link Service Account to Play Console
1. Copy the service account email (e.g., `play-billing-service@your-project.iam.gserviceaccount.com`)
2. Go back to **Google Play Console**
3. Navigate to **Settings → API access**
4. Click **Link** on your Google Cloud project
5. Grant access to the service account:
   - Click on the service account
   - Click **Invite user**
   - Grant **Financial data** permissions (View app information and financial data)
   - Click **Invite user**

#### Step 2.5: Set Up Cloud Pub/Sub for RTDN
1. In **Google Cloud Console**, go to **Pub/Sub → Topics**
2. Click **Create Topic**
3. Topic ID: `play-rtdn`
4. Click **Create**
5. Copy the full topic name: `projects/YOUR_PROJECT_ID/topics/play-rtdn`

#### Step 2.6: Configure RTDN in Play Console
1. Go to **Google Play Console → Monetization setup**
2. Scroll to **Real-time developer notifications**
3. Enter topic name: `projects/YOUR_PROJECT_ID/topics/play-rtdn`
4. Click **Send test notification** to verify
5. Click **Save changes**

#### Step 2.7: Create Pub/Sub Push Subscription
1. In **Google Cloud Console**, go to **Pub/Sub → Subscriptions**
2. Click **Create Subscription**
3. Subscription ID: `play-rtdn-subscription`
4. Select topic: `play-rtdn`
5. Delivery type: **Push**
6. Endpoint URL: `https://your-backend-domain.com/api/rtdn/webhook`
7. Click **Create**

---

### 3. Backend Setup

#### Step 3.1: Install Dependencies
```bash
cd backend
npm install
```

#### Step 3.2: Configure Environment Variables
```bash
cp .env.example .env
```

Edit `.env` file:
```env
# Server Configuration
PORT=3000
NODE_ENV=development

# Google Play Console Configuration
GOOGLE_APPLICATION_CREDENTIALS=./service-account-key.json
PACKAGE_NAME=com.example.yourapp

# RTDN Configuration
RTDN_TOPIC=projects/YOUR_PROJECT_ID/topics/play-rtdn
```

#### Step 3.3: Add Service Account Key
1. Copy the `service-account-key.json` file to the `backend/` directory
2. Ensure it's in the same location as specified in `.env`

#### Step 3.4: Start the Server
```bash
# Development mode with auto-reload
npm run dev

# Production mode
npm start
```

You should see:
```
╔════════════════════════════════════════════════════════╗
║  Google Play Subscription Backend Server              ║
║  Status: Running                                       ║
║  Port: 3000                                            ║
║  Environment: development                              ║
║  Package: com.example.yourapp                          ║
╚════════════════════════════════════════════════════════╝
```

#### Step 3.5: Deploy to Production
For RTDN to work, your backend must be publicly accessible with HTTPS.

**Option 1: Deploy to Cloud Platform**
- Google Cloud Run
- AWS Elastic Beanstalk
- Heroku
- DigitalOcean App Platform

**Option 2: Use ngrok for Testing**
```bash
# Install ngrok
npm install -g ngrok

# Start your server
npm run dev

# In another terminal, expose it
ngrok http 3000
```

Copy the HTTPS URL (e.g., `https://abc123.ngrok.io`) and update your Pub/Sub push subscription endpoint to:
```
https://abc123.ngrok.io/api/rtdn/webhook
```

---

### 4. Android App Setup

#### Step 4.1: Add Billing Library Dependency

In your app's `build.gradle`:
```gradle
dependencies {
    // Google Play Billing Library
    implementation 'com.android.billingclient:billing:6.1.0'

    // For Kotlin coroutines support
    implementation 'com.android.billingclient:billing-ktx:6.1.0'

    // Networking (Retrofit for API calls)
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
}
```

#### Step 4.2: Add Internet Permission

In `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
```

#### Step 4.3: Implement Billing Manager (Kotlin Example)

Create `BillingManager.kt`:

```kotlin
package com.example.subscriptions

import android.app.Activity
import android.content.Context
import android.util.Log
import com.android.billingclient.api.*
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

class BillingManager(
    private val context: Context,
    private val purchaseListener: PurchaseListener
) : PurchasesUpdatedListener {

    interface PurchaseListener {
        fun onPurchaseSuccess(purchase: Purchase)
        fun onPurchaseFailure(errorMessage: String)
    }

    private var billingClient: BillingClient = BillingClient.newBuilder(context)
        .setListener(this)
        .enablePendingPurchases()
        .build()

    init {
        startConnection()
    }

    private fun startConnection() {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(billingResult: BillingResult) {
                if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    Log.d(TAG, "Billing client connected")
                    querySubscriptions()
                } else {
                    Log.e(TAG, "Billing setup failed: ${billingResult.debugMessage}")
                }
            }

            override fun onBillingServiceDisconnected() {
                Log.w(TAG, "Billing service disconnected. Attempting to reconnect...")
                startConnection()
            }
        })
    }

    fun querySubscriptions() {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(
                listOf(
                    QueryProductDetailsParams.Product.newBuilder()
                        .setProductId("premium_monthly")
                        .setProductType(BillingClient.ProductType.SUBS)
                        .build(),
                    QueryProductDetailsParams.Product.newBuilder()
                        .setProductId("premium_yearly")
                        .setProductType(BillingClient.ProductType.SUBS)
                        .build()
                )
            )
            .build()

        billingClient.queryProductDetailsAsync(params) { billingResult, productDetailsList ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                Log.d(TAG, "Found ${productDetailsList.size} products")
                productDetailsList.forEach { productDetails ->
                    Log.d(TAG, "Product: ${productDetails.productId}")
                }
            } else {
                Log.e(TAG, "Error querying products: ${billingResult.debugMessage}")
            }
        }
    }

    fun launchSubscriptionFlow(activity: Activity, productId: String) {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(
                listOf(
                    QueryProductDetailsParams.Product.newBuilder()
                        .setProductId(productId)
                        .setProductType(BillingClient.ProductType.SUBS)
                        .build()
                )
            )
            .build()

        billingClient.queryProductDetailsAsync(params) { billingResult, productDetailsList ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK
                && productDetailsList.isNotEmpty()) {

                val productDetails = productDetailsList[0]
                val offerToken = productDetails.subscriptionOfferDetails?.get(0)?.offerToken

                if (offerToken != null) {
                    val productDetailsParamsList = listOf(
                        BillingFlowParams.ProductDetailsParams.newBuilder()
                            .setProductDetails(productDetails)
                            .setOfferToken(offerToken)
                            .build()
                    )

                    val flowParams = BillingFlowParams.newBuilder()
                        .setProductDetailsParamsList(productDetailsParamsList)
                        .build()

                    val launchResult = billingClient.launchBillingFlow(activity, flowParams)
                    if (launchResult.responseCode != BillingClient.BillingResponseCode.OK) {
                        Log.e(TAG, "Failed to launch billing flow: ${launchResult.debugMessage}")
                    }
                }
            }
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
                Log.d(TAG, "User canceled the purchase")
                purchaseListener.onPurchaseFailure("Purchase canceled")
            }
            else -> {
                Log.e(TAG, "Purchase failed: ${billingResult.debugMessage}")
                purchaseListener.onPurchaseFailure(billingResult.debugMessage)
            }
        }
    }

    private fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
            if (!purchase.isAcknowledged) {
                // Send to backend for verification
                verifyPurchaseWithBackend(purchase)
            } else {
                purchaseListener.onPurchaseSuccess(purchase)
            }
        }
    }

    private fun verifyPurchaseWithBackend(purchase: Purchase) {
        CoroutineScope(Dispatchers.IO).launch {
            try {
                // Call your backend API
                val subscriptionId = purchase.products[0]
                val purchaseToken = purchase.purchaseToken

                // Example: Send to backend
                Log.d(TAG, "Verifying purchase with backend...")
                Log.d(TAG, "Subscription ID: $subscriptionId")
                Log.d(TAG, "Purchase Token: $purchaseToken")

                // TODO: Make actual API call to your backend
                // val response = apiService.verifyPurchase(subscriptionId, purchaseToken)

                // After backend verification and acknowledgment
                withContext(Dispatchers.Main) {
                    purchaseListener.onPurchaseSuccess(purchase)
                }

            } catch (e: Exception) {
                Log.e(TAG, "Error verifying purchase: ${e.message}")
                withContext(Dispatchers.Main) {
                    purchaseListener.onPurchaseFailure(e.message ?: "Verification failed")
                }
            }
        }
    }

    fun queryActivePurchases() {
        billingClient.queryPurchasesAsync(
            QueryPurchasesParams.newBuilder()
                .setProductType(BillingClient.ProductType.SUBS)
                .build()
        ) { billingResult, purchases ->
            if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                Log.d(TAG, "Active purchases: ${purchases.size}")
                purchases.forEach { purchase ->
                    Log.d(TAG, "Purchase: ${purchase.products[0]}")
                }
            }
        }
    }

    fun endConnection() {
        billingClient.endConnection()
    }

    companion object {
        private const val TAG = "BillingManager"
    }
}
```

#### Step 4.4: Use BillingManager in Activity

Create `MainActivity.kt`:

```kotlin
package com.example.subscriptions

import android.os.Bundle
import android.widget.Button
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import com.android.billingclient.api.Purchase

class MainActivity : AppCompatActivity(), BillingManager.PurchaseListener {

    private lateinit var billingManager: BillingManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Initialize billing
        billingManager = BillingManager(this, this)

        // Subscribe button
        findViewById<Button>(R.id.btnSubscribeMonthly).setOnClickListener {
            billingManager.launchSubscriptionFlow(this, "premium_monthly")
        }

        findViewById<Button>(R.id.btnSubscribeYearly).setOnClickListener {
            billingManager.launchSubscriptionFlow(this, "premium_yearly")
        }

        findViewById<Button>(R.id.btnCheckStatus).setOnClickListener {
            billingManager.queryActivePurchases()
        }
    }

    override fun onPurchaseSuccess(purchase: Purchase) {
        Toast.makeText(this, "Purchase successful!", Toast.LENGTH_SHORT).show()
        // Grant premium access to user
    }

    override fun onPurchaseFailure(errorMessage: String) {
        Toast.makeText(this, "Purchase failed: $errorMessage", Toast.LENGTH_SHORT).show()
    }

    override fun onDestroy() {
        super.onDestroy()
        billingManager.endConnection()
    }
}
```

#### Step 4.5: Create Backend API Service (Retrofit)

Create `ApiService.kt`:

```kotlin
package com.example.subscriptions.network

import retrofit2.Response
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.http.*

data class VerifyRequest(
    val subscriptionId: String,
    val purchaseToken: String
)

data class VerifyResponse(
    val success: Boolean,
    val isValid: Boolean,
    val subscription: Any?
)

interface ApiService {
    @POST("api/subscriptions/verify")
    suspend fun verifyPurchase(@Body request: VerifyRequest): Response<VerifyResponse>

    @POST("api/subscriptions/acknowledge")
    suspend fun acknowledgePurchase(@Body request: VerifyRequest): Response<Any>

    @GET("api/subscriptions/status")
    suspend fun getStatus(
        @Query("subscriptionId") subscriptionId: String,
        @Query("purchaseToken") purchaseToken: String
    ): Response<Any>

    companion object {
        private const val BASE_URL = "https://your-backend-domain.com/"

        fun create(): ApiService {
            val retrofit = Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .build()

            return retrofit.create(ApiService::class.java)
        }
    }
}
```

---

## API Documentation

### Base URL
```
http://localhost:3000
```

### Endpoints

#### 1. Verify Subscription
Verify a subscription purchase with Google Play.

**Endpoint:** `POST /api/subscriptions/verify`

**Request Body:**
```json
{
  "subscriptionId": "premium_monthly",
  "purchaseToken": "xxxxxxxxxxxxxxxxxxxxxxxx"
}
```

**Response (Success):**
```json
{
  "success": true,
  "isValid": true,
  "subscription": {
    "kind": "androidpublisher#subscriptionPurchase",
    "startTimeMillis": "1699000000000",
    "expiryTimeMillis": "1701592800000",
    "autoRenewing": true,
    "priceCurrencyCode": "USD",
    "priceAmountMicros": "4990000",
    "countryCode": "US",
    "paymentState": 1,
    "orderId": "GPA.1234-5678-9012-34567"
  }
}
```

#### 2. Acknowledge Subscription
Acknowledge a subscription purchase (required within 3 days).

**Endpoint:** `POST /api/subscriptions/acknowledge`

**Request Body:**
```json
{
  "subscriptionId": "premium_monthly",
  "purchaseToken": "xxxxxxxxxxxxxxxxxxxxxxxx"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Subscription acknowledged"
}
```

#### 3. Get Subscription Status
Check if a subscription is currently active.

**Endpoint:** `GET /api/subscriptions/status`

**Query Parameters:**
- `subscriptionId`: The subscription product ID
- `purchaseToken`: The purchase token

**Response:**
```json
{
  "success": true,
  "isActive": true,
  "expiryTime": "2024-12-03T00:00:00.000Z",
  "autoRenewing": true
}
```

#### 4. Cancel Subscription
Cancel a subscription (server-side).

**Endpoint:** `POST /api/subscriptions/cancel`

**Request Body:**
```json
{
  "subscriptionId": "premium_monthly",
  "purchaseToken": "xxxxxxxxxxxxxxxxxxxxxxxx"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Subscription cancelled"
}
```

#### 5. RTDN Webhook
Receive real-time notifications from Google Play (Pub/Sub push).

**Endpoint:** `POST /api/rtdn/webhook`

**Request Body (from Pub/Sub):**
```json
{
  "message": {
    "data": "base64-encoded-notification-data",
    "messageId": "123456789",
    "publishTime": "2024-11-10T10:00:00Z"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "Notification received and processed",
  "result": {
    "success": true,
    "eventType": "SUBSCRIPTION_PURCHASED",
    "subscriptionId": "premium_monthly",
    "purchaseToken": "xxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

#### 6. Test RTDN Webhook
Test the RTDN webhook with a sample notification.

**Endpoint:** `POST /api/rtdn/test`

**Response:**
```json
{
  "success": true,
  "message": "Test notification processed",
  "result": {
    "success": true,
    "eventType": "SUBSCRIPTION_PURCHASED",
    "subscriptionId": "premium_monthly",
    "purchaseToken": "test-purchase-token-123"
  }
}
```

---

## RTDN Implementation

### What is RTDN?

Real-Time Developer Notifications (RTDN) is a Google Cloud Pub/Sub-based system that sends notifications about subscription state changes in real-time.

### Benefits of RTDN

1. **Real-time updates**: Know immediately when subscriptions change
2. **Reduced API calls**: No need to poll Google Play API constantly
3. **Better user experience**: Grant/revoke access instantly
4. **Fraud detection**: Detect refunds and cancellations quickly
5. **Analytics**: Track subscription lifecycle events

### RTDN Notification Flow

```
┌──────────────────┐
│  Google Play     │
│  (User Action)   │
└────────┬─────────┘
         │
         │ Subscription Event
         ▼
┌──────────────────────┐
│  Cloud Pub/Sub       │
│  Topic: play-rtdn    │
└────────┬─────────────┘
         │
         │ Push Notification
         ▼
┌──────────────────────────┐
│  Your Backend            │
│  POST /api/rtdn/webhook  │
└────────┬─────────────────┘
         │
         │ Process Event
         ▼
┌──────────────────────────┐
│  Update Database         │
│  Grant/Revoke Access     │
│  Send User Notification  │
└──────────────────────────┘
```

### Notification Structure

Decoded RTDN message structure:
```json
{
  "version": "1.0",
  "packageName": "com.example.yourapp",
  "eventTimeMillis": "1699000000000",
  "subscriptionNotification": {
    "version": "1.0",
    "notificationType": 4,
    "purchaseToken": "xxxxxxxxxxxxxxxxxxxxxxxx",
    "subscriptionId": "premium_monthly"
  }
}
```

### Notification Types Reference

| Type | Code | Description | Action Required |
|------|------|-------------|-----------------|
| SUBSCRIPTION_RECOVERED | 1 | Subscription recovered from account hold | Grant access |
| SUBSCRIPTION_RENEWED | 2 | Active subscription renewed | Extend access |
| SUBSCRIPTION_CANCELED | 3 | Subscription voluntarily canceled | Revoke at expiry |
| SUBSCRIPTION_PURCHASED | 4 | New subscription purchased | Grant access |
| SUBSCRIPTION_ON_HOLD | 5 | Subscription on hold (payment issue) | Maintain access temporarily |
| SUBSCRIPTION_IN_GRACE_PERIOD | 6 | In grace period (payment failed) | Send payment reminder |
| SUBSCRIPTION_RESTARTED | 7 | Subscription restarted | Grant access |
| SUBSCRIPTION_PRICE_CHANGE_CONFIRMED | 8 | User accepted price change | No action needed |
| SUBSCRIPTION_DEFERRED | 9 | Subscription renewal deferred | Update expiry date |
| SUBSCRIPTION_PAUSED | 10 | Subscription paused by user | Suspend access |
| SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED | 11 | Pause schedule changed | Update schedule |
| SUBSCRIPTION_REVOKED | 12 | Subscription revoked (refund) | Revoke access immediately |
| SUBSCRIPTION_EXPIRED | 13 | Subscription expired | Revoke access |

---

## Testing

### Test the Backend API

#### 1. Health Check
```bash
curl http://localhost:3000/health
```

#### 2. Test RTDN Webhook
```bash
curl -X POST http://localhost:3000/api/rtdn/test
```

#### 3. Verify Subscription (requires real purchase token)
```bash
curl -X POST http://localhost:3000/api/subscriptions/verify \
  -H "Content-Type: application/json" \
  -d '{
    "subscriptionId": "premium_monthly",
    "purchaseToken": "your-purchase-token-here"
  }'
```

### Test on Android

#### Using License Testing
1. Add your test account to **License Testing** in Play Console
2. Install the app on a test device with that account
3. Make a test purchase (you won't be charged)
4. Verify the purchase token is sent to your backend

#### Testing RTDN
1. Make a test purchase
2. Check your backend logs for RTDN notifications
3. Test different scenarios:
   - Purchase
   - Cancel subscription
   - Let subscription expire (for testing, set short billing period)

### Send Test Notification from Play Console
1. Go to **Play Console → Monetization setup**
2. Under **Real-time developer notifications**
3. Click **Send test notification**
4. Check your backend receives it at `/api/rtdn/webhook`

---

## Flow Diagrams

### Purchase Flow

```
User (Android App)          Backend Server          Google Play API
      │                          │                         │
      │ 1. Click Subscribe       │                         │
      │────────────────────────> │                         │
      │                          │                         │
      │ 2. Launch Billing Flow   │                         │
      │<──────────────────────── │                         │
      │                          │                         │
      │ 3. Complete Purchase     │                         │
      │──────────────────────────────────────────────────> │
      │                          │                         │
      │ 4. Purchase Token        │                         │
      │<────────────────────────────────────────────────── │
      │                          │                         │
      │ 5. Send Token to Backend │                         │
      │─────────────────────────>│                         │
      │                          │                         │
      │                          │ 6. Verify Purchase      │
      │                          │────────────────────────>│
      │                          │                         │
      │                          │ 7. Purchase Details     │
      │                          │<────────────────────────│
      │                          │                         │
      │                          │ 8. Acknowledge Purchase │
      │                          │────────────────────────>│
      │                          │                         │
      │ 9. Grant Access          │                         │
      │<─────────────────────────│                         │
```

### RTDN Flow

```
Google Play          Cloud Pub/Sub          Backend Server          Database
     │                    │                        │                    │
     │ Subscription Event │                        │                    │
     │───────────────────>│                        │                    │
     │                    │                        │                    │
     │                    │ Push Notification      │                    │
     │                    │───────────────────────>│                    │
     │                    │                        │                    │
     │                    │                        │ Decode Message     │
     │                    │                        │──────────┐         │
     │                    │                        │          │         │
     │                    │                        │<─────────┘         │
     │                    │                        │                    │
     │                    │                        │ Update User Status │
     │                    │                        │───────────────────>│
     │                    │                        │                    │
     │                    │ 200 OK                 │                    │
     │                    │<───────────────────────│                    │
```

---

## Troubleshooting

### Common Issues

#### 1. "Service account not linked" Error
**Problem:** Backend can't access Google Play API

**Solution:**
- Ensure service account is created in Google Cloud Console
- Service account email is added to Google Play Console API access
- Service account has "Financial data" permissions
- `service-account-key.json` is correctly placed in backend directory

#### 2. "Purchase token not found" Error
**Problem:** Backend can't verify purchase

**Solution:**
- Ensure you're using a real purchase (not a test token)
- Purchase is for the correct package name
- Purchase hasn't been refunded or expired

#### 3. RTDN Notifications Not Received
**Problem:** Backend doesn't receive notifications

**Solution:**
- Verify Pub/Sub topic is correctly configured in Play Console
- Check push subscription endpoint URL is correct and publicly accessible
- Endpoint must use HTTPS (not HTTP)
- Test with "Send test notification" in Play Console
- Check backend logs for errors

#### 4. "Billing client not ready" Error
**Problem:** Android app billing initialization failed

**Solution:**
- Ensure app is signed with the release key uploaded to Play Console
- App version code matches the one in Play Console
- License testing account is configured
- Device has Google Play Services

#### 5. Acknowledgment Failures
**Problem:** Purchase not acknowledged within 3 days

**Solution:**
- Call acknowledge API immediately after verification
- Implement retry logic for acknowledgment
- Monitor RTDN for `SUBSCRIPTION_REVOKED` events

### Debug Checklist

- [ ] Service account created and key downloaded
- [ ] Service account linked to Play Console with proper permissions
- [ ] Environment variables configured correctly
- [ ] Package name matches exactly
- [ ] Subscription products created in Play Console
- [ ] Products are active (not draft)
- [ ] Test account added to license testers
- [ ] App signed with release key
- [ ] Pub/Sub topic created
- [ ] Push subscription configured with correct endpoint
- [ ] Backend publicly accessible with HTTPS
- [ ] Billing library added to Android app

---

## Best Practices

### Security

1. **Never trust client-side data**
   - Always verify purchases server-side
   - Don't grant access based only on client claims

2. **Secure your service account key**
   - Never commit `service-account-key.json` to version control
   - Add to `.gitignore`
   - Use environment variables in production
   - Rotate keys periodically

3. **Verify Pub/Sub requests**
   - Implement JWT token verification for RTDN webhooks
   - Ensure requests are from Google's servers

4. **Use HTTPS**
   - Always use HTTPS for backend API
   - Required for Pub/Sub push subscriptions

### Performance

1. **Cache subscription status**
   - Store subscription states in database
   - Update via RTDN instead of polling
   - Reduce Google Play API calls

2. **Implement retry logic**
   - Handle transient API failures
   - Retry acknowledgments
   - Queue RTDN processing

3. **Async processing**
   - Process RTDN notifications asynchronously
   - Return 200 OK immediately to Pub/Sub
   - Use message queues for complex processing

### User Experience

1. **Handle edge cases**
   - Grace period: Maintain access but warn user
   - Account hold: Give user time to update payment
   - Canceled subscriptions: Maintain access until expiry

2. **Acknowledge within 3 days**
   - Purchases must be acknowledged
   - Unacknowledged purchases are refunded after 3 days
   - Implement automated acknowledgment

3. **Provide clear feedback**
   - Show loading states during purchase
   - Handle cancellations gracefully
   - Display subscription status clearly

### Monitoring

1. **Log all events**
   - Log all RTDN notifications
   - Log API calls and responses
   - Monitor acknowledgment success rate

2. **Set up alerts**
   - Alert on repeated verification failures
   - Monitor RTDN webhook downtime
   - Track refund rates

3. **Analytics**
   - Track subscription conversion rates
   - Monitor cancellation reasons
   - Analyze renewal rates

---

## Additional Resources

### Official Documentation
- [Google Play Billing Library](https://developer.android.com/google/play/billing)
- [Google Play Developer API](https://developers.google.com/android-publisher)
- [Real-Time Developer Notifications](https://developer.android.com/google/play/billing/rtdn-reference)
- [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs)

### Testing Resources
- [Test Google Play Billing](https://developer.android.com/google/play/billing/test)
- [License Testing](https://support.google.com/googleplay/android-developer/answer/6062777)

### Tools
- [ngrok](https://ngrok.com/) - Expose local server for testing
- [Postman](https://www.postman.com/) - API testing
- [Google Play Console](https://play.google.com/console)
- [Google Cloud Console](https://console.cloud.google.com/)

---

## Support and Contribution

### Getting Help
- Check the troubleshooting section
- Review official Google documentation
- Test with license testing before production

### Important Notes
- This is a POC implementation for learning purposes
- Add proper error handling for production use
- Implement database persistence for subscription states
- Add user authentication/authorization
- Implement proper logging and monitoring
- Add unit and integration tests
- Follow Google Play policies and guidelines

---

## License

MIT License - Free to use for learning and commercial projects

---

## Quick Start Summary

1. **Google Play Console**: Create app, add subscriptions, configure RTDN
2. **Google Cloud**: Enable API, create service account, set up Pub/Sub
3. **Backend**: Install dependencies, configure .env, start server
4. **Android**: Add billing library, implement BillingManager, test purchase
5. **Test**: Use license testing, verify RTDN, check backend logs

**Total Setup Time**: ~2-3 hours

---

**Happy Coding!** If you encounter any issues, refer to the troubleshooting section or official documentation.
