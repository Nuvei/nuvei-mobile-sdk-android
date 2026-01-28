## Nuvei SimplyConnect Android SDK

### Requirements

| Platform       | Minimum API Level | Minimum Kotlin Version | Installation            |
| -------------- | ----------------- | ---------------------- | ----------------------- |
| Android 5.0+   | API 21            | 1.5.0+                 | Gradle / Maven Central  |

---

## Installation

### Gradle (Recommended)

Add the Nuvei Maven repository to the settings.gradle dependencyResolutionManagement block

```groovy
pluginManagement {
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
        maven {                                                                                 ///
            url = uri("https://raw.githubusercontent.com/Nuvei/nuvei-maven-android/master")     /// Optional!
        }                                                                                       ///
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven {                                                                                             ///
            url = uri("https://raw.githubusercontent.com/Nuvei/nuvei-mobile-simply-connect-android/master") /// Add this Nuvei Repository
            content {                                                                                       ///
                includeGroup("com.nuvei.mobile.sdk")                                         /// and its content
            }                                                                                               ///
        }                                                                                                   ///
    }
}
```

Then in your app module `build.gradle`:

```groovy
dependencies {
    // Core module (mandatory)
    implementation("com.nuvei.mobile.sdk:core:1.4.1")

    // Submodules
    implementation("com.nuvei.mobile.sdk:googlepay:1.4.1") // For native Google Pay payments
    implementation("com.nuvei.mobile.sdk:fields:1.4.1")
    implementation("com.nuvei.mobile.sdk:simplyconnect:1.4.1")
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
    implementation(name: 'nuvei-core-1.4.1', ext: 'aar')
    implementation(name: 'nuvei-googlepay-1.4.1', ext: 'aar')
    implementation(name: 'nuvei-simplyconnect-1.4.1', ext: 'aar')
    implementation(name: 'nuvei-fields-1.4.1', ext: 'aar')
}
```

---

## Setup

Before making any calls, configure the environment:

```kotlin
// e.g. in Application.onCreate()

// Setup Nuvei
Nuvei.setup(context, Nuvei.Environment.QA)
// or STAGING / PROD 
```

---

## Usage

### Open Order

Build your order request and send it via OkHttp or Retrofit. Example with OkHttp:
#### ⚠️ Important - BillingAddress(), Item(), Order() and OrderResponse() are example data objects which are not part of the SDK.


