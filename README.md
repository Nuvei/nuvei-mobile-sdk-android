## Nuvei SimplyConnect Android SDK

### Requirements

| Platform       | Minimum API Level | Minimum Kotlin Version | Installation            |
| -------------- | ----------------- | ---------------------- | ----------------------- |
| Android 5.0+   | API 21            | 1.5.0+                 | Gradle / Maven Central  |

---

## Installation

### Gradle (Recommended)

Add the Nuvei Maven repository (if not already present):

```groovy
allprojects {
    repositories {
        mavenCentral()
            maven {
            url = uri("https://raw.githubusercontent.com/Nuvei/nuvei-maven-android/master")
        }
    }
}
```

Then in your app module `build.gradle`:

```groovy
dependencies {
    // Core module (mandatory)
    implementation("com.nuvei.mobile.simply-connect-sdk:core:1.3.0")
    
    // Submodules
    implementation("com.nuvei.mobile.simply-connect-sdk:googlepay:1.3.0") // For native Google Pay payments
    implementation("com.nuvei.mobile.simply-connect-sdk:fields:1.3.0")
    implementation("com.nuvei.mobile.simply-connect-sdk:simplyconnect:1.3.0")
}
```

### Manual (AAR)

1. Download the `.aar` files for each Nuvei module you need.
2. Place them under a folder, e.g. `libs/nuvei/` in your module.
3. In your module `build.gradle`:

```groovy
repositories {
    flatDir {
        dirs 'libs/nuvei'
    }
}

dependencies {
    implementation(name: 'nuvei-core-1.3.0', ext: 'aar')
    implementation(name: 'nuvei-googlepay-1.3.0', ext: 'aar')
    implementation(name: 'nuvei-simplyconnect-1.3.0', ext: 'aar')
    implementation(name: 'nuvei-fields-1.3.0', ext: 'aar')
}
```

---

## Setup

Before making any calls, configure the environment:

```kotlin
// e.g. in Application.onCreate()

// Setup App
MerchantServer.baseUrl = appEnvironment.merchantBaseUrl
// Setup Nuvei
Nuvei.setup(context, Nuvei.Environment.QA)
// or STAGING / PROD
```

---

## Usage

### 1. Open Order

Build your order request and send it via OkHttp or Retrofit. Example with OkHttp:

```kotlin
// Create open order request

// Get country based on the SIM carrier 
val countryCode = (getSystemService(Context.TELEPHONY_SERVICE) as? TelephonyManager)?.networkCountryIso?.uppercase()

// Setup the Billing Address
val billingAddress = BillingAddress(
                country = if (countryCode.isNullOrBlank()) "US" else countryCode,
                email = "test@user.com")

// Create the Order object itself
  val order = Order(
                "amount",
                "currency",
                "userTokenId",
                "merchantId",
                "merchantSiteId",
                "secret",
                billingAddress,
                items
            )

fun openOrder(
    input: Order,
    callback: (OrderResponse?, Throwable?) -> Unit
) {
    val json = gson.toJson(input)
    val body = RequestBody.create(
        MediaType.parse("application/json"), json
    )

    val request = Request.Builder()
        .url(Nuvei.getEnvBaseUrl() + "openOrder.do")
        .post(body)
        .build()

    OkHttpClient().newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            callback(null, e)
        }
        override fun onResponse(call: Call, response: Response) {
            val respObj = gson.fromJson(response.body?.string(), OrderResponse::class.java)
            callback(respObj, null)
        }
    })
}
```

Check the response for a non-null `sessionToken` then proceed to payment.

## Google Pay

1. Initialize handler and button:

Add the Google Pay Button view to your XML
```xml
<com.google.android.gms.wallet.button.PayButton
  android:id="@+id/googlePayButton"
  android:layout_width="match_parent"
  android:layout_height="wrap_content"/>
```

and initialize it

```kotlin
val googlePayHandler = GooglePayHandler(activity) //Pass activity of fragment
googlePayHandler.initializeButton(R.id.googlePayButton)
GooglePayHandler.auth3DSupport = true // pass true or false :Boolean
```

2. Prepare payment data:

```kotlin
val nvPayment = NVPayment(
    amount = 99.99,
    currency = "EUR",
    sessionToken = "..."
)
```

3. On click:

```kotlin
findViewById<Button>(R.id.googlePayButton)
    .setOnClickListener {
        val token = googlePayHandler.openGooglePay(nvPayment)
        if (token != null) {
            // success
        } else {
            // handle error
        }
    }
```

