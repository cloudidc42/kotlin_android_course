# Part 71: Payment Integration
## ขั้นตอนที่ 1251-1275

---

## ขั้นตอนที่ 1251: Payment Options

```
Payment Methods ใน Android:

1. Google Pay - simplest, uses stored cards
2. Stripe SDK - full payment processing
3. PromptPay/TrueWallet - Thai payment
4. PayPal SDK
5. In-App Billing (Google Play Billing) - subscriptions

Security Requirements:
- Never store raw card data
- Use tokenization
- PCI DSS compliance
- SSL/TLS for all transactions
```

---

## ขั้นตอนที่ 1252: Google Pay

```kotlin
// build.gradle.kts
// implementation("com.google.android.gms:play-services-wallet:19.x")

class GooglePayManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val paymentsClient: PaymentsClient by lazy {
        Wallet.getPaymentsClient(
            context,
            Wallet.WalletOptions.Builder()
                .setEnvironment(WalletConstants.ENVIRONMENT_TEST)  // Use PRODUCTION for release
                .build()
        )
    }
    
    suspend fun isReadyToPay(): Boolean {
        val request = IsReadyToPayRequest.fromJson(
            """
            {
                "apiVersion": 2,
                "apiVersionMinor": 0,
                "allowedPaymentMethods": [
                    {
                        "type": "CARD",
                        "parameters": {
                            "allowedAuthMethods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
                            "allowedCardNetworks": ["AMEX", "MASTERCARD", "VISA"]
                        }
                    }
                ]
            }
            """.trimIndent()
        )
        
        return paymentsClient.isReadyToPay(request).await()
    }
    
    fun createPaymentDataRequest(totalPrice: String): PaymentDataRequest {
        val paymentDataRequestJson = """
        {
            "apiVersion": 2,
            "apiVersionMinor": 0,
            "allowedPaymentMethods": [
                {
                    "type": "CARD",
                    "parameters": {
                        "allowedAuthMethods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
                        "allowedCardNetworks": ["AMEX", "MASTERCARD", "VISA"]
                    },
                    "tokenizationSpecification": {
                        "type": "PAYMENT_GATEWAY",
                        "parameters": {
                            "gateway": "stripe",
                            "stripe:version": "2022-11-15",
                            "stripe:publishableKey": "pk_test_xxxx"
                        }
                    }
                }
            ],
            "transactionInfo": {
                "totalPriceStatus": "FINAL",
                "totalPrice": "$totalPrice",
                "currencyCode": "THB"
            },
            "merchantInfo": {
                "merchantName": "My Shop"
            }
        }
        """.trimIndent()
        
        return PaymentDataRequest.fromJson(paymentDataRequestJson)
    }
}

// Activity Result Launcher
class CheckoutActivity : ComponentActivity() {
    
    private val googlePayLauncher = registerForActivityResult(
        ActivityResultContracts.StartIntentSenderForResult()
    ) { result ->
        when (result.resultCode) {
            RESULT_OK -> {
                val data = result.data
                val paymentData = PaymentData.getFromIntent(data!!)
                handlePaymentSuccess(paymentData)
            }
            RESULT_CANCELED -> handlePaymentCanceled()
        }
    }
    
    private fun handlePaymentSuccess(paymentData: PaymentData?) {
        val paymentInfo = paymentData?.toJson() ?: return
        // Send payment token to backend
        val token = JSONObject(paymentInfo)
            .getJSONObject("paymentMethodData")
            .getJSONObject("tokenizationData")
            .getString("token")
        
        viewModel.processPayment(token)
    }
}
```

---

## ขั้นตอนที่ 1253: Stripe Integration

```kotlin
// implementation("com.stripe:stripe-android:20.x")

class StripePaymentManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val backendService: BackendService
) {
    
    private val stripe = Stripe(context, "pk_test_xxxx")
    
    // Create PaymentIntent on backend, confirm on client
    suspend fun processPayment(
        amount: Long,  // in smallest currency unit (satang for THB)
        currency: String = "thb",
        customerId: String? = null
    ): PaymentResult {
        
        return try {
            // 1. Create PaymentIntent on your backend
            val clientSecret = backendService.createPaymentIntent(
                amount = amount,
                currency = currency,
                customerId = customerId
            ).clientSecret
            
            // 2. Confirm payment
            val confirmParams = ConfirmPaymentIntentParams.createWithPaymentMethodId(
                paymentMethodId = "pm_card_visa",  // from Stripe Element
                clientSecret = clientSecret
            )
            
            val paymentIntent = stripe.confirmPayment(confirmParams).await()
            
            when (paymentIntent.status) {
                StripeIntent.Status.Succeeded -> PaymentResult.Success(paymentIntent.id!!)
                StripeIntent.Status.RequiresAction -> PaymentResult.RequiresAction(clientSecret)
                else -> PaymentResult.Failed("Payment status: ${paymentIntent.status}")
            }
            
        } catch (e: Exception) {
            PaymentResult.Failed(e.message ?: "Payment failed")
        }
    }
    
    // Save card for future use
    suspend fun saveCard(cardNumber: String, expMonth: Int, expYear: Int, cvc: String): String {
        val card = CardParams(cardNumber, expMonth, expYear, cvc)
        val paymentMethod = stripe.createPaymentMethod(
            PaymentMethodCreateParams.create(card.toPaymentMethodParamsCard())
        ).await()
        
        return paymentMethod?.id ?: throw Exception("Failed to save card")
    }
}

sealed class PaymentResult {
    data class Success(val paymentIntentId: String) : PaymentResult()
    data class RequiresAction(val clientSecret: String) : PaymentResult()
    data class Failed(val error: String) : PaymentResult()
}
```