```kotlin
// Create open order request

//Order parameters
val amount: String = "153"
val currency: String = "EUR"
val userTokenId: String = "User token Id"
val merchantId: String = "Merchant Id"
val merchantSiteId: String = "Merchant site Id"
val secret: String = "Secret"
val clientRequestId: String = UUID.randomUUID().toString()
val timeStamp: String = SimpleDateFormat("yyyyMMddHHmmss", Locale.US).format(Date())
val checksum: String = createChecksum(amount, currency, merchantId, merchantSiteId, clientRequestId, timeStamp, secret)

// Get country based on the SIM carrier
val countryCode = (getSystemService(Context.TELEPHONY_SERVICE) as? TelephonyManager)?.networkCountryIso?.uppercase()

// Setup the Billing Address
val billingAddress = BillingAddress(
    country = if (countryCode.isNullOrBlank()) "US" else countryCode,
    email = "test@user.com")

val items = arrayListOf<Item>()
items.add(Item(
    price = amount,
    quantity = "1",
    name = "Test"
))

class DeviceDetails(
    val deviceType: String? = null,
    val deviceName: String? = null,
    val deviceOS: String? = null,
    val ipAddress: String? = null
)

object DeviceDetailsProvider {

    fun collect(context: Context): DeviceDetails {
        return DeviceDetails(
            deviceType  = context.getDeviceTypeString(),
            deviceName  = buildDeviceName(),
            deviceOS    = buildOsString(),
            ipAddress   = HostAppDetails.getIPAddress(true)
        )
    }

    private fun buildDeviceName(): String {
        val manufacturer = Build.MANUFACTURER?.trim().orEmpty()
        val model = Build.MODEL?.trim().orEmpty()
        val name = (manufacturer + " " + model).trim()
        return name.ifBlank { null } ?: "Android Device"
    }

    private fun buildOsString(): String {
        val release = Build.VERSION.RELEASE ?: "?"
        val sdk = Build.VERSION.SDK_INT
        return "Android $release (SDK $sdk)"
    }

    fun Context.getDeviceTypeString(): String {
        val uiModeManager = getSystemService(Context.UI_MODE_SERVICE) as? UiModeManager
        val modeType = uiModeManager?.currentModeType

        return when (modeType) {
            Configuration.UI_MODE_TYPE_TELEVISION -> "TV"
            Configuration.UI_MODE_TYPE_CAR -> "AUTO"
            Configuration.UI_MODE_TYPE_WATCH -> "WATCH"
            else -> {
                val sw = resources.configuration.smallestScreenWidthDp
                if (sw >= 600) "TABLET" else "SMARTPHONE"
            }
        }
    }
}

// Create the Order object itself
val order = Order(
    amount,
    currency,
    userTokenId,
    merchantId,
    merchantSiteId,
    secret,
    clientRequestId,
    timeStamp,
    checksum,
    billingAddress,
    items,
    DeviceDetailsProvider.collect(context)
)

// Make the request
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

    // Example OrderResponse data object
    data class OrderResponse(
        val sessionToken: String = "",
        val merchantId: String = "",
        val merchantSiteId: String = "",
        val clientRequestId: String = "",
        val internalRequestId: Int = 0,
        val status: String = "",
        val errCode: Int = 0,
        val reason: String = "",
        val version: String = "",
        val orderId: Int = 0,
        val userTokenId: String = ""
    )


    fun createChecksum(
        amount: String,
        currency: String,
        merchantId: String,
        merchantSiteId: String,
        clientRequestId: String,
        timeStamp: String,
        secret: String): String {

        val str = "$merchantId$merchantSiteId$clientRequestId$amount$currency$timeStamp$secret"

        val digest = MessageDigest.getInstance("SHA-256")
        digest.reset()
        val byteData = digest.digest(str.toByteArray(Charset.forName("UTF-8")))
        val sb = StringBuffer()

        for (byte in byteData) {
            val hex = Integer.toHexString(0xff and byte.toInt())
            if (hex.length == 1) sb.append('0')
            sb.append(hex)
        }
        return sb.toString()
    }
}
```

Check the response for a non-null `sessionToken` then proceed to payment.

---
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
val googlePayHandler = GooglePayHandler(activity) //Pass activity or fragment
googlePayHandler.initializeButton(R.id.googlePayButton)
GooglePayHandler.auth3DSupport = true // pass true or false :Boolean
```

2. Prepare payment object:

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
        googlePayHandler.openGooglePay(nvPayment) {
                result ->
            // Use the result object of type  NVCreatePaymentOutput to verify the payment status
        }
    }
```
---
## SimplyConnect
Imports needed for Authenticate3D:
```kotlin
import com.nuvei.mobilesdk.core.Nuvei
import com.nuvei.mobilesdk.core.model.NVPayment
import com.nuvei.mobilesdk.core.model.NVAuthenticate3dOutput
```
```kotlin
Nuvei.authenticate3d(
    activity: Activity,
    input: NVPayment,
    additionalParams: Map<String, Any>?,
    forceWebChallenge: Boolean,
    source: String,
    callback: Callback<NVAuthenticate3dOutput> object : Callback<NVAuthenticate3dOutput>{
    override fun onComplete(response: NVAuthenticate3dOutput) {
        //do something
    }
}
)
```

Authenticate3d will authenticate a payment. For input you must create instance of NVPayment with the payment data.

Imports needed for InternalCreatePayment:
```kotlin
import com.nuvei.mobilesdk.core.Nuvei
import com.nuvei.mobilesdk.core.model.NVPayment
import com.nuvei.mobilesdk.core.model.NVCreatePaymentOutput
```
```kotlin
Nuvei.internalCreatePayment(
    activity: Activity,
    input: NVPayment,
    additionalParams: Map<String, Any>?,
    forceWebChallenge: Boolean,
    source: String,
    callback: Callback<NVCreatePaymentOutput> object : Callback<NVCreatePaymentOutput>{
    override fun onComplete(response: NVCreatePaymentOutput) {
        //do something
    }
}
)
```

