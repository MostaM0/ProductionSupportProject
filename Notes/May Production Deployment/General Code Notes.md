# Sprint 1


## Multi-Account Support (SA, TD, Payroll)

- **Mapping between the Rize customer ID and Bank CIC is located in "customers" table inside the customer-service database.**:
```Kotlin
	override fun getCustomerMigrationInquiry(customerId: String): ResponseEntity<CustomerMigrationInquiryResponse> {
        return if (customerId.isUUID()) {
            ResponseEntity.ok(CustomerMigrationInquiryResponse(MigrationStatus.MIGRATED))
        } else {
            ResponseEntity.ok(customerQueryService.getCustomerMigrationInfo(customerId.toLongOrThrow()))
        }
    }
```
- The previous code implies that all the customers with prospect customer ID will be considered migrated, all the ones that started onboarding before this will not continue.
- API List excel file is located on the 09 - CASA-Inquiry folder in rize-transaformation-design-diagrams repo.
- **The transaction list and the statements get fetched directly from RCM through the customer-transaction-service in case of transaction list and through statement-service in case of statements**.
- CASA-Inq_Get_transaction_List sequence digaram has a typo, it's transaction-service not customer-transaction-service.
- **The SOAP request and response to and from RBS are all printed in the RCM components, meanign any place the RCM components are used, you will see the request and the reposne, you can find these logs in SOAPLogHandler.kt .**
- In order to get the balance of a certain customer we use the /v1/customer-accounts/{customerId}/accounts endpoint in the man-deposit-balance-service, the service eventually talks to the deposit-account-service-v2 account inquiry, any service that gets the balance does so.
- The getAccounts function  in the deposit-account-service-v2:
```Kotlin
    fun getAccounts(
        xTraceId: String,
        customerId: String,
        includeSavingPots: Boolean,
        accountNumber: String?,
        accountType: List<String>?,
        accountStatus: List<String>?
    ): AccountInquiryResponse 
```
- It was required that only this function is used to inquire about accounts, here are some important notes:
  1- When the customerId = * this means that who asked to inquire is using the account number not the customer Id, but the customerId was required to be on the path so it can't be null.
  2- accountNumber is a query parameter so it can be null.
  3- There can also be includeSavingPots field.
- Here are all possible paths based on these three parameters (all code snippets are found in customer-service):
    1- Get All customer accounts given customerId, don't include savings pots:
```Kotlin
    customerId != "*" && accountNumber == null && !includeSavingPots ->
                rbsAcctInqService.getAllCustomerAccounts(
                    customerId,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                ) 
```
    2- Get specific customer account given accountNumber, don't include savings pots:
```Kotlin
    customerId == "*" && accountNumber != null && !includeSavingPots ->
                rbsAcctInqService.getCustomerAccount(
                    accountNumber,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                )
```
    3- Get specific customer account given accountNumber even though customer Id was passed, don't include savings pots:
```Kotlin
    customerId != "*" && accountNumber != null && !includeSavingPots ->
                rbsAcctInqService.getCustomerAccount(
                    accountNumber,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                )
```
    4- Get all customer accounts given customerId, include savings pots:
```Kotlin
    customerId != "*" && accountNumber == null && includeSavingPots ->
                rbsAcctInqService.getAllCustomerAccountsIncludingSavingPots(
                    customerId,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                )   
```
    5- Get specific customer account given accountNumber, include savings pots:
```Kotlin
    customerId == "*" && accountNumber != null && includeSavingPots ->
                rbsAcctInqService.getCustomerAccountIncludingSavingPots(
                    accountNumber,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                )
```
    6- Get specific customer account given accountNumber even though customer Id was passed, include savings pots:
```Kotlin
    customerId != "*" && accountNumber != null && includeSavingPots ->
                rbsAcctInqService.getCustomerAccountIncludingSavingPots(
                    accountNumber,
                    xTraceId,
                    dataMappings,
                    accountType,
                    accountStatus
                )
```
- **In the customer service, there is a very important flag called: digibank.migration-check-flag inside MigrationUtils.kt, this flag should be flipped to false when we are sure that every single customer was migrated to avoid all the checks being made.**
- Inside MigrationHandler.kt interface, you can see multiple functions with different implementations based on if this customer is migrated or not. You can find the implementations on MigratedCustomerHandler.kt or NotMigratedCustomerHandler.kt.
- The account number is not sent and received from RBS as a whole, **it is divided into: Account ID, Account Type and Branch Code and Currency Code.**.
- The logic of mapping the account number to these four values can be found in RCM components in AccountUtil.kt file, it's important to know how to construct and deconstruct these values to be used to get logs.
- Inside the "May Production Deployment" you can find "Assemble Account Number Code" this is a kotlin function I wrote you can use to assemble account number based on the four parts.
- You can notice any RBS request log always starts with "AXIS Request" and similarly RBS response start with "AXIS Response". You will also find "Response Retrieved from Method: xxxx with rqUID: xxxxx" log, very useful.
- In account inquiry, you can see the balance in both "ledgarAcctBal" and "availableAcctBal" fields.