---

## ขั้นตอนที่ 1254: Google Play Billing (Subscriptions)

```kotlin
// implementation("com.android.billingclient:billing-ktx:7.x")

class BillingManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val backendService: BackendService
) {
    
    private var billingClient: BillingClient? = null
    
    private val _purchaseUpdates = MutableSharedFlow<PurchaseUpdate>()
    val purchaseUpdates: SharedFlow<PurchaseUpdate> = _purchaseUpdates.asSharedFlow()
    
    fun initialize() {
        billingClient = BillingClient.newBuilder(context)
            .setListener { billingResult, purchases ->
                CoroutineScope(Dispatchers.IO).launch {
                    handlePurchaseUpdate(billingResult, purchases)
                }
            }
            .enablePendingPurchases()
            .build()
        
        billingClient?.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(billingResult: BillingResult) {
                if (billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
                    queryPurchases()
                }
            }
            
            override fun onBillingServiceDisconnected() {
                // Retry connection
            }
        })
    }
    
    suspend fun getSubscriptionProducts(): List<ProductDetails> {
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(listOf(
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId("premium_monthly")
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build(),
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId("premium_yearly")
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build()
            ))
            .build()
        
        val result = billingClient?.queryProductDetails(params)
        return result?.productDetailsList ?: emptyList()
    }
    
    fun subscribe(activity: Activity, productDetails: ProductDetails) {
        val offerToken = productDetails.subscriptionOfferDetails?.first()?.offerToken ?: return
        
        val params = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(
                BillingFlowParams.ProductDetailsParams.newBuilder()
                    .setProductDetails(productDetails)
                    .setOfferToken(offerToken)
                    .build()
            ))
            .build()
        
        billingClient?.launchBillingFlow(activity, params)
    }
    
    private suspend fun handlePurchaseUpdate(
        billingResult: BillingResult,
        purchases: List<Purchase>?
    ) {
        if (billingResult.responseCode != BillingClient.BillingResponseCode.OK) {
            _purchaseUpdates.emit(PurchaseUpdate.Error(billingResult.debugMessage))
            return
        }
        
        purchases?.forEach { purchase ->
            if (purchase.purchaseState == Purchase.PurchaseState.PURCHASED) {
                // Verify with backend
                val verified = backendService.verifyPurchase(
                    purchaseToken = purchase.purchaseToken,
                    productId = purchase.products.first()
                )
                
                if (verified) {
                    // Acknowledge purchase
                    billingClient?.acknowledgePurchase(
                        AcknowledgePurchaseParams.newBuilder()
                            .setPurchaseToken(purchase.purchaseToken)
                            .build()
                    )
                    
                    _purchaseUpdates.emit(PurchaseUpdate.Success(purchase))
                }
            }
        }
    }
    
    private fun queryPurchases() {
        billingClient?.queryPurchasesAsync(
            QueryPurchasesParams.newBuilder()
                .setProductType(BillingClient.ProductType.SUBS)
                .build()
        ) { _, purchases ->
            // Process any unacknowledged purchases
        }
    }
}

sealed class PurchaseUpdate {
    data class Success(val purchase: Purchase) : PurchaseUpdate()
    data class Error(val message: String) : PurchaseUpdate()
}
```

---

## แบบฝึกหัด Part 71

```kotlin
// แบบฝึกหัด: Payment Flow ใน Compose

// TODO: สร้าง PaymentScreen ที่:
// 1. แสดง order summary
// 2. เลือกวิธีชำระเงิน (Google Pay, Credit Card, QR PromptPay)
// 3. ถ้า Credit Card → input form พร้อม validation
// 4. ปุ่ม "ยืนยันการชำระเงิน" ที่ disable จนกว่าจะเลือก method
// 5. Loading state ระหว่าง processing
// 6. Success/Error screen

sealed class PaymentMethod {
    object GooglePay : PaymentMethod()
    data class CreditCard(val number: String, val expiry: String, val cvv: String) : PaymentMethod()
    object PromptPay : PaymentMethod()
}

@Composable
fun PaymentScreen(
    orderTotal: Double,
    onPaymentSuccess: (String) -> Unit,
    onPaymentFailed: (String) -> Unit,
    viewModel: PaymentViewModel = hiltViewModel()
) {
    // TODO: implement payment UI
}
```

---

*Part 71 จบแล้ว | ก่อนหน้า: [Part 70](../part70/README.md) | ถัดไป: [Part 72](../part72/README.md)*