InternalCreatePayment will make a payment. For input you must create instance of NVPayment with the payment data.

```kotlin
SimplyConnect.installments.options = arrayListOf(
    Installments.Option(Installments.Type.SINGLE_PAYMENT),
    Installments.Option(Installments.Type.DEFERRED_WITH_INTEREST, intArrayOf(2,4,6)),
    Installments.Option(Installments.Type.DEFERRED_WITHOUT_INTEREST, intArrayOf(3, 6, 9, 12)),
    Installments.Option(Installments.Type.DEFERRED_WITHOUT_INTEREST_AND_GRACE_PERIOD, intArrayOf(4, 8, 12, 16, 20)),
)

SimplyConnect.installments.requirePersonalID = true // if needed
```
This creates instance of Installment with installment options. You can choose which options you will add. If the option is not `SINGLE_PAYMENT` you have to pass list of periods from which the user can choose. You can choose if the national id is required. It depends on the country code. Country codes that we support national id are AR, BR, CL, PE, CO, MX, PY, UY, IL. It is optional.

```kotlin
  fun checkout(
    activity: Activity,
    input: NVPayment,
    additionalParams: Map<String, Any?>? = null,
    forceWebChallenge: Boolean,
    callback: Callback<NVOutput>,
    declineFallbackDecision: ((NVOutput, FragmentActivity, ((Nuvei.CheckoutCompletionAction)->Unit))->Unit)?
)

```
Imports needed for Checkout:
```kotlin
import com.nuvei.mobilesdk.simplyconnect.SimplyConnect
import com.nuvei.mobilesdk.core.model.NVPayment
import com.nuvei.mobilesdk.core.model.NVOutput
```

Usage
```kotlin
SimplyConnect.checkout(
    activity: Activity,
    input: NVPayment,
    additionalParams: Map<String, Any?>?,
    forceWebChallenge: Boolean,
    callback: Callback<NVOutput> = object : Callback<NVOutput>{
    override fun onComplete(response: NVOutput) {
        response
    }
}
){
        response, activity, declineFallback ->
}
```
Checkout will load a screen with saved cards and other payment options. For input you must create instance of NVPayment with the payment data. The callback tells you when the proccess is finished either successfully or with an error. declineFallbackDecision tells you when there is an error.

#### 🎨 SimplyConnect UI Customization

Use SimplyConnectUICustomization to style the SimplyConnect checkout screen (toolbar, labels, text fields, error boxes, buttons) so it matches your app.
If you don’t override a property, the SDK’s default styling is used. This lets you start from defaults and only tweak what you care about (e.g. just toolbar + button colors).
Call this before SimplyConnect.checkout(...):