## SimplyConnect
```kotlin
SimplyConnect.installments.options = listOf(
    Installments.Option(Installments.Type.DEFERRED_WITH_INTEREST, intArrayOf(2,4,6)),
    Installments.Option(Installments.Type.SINGLE_PAYMENT, intArrayOf(1))
)
SimplyConnect.installments.requirePersonalID = true // if needed
```
This creates instance of Installment with installment options. You can choose which options you will add. If the option is not `SINGLE_PAYMENT` you have to pass list of periods from which the user can choose. You can choose if the national id is required. It depends on the country code. Country codes that we support national id are AR, BR, CL, PE, CO, MX, PY, UY, IL. It is optional.

```kotlin
SimplyConnect.checkout(nvPayment)
```
Before showing the checkout, customize UI texts:

```kotlin
with(CheckoutI18N) {
    cardHolderNameTitle = "Cardholder Name"
    // ... all other properties as needed
}
```
This creates instance of `CheckoutI18n` with your own localized texts. You can initialize it with the properties and text you want to modify.
If you skip this step, default English strings apply.

For input you must create instance of `NVInput` with the payment data. The callback tells you when the proccess is finished either successfully or with an error. `DeclineFallbackDecision` tells you when there is an error.

---
## Fields

1. Add the NuveiCreditCardField view to your XML
```xml
<com.nuvei.mobilesdk.fields.NuveiCreditCardField
            android:id="@+id/creditCardField"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
```
2. Apply customization
```kotlin
 binding.creditCardField.applyCustomization(NuveiFieldCustomization::class)
```

Customize UI texts:

```kotlin
with(FieldsI18N) {
    cardHolderNameTitle = "Cardholder Name"
    // ... all other properties as needed
}
```

```kotlin
        creditCardField.transactionDetails = transactionDetails
                creditCardField.onInputUpdated = { hasFocus -> }

                creditCardField.onInputValidated = { errors -> }

                creditCardField.onCardDetailsUpdate = { output, error ->
                 creditCardField.showCardNumberError("ERROR MESSAGE")
                }
```

This creates instance of `NuveiCreditCardField`.
`OnInputUpdated` is callback that tells you when a field text is modified and when it loses focus.
`OnInputValidated` is callback that gives you list of error if there is any after validation of a field.
`OnCardDetailsUpdated` is callback that gives you detailed information about the card after the card number is valid. Then based on your requirements you can call `creditCardField.showCardNumberError(message:)` and your error message will be shown under the card number field.

```kotlin
creditCardField?.validate()

creditCardField.tokenize(this@Activity, transactionDetails, object :TokenizeCallback {
                        override fun onComplete(token: String?, error: Error?, additionalInfo: Map<String, Any>?) {
                            if (error == null) {
                                if (token != null) {
                                    // do something
                                }
                            } else {
                                // do something
                            }
                            // do something
                        }
                    })
creditCardField.createPayment(this@FieldsPresenterActivity, transactionDetails, object :Callback<NVCreatePaymentOutput> {
                        override fun onComplete(response: NVCreatePaymentOutput) {
                            response.rawResult?.let {
                                // do something
                            }
                        }
                    }) { response, activity, declineFallback ->
                        // do something
                    }

```
`Validate` method will validate the card filled in the fields. The closure tells you whether the validation was successful or there was some error.
`Tokenize` method will create token with which you can create payment. In the closure you can get the token or handle an error if there is so.
`CreditCardField.createPayment` method will create payment with the card that the user have filled in. The callback tells you when the proccess is finished either successfully or with an error. `DeclineFallbackDecision` tells you when there is an error.

```kotlin
 val paymentOption = PaymentOption(null,
            CardDetails(
                ccTempToken = "temp token string",
                cardHolderName = "Ivan Petrov"
                )
            )
```

NuveiFields.createPayment will create a payment with a token. For this method you don't need the `Fields` view . You have to pass an object of type `NVPayment` in which you have to put the token and optionally the card holder name(`PaymentOption` data model).

---
## Checksum Utility

Generate your `checksum` exactly as on server:

```kotlin
fun getCheckSum(
    merchantId: String,
    merchantSiteId: String,
    clientRequestId: String,
    amount: String,
    currency: String,
    timeStamp: String,
    secret: String
): String {
    val raw = merchantId + merchantSiteId + clientRequestId + amount + currency + timeStamp + secret
    val digest = MessageDigest.getInstance("SHA-256").digest(raw.toByteArray())
    return digest.joinToString("") { "%02x".format(it) }
}
```

---

## 📝 Notes

- Use `payButtonTitle = "Pay %1.2f %2s"` to display amount and currency (e.g. "Pay 99.99 EUR").
- Ensure `sessionToken` from `openOrder.do` is valid before calling any payment flow.
- All network calls should be off the UI thread (e.g. coroutines on `Dispatchers.IO`).