## Caching System
- Data being cached: Customer Accounts and Customer Details because fetching data from RCM was so slow.
- Inside customer-service in the CustomerIntegrationService.kt, you can find the caching functions that fetch data from RBS or CRS in case we can't find the data inside the database table, then insert it in the database.
- **Every two minutes, any cache record is invalidated so that when we try to fetch it, it won't be found, and we will have to call RBS or CRS again to get the corresponding data (code from customer-service):**
```Kotlin
@Value("\${cache.ttl}")
    private val ttl: Long = 120
```
The cache.ttl flag is found at the application.yaml files
- **Every hour, the data itself will be deleted from the cache in order to prevent data overflowing, this is done through the CacheCleanupTask.kt, the cache.cleanup-interval flag can be found on application.yaml too.**
-----------------------------------------------------------------------------------------------------------------------------------------------------
# Sprint 2
## JomPAY
- We call RBS to validate the biller info with the code, rrn (retrieval reference number) and rrn2 (secondary retrieval reference number).
- The favourite data is located inside favourite_bill table, inside the staging_table table you can find records for **only batch bill payments, not individual payments**.
- **How batch payments work ?** (very important: always check the action column inside the JomPAY_API_SHEET_V4, updated api sheet can always be found on TFS):
  1. We store every payment in the batch inside the staging_table.
  2. We send one-by-one to a kafka topic for processing, and change it's status in the database to FINISHED.
  3. In the end of every bill processing, check that all the bills with the same batch are not of PENDING status, if so then send to the async service the result, even if one of them is REJECTED.
- After each payment, we construct a JWT favourite token to be returned to the frontend to be used if the customer decided to add the biller as a favourite.
```Kotlin
    addToFav=jwtUtilService.createToken("Jompay Add To Fav",
    FavoriteBill.builder()
    .billerCode(bill.getBillerCode())
    .billerName(bill.getBillerName())
    .rrn(bill.getRrn())
    .rrn2(bill.getRrn2())
    .build(),
    JWTUtilService.Secrets.JOMPAY_BILL_PAYMENT_SECRET_KEY.getValue(),
    jwtProperties.getValidateTokenInMsec());
```
-----------------------------------------------------------------------------------------------------------------------------------------------------
## Limits and Fees Management
- The limits algorithm is located in both the transaction-service and the transaction-limit-service.
- The image called "New Payments with Fees" describes how payments with fees work with the introduction with RBS.
- In the image there is two paths starting from number 4, the normal RBS path for RBS accounts (IBK) and internal Rize accounts (DGB).
- In pay to proxy, the payment service sends fees to the FIS gateway in the payment details field separated by "||||".
- In pay to QR, it is being sent in the recipient reference field also separated by "||||".
- The normal cooling-off record for changing the limit can be found on the customer-restriction-service blocks table, but the new kind of cooling-off, the one related to using the effective limit, is in a table in customer-limit-service.
- We saw the customer segment is fetched from the customers table in the customer-service database, with values AFFLUENT_RESIDENT, MASS_RESIDENT and MASS_NON_RESIDENT.
- The "validateTransaction" function in the transaction-service is a huge function that goes throw all the limit validation process done on a transaction, you will find all the validation logic there with different exceptions thrown if any validation failed.
- Our microservices don't communicate directly with the backoffice when it comes to fetching fees or anything else, instead, any change from the backoffice reflects on the Rize Database, this is why some database scripts skipped the deployment of the backoffice all together.
- Inside the customer_consumption table inside the transaction-service database you can find the cumulative consumption of a customer in a certain transaction type.
- There are no logs inside the function but if any kind of limit validation was violated it will throw it's corresponding exception.
- There are fees calculated for every transaction based on customerSegment, transactionType, transactionAmount and the currency. There is also vat value but currently this vat is set to zero by default.
- The sequence diagram "Account-Payment-Approval-MFA + Cooling_Off v1.1" is wrong in the part where we decide the "byPassSecureTransactionLimit" based on the response of validatePayment, instead it has it's dedicated API that is called in the payment initiation of account or proxy payment.
- **How the communication between Backoffice, transaction-service and transaction limit service works out ?** :

  1- When BO changes any kind of configuration, its is published for both transaction-service and transaction limit service, the kafka listeners are found in consumer > backoffice.

  2- 


-----------------------------------------------------------------------------------------------------------------------------------------------------
## Onboarding (Branch ETB with NRIC)

- There was a critical issue during regression where during syncing the prospect customer data with the customer service data in the very end, the Rize email was not saved in the database before sending the Kafka message that syncs the data. This means the email will always be null on the customer database and any other flow that depends on it will always fail.


-----------------------------------------------------------------------------------------------------------------------------------------------------