```kotlin
with(SimplyConnectUICustomization) {
    // Customization of the SimplyConnect screen
    with(recyclerViewCustomization) {
        backgroundColor     = Color.parseColor("#F3F5F7")     // Background color for the whole checkout list (RecyclerView)
        sectionTextFontSize = 16                              // Section header text size (e.g. "My payment methods")
        sectionTextColor    = Color.BLACK                     // Section header text color
        sectionTextFontName = R.font.momoregular              // Custom font for section headers
        cellBackgroundColor = Color.WHITE                     // Background color for each cell/row in the list
        cellTextColor       = Color.parseColor("#4A5568")     // Text color for the main text inside each cell (card, APM, etc.)
        cellTextFontSize    = 14                              // Text size for the main text inside each cell
        cellTextFontName    = R.font.momoregular              // Custom font for the main text inside each cell
    }

    // Styles the top app bar of the SimplyConnect screen:
    with(toolbarCustomization) {
        //backgroundColor           = ContextCompat.getColor(this@Activity, R.color.white)  //Example using colors from your app resources
        backgroundColor             = Color.parseColor("#081F2B")                           // toolbar background
        textFontSize                = 16                                                    // toolbar title size.
        textColor                   = Color.parseColor("#C7CCCF")                           // toolbar title color
        headerText                  = "Checkout"                                            // title text (e.g. "Checkout")
        closeButtonBackgroundColor  = Color.WHITE                                           // background for the close/exit button
        textFontName                = R.font.momoregular                                    // custom font for the title
        logoResId                   = R.drawable.app_icon_foreground                        // logo shown in the toolbar
    }

    // Controls all text fields in the checkout form:
    with(textBoxCustomization) {
        headingTextFontSize         = 16                           // size for section titles above a text input field.
        headingTextFontColor        = Color.BLACK                  // color for section titles above a text input field.
        headingTextFontName         = R.font.momoregular           // font for headings
        textFontName                = R.font.momoregular           // font for input text
        placeholderTextFontSize     = 14                           // size for hint/placeholder style inside fields.
        placeholderFontColor        = Color.GRAY                   // color for hint/placeholder style inside fields.
        textFontSize                = 14                           // actual input text size
        textColor                   = Color.BLACK                  // actual input text color
        borderColor                 = Color.parseColor("#95AAC1")  // outline color of the text fields
        cornerRadius                = 5                            // radius for rounded text field corners
        borderWidth                 = 1                            // border thickness
        backgroundColor             = Color.WHITE                  // background of the input area.
    }

    // Styles error messages shown under fields:
    with(errorBoxCustomization) {
        textFontSize                = 12                           // error text size
        textColor                   = Color.RED                    // error text color
        borderColor                 = Color.RED                    // border color of the error box
        textFontName                = R.font.momoregular           // custom font for error text
    }

    // Styles for the main payment button:
    with(payButtonCustomization) {
        textFontSize                = 14                           // button label size
        textColor                   = Color.WHITE                  // button text color
        backgroundColor             = Color.parseColor("#493DAA")  // button background
        cornerRadius                = 10                           // rounded corners for the button
        textFontName                = R.font.momoregular           // custom font
    }

    // Styles checkbox-like actions (e.g. “Save card”, “Remember my details”):
    with(checkboxButtonCustomization) {
        textFontSize                = 14                           // checkbox label size
        textColor                   = Color.parseColor("#D3DCE6")  // checkbox text color
        backgroundColor             = Color.parseColor("#493DAA")  // checkbox background
        textFontName                = R.font.momoregular           // custom font
    }
}


```

#### 🌍 CheckoutI18N

Before showing the checkout, customize UI texts:

```kotlin
with(CheckoutI18N) {
    cardHolderNameTitle                 = "Cardholder Name"
    cardHolderNamePlaceholder           = "John Smith"
    cardNumberTitle                     = "Card Number"
    cardNumberPlaceholder               = "1234-5678-1234-5678"
    expirationDateTitle                 = "Expiry Date"
    expirationDatePlaceholder           = "12/23"
    cvvTitle                            = "CVV"
    cvvPlaceholder                      = "123"
    cvvPlaceholderAMEX                  = "1234"
    installmentsProgramTitle            = "Installments Program"
    singlePaymentText                   = "Single Payment"
    deferredWithInterestText            = "Deferred with interest"
    deferredWithoutInterestText         = "Deferred without interest"
    deferredWithoutInterestAndGraceText = "Deferred without interest and grace period"
    numberOfInstallmentsTitle           = "Number of Installments"
    personalIdTitle                     = "Personal ID"
    personalIdPlaceholder               = "Personal ID"
    saveDetailsText                     = "Save my details for future use"
    payButtonTitle                      = "Pay %1.2f%2s" // add %1.2f%2s if you want to show amount and currency in button

    numberEmpty                         = "Please fill the credit card number"
    creditCardInvalid                   = "Please enter a valid card number"
    expiryEmpty                         = "Please fill the expiry date"
    expiryInvalid                       = "Invalid expiration date"
    cvvEmpty                            = "Please fill the CVV"
    cvvInvalid                          = "CVV is invalid"
    holderNameEmpty                     = "Please fill the card holder name"
    holderNameInvalid                   = "Invalid cardholder name"
    personalIDEmpty                     = "Please fill the ID number"
    personalIDInvalid                   = "Invalid ID number"
}
```
This creates instance of `CheckoutI18n` with your own localized texts. You can initialize it with the properties and text you want to modify.
If you skip this step, default English strings apply.

#### 🌍 CheckoutApmI18N – APM (Alternative Payment Methods) Localization

`CheckoutApmI18N` lets you localize APM method names, field labels, validation errors, and common UI strings (e.g., “Save details”, “Pay”).
Use it to override server captions or provide translations per market.

---

####  Example Initialization

```kotlin
val apmI18n = CheckoutApmI18N.from(
    listOf(
        MethodMsg(
            methodId = "apmgw_mBank",
            title = "MyBank",
            fields = listOf(
                FieldMsg("bank_account_number", "Bank Account Number", "Invalid account number"),
                FieldMsg("bank_code", "Bank Code", "Invalid bank code"),
                FieldMsg("bank_account_prefix", "Prefix", "Invalid prefix"),
            )
        ),
        MethodMsg("apmgw_Neosurf", title = "Neosurf"),
        MethodMsg("apmgw_expresscheckout", title = "PayPal", fields = emptyList()),
        MethodMsg(
            methodId = "apmgw_Open_Banking",
            title = "Online Transfer",
            fields = listOf(
                FieldMsg("ob_sort_code", "Sort Code", "Please enter a valid sort code"),
                FieldMsg("ob_iban", "IBAN", "Invalid IBAN"),
                FieldMsg("ob_account_number", "Account Number", "Invalid account number"),
                FieldMsg(
                    key = "ob_bank_id",
                    text = "Bank",
                    error = "Please choose a bank",
                    options = listOf(
                        OptionMsg("ob-barclays", "Barclays"),
                        OptionMsg("ob-hsbc", "HSBC"),
                        OptionMsg("ob-revolut", "Revolut")
                        // others will fall back to the server captions
                    )
                )
            )
        )
    ),
    apmUiStrings = CheckoutApmStrings(
        saveDetailsText = "Remember my details",
        payButtonTitleText = "Pay Now",
        emptyFieldErrorText = "This field cannot be empty"
    )
)

CheckoutApmI18NStore.init(apmI18n)

```
* MethodMsg – Defines a single APM method (ID + localized title + optional fields).
* FieldMsg – Localized label & validation error for a specific input field; may include options for dropdowns.
* OptionMsg – Key + display text for selectable values.
* CheckoutApmStrings – Common UI strings rendered across all APM methods.
* CheckoutApmI18N.from(methods, apmUiStrings) – Factory that builds the full i18n map.

## Fields

1. Add the NuveiCreditCardField view to your XML
```xml
<com.nuvei.mobilesdk.fields.NuveiCreditCardField
    android:id="@+id/creditCardField"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```
2. Apply customization
#### 🎨 Fields UI Customization (NuveiCreditCardField)

Use FieldsUICustomization to style the NuveiCreditCardField (background, borders, labels, inputs, errors) so it matches your app.
If you don’t override a property, the SDK’s default styling is used.

```kotlin
// Configure the FieldsUICustomization appearance:
with(FieldsUICustomization) {
    backgroundColor = Color.WHITE              // Background for the entire NuveiCreditCardField container
    borderColor     = Color.BLACK              // Border color around the entire NuveiCreditCardField container
    cornerRadius    = 1                        // Corner radius for the outer container border
    borderWidth     = 1                        // Width of the outer container border

    // Text input boxes: labels/headings, placeholders, values, and their borders
    with(textBoxCustomization) {
        backgroundColor         = ContextCompat.getColor(this@Activity, R.color.white)  // Background color of each text field
        headingTextFontSize     = 16                                                    // Size of labels/headings above the input fields
        headingTextFontColor    = Color.parseColor("#4A5568")                           // Color of labels/headings above the input fields
        headingTextFontName     = R.font.momoregular                                    // Custom font for labels/headings
        placeholderFontColor    = Color.GRAY                                            // Color of hint/placeholder text inside fields
        placeholderTextFontSize = 12                                                    // Size of hint/placeholder text inside fields
        textFontSize            = 14                                                    // Size of the actual input text
        textColor               = Color.BLACK                                           // Color of the actual input text
        borderWidth             = 1                                                     // Border width of each text field
        cornerRadius            = 5                                                     // Corner radius of each text field
        borderColor             = Color.parseColor("#95AAC1")                           // Border color of each text field
        textFontName            = R.font.momoregular                                    // Custom font for the input text
    }

    // Error messages under the inputs
    with(errorBoxCustomization) {
        textFontSize = 12                                        // Error text size
        textColor    = Color.RED                                 // Error text color
        borderColor  = Color.RED                                 // Border color for the error box
        textFontName = R.font.momoregular                        // Custom font for error text
    }
}

```
Customize UI texts:

```kotlin
with(FieldsI18N) {
    cardHolderNameTitle         = "Cardholder Name"
    cardHolderNamePlaceholder   = "John Smith"
    cardNumberTitle             = "Card Number"
    cardNumberPlaceholder       = "1234-5678-1234-5678"
    expirationDateTitle         = "Expiry Date"
    expirationDatePlaceholder   = "12/23"
    cvvTitle                    = "CVV"
    cvvPlaceholder              = "123"
    cvvPlaceholderAmex          = "1234"

    numberEmpty                 = "Please fill the credit card number"
    creditCardInvalid           = "Please enter a valid card number"
    expiryEmpty                 = "Please fill the expiry date"
    expiryInvalid               = "Invalid expiration date"
    cvvEmpty                    = "Please fill the CVV"
    cvvInvalid                  = "CVV is invalid"
    holderNameEmpty             = "Please fill the card holder name"
    holderNameInvalid           = "Invalid cardholder name"
}
```
Callbacks:
```kotlin
creditCardField.transactionDetails = transactionDetails
creditCardField.onInputUpdated = { hasFocus, еxpMonth, еxpYear  -> }

creditCardField.onInputValidated = { errors -> }

creditCardField.onCardDetailsUpdate = { output, error ->
    creditCardField.showCardNumberError("ERROR MESSAGE")
}
```

This creates instance of `NuveiCreditCardField`.
* `OnInputUpdated` is a callback that tells you when a field text is modified and when it loses focus. Also if a valid card expiration date is provided in the expiry date field, it will be split and returned in the еxpMonth and еxpYear parameters, otherwise they will be null
* `OnInputValidated` is a callback that gives you a list of errors if there are any after validation of a field.
* `OnCardDetailsUpdated` is a callback that gives you detailed information about the card after the card number is valid. Then based on your requirements you can call `creditCardField.showCardNumberError(message:)` and your error message will be shown under the card number field.

#### 💳 NVCardDetailsOutput Example

`NVCardDetailsOutput` is a data model representing the details of a payment card returned from an API call.  
It contains brand information, issuer details, supported features, and other metadata you may need for your payment flow.

```kotlin
val cardDetails = NVCardDetailsOutput(
    brand = "VISA",
    secondaryBrand = "VISA Debit",
    cardType = "DEBIT",
    program = "Classic",
    visaDirectSupport = "SUPPORTED",
    mastercardSendSupport = null,
    isPrepaid = false,
    issuerCountry = "US",
    currency = "USD",
    dccAllowed = true,
    sessionToken = "abc123-session-token",
    internalRequestId = 987654,
    status = "APPROVED",
    reason = null,
    errCode = 0,
    merchantId = "merchant_123",
    merchantSiteId = "site_456",
    version = "1.0.0",
    clientRequestId = "req_001",
    issuerBankName = "Chase Bank",
    clientUniqueId = "unique_789",
    bin = "411111",
    last4Digits = "1111",
    ccExpMonth = "12",
    ccExpYear = "2030",
    isNotReloadableCard = false,
    cardProduct = "Standard Debit"
)
```

Methods:

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

val nvPayment = NVPayment(
    sessionToken = transactionDetails.sessionToken,
    merchantId = transactionDetails.merchantId,
    merchantSiteId = transactionDetails.merchantSiteId,
    currency = transactionDetails.currency,
    amount = transactionDetails.amount,
    paymentOption = paymentOption,
    userTokenId = transactionDetails.userTokenId,
    clientUniqueId = transactionDetails.clientUniqueId,
    clientRequestId = transactionDetails.clientRequestId)
```

Nuvei.internalCreatePayment will create a payment with a token. For this method you don't need the `Fields` view . You have to pass an object of type `NVPayment` in which you have to put the token and optionally the card holder name(`PaymentOption` data model).


---

## 📝 Notes

- Use `payButtonTitle = "Pay %1.2f %2s"` to display amount and currency (e.g. "Pay 99.99 EUR").
- Ensure `sessionToken` from `openOrder.do` is valid before calling any payment flow.
- All network calls should be off the UI thread (e.g. coroutines on `Dispatchers.IO`).
