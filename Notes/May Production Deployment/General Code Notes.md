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
  
  1- When the customerId = * this means that who asked to inquire is using the account number not the customer Id, but the customerId was requried to be on the path so it can't be null.
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


## Caching System

- Data being cached: Customer Accounts and Customer Details because fetching data from RCM was so slow.

- Inside customer-service in the CustomerIntegrationService.kt, you can find the caching functions that fetch data from RBS or CRS in case we can't find the data inside the database table, then insert it in the database.

- **Every two minutes, any cache record is invalidated so that when we try to fetch it, it won't be found and we will have to call RBS or CRS again to get the corresponding data (code from customer-service):**

```Kotlin
@Value("\${cache.ttl}")
    private val ttl: Long = 120
```
The cache.ttl flag is found at the application.yaml files

- **Every hour, the data itself will be deleted from the cache in order to prevent data overflowing, this is done through the CacheCleanupTask.kt, the cache.cleanup-interval flag can be found on application.yaml too.**


-----------------------------------------------------------------------------------------------------------------------------------------------------

# Sprint 2

## JomPAY

- **In the customer service, there is a very important flag called: digibank.migration-check-flag inside MigrationUtils.kt, this flag should be flipped to false when we are sure that every single customer was migrated to avoid all the checks being made.**

- Inside MigrationHandler.kt interface, you can see multiple functions with different implementations based on if this customer is migrated or not. You can find the implementations on MigratedCustomerHadnler.kt or NotMigratedCustomerHadnler.kt.

- 


-----------------------------------------------------------------------------------------------------------------------------------------------------

## Limits and Fees Management

- The limits algorithm is located in both the transaction-service and the transaction-limit-service.

- The image called "New Payments with Fees" describes how payments with fees work with the introduction with RBS.

- In the image there is two paths starting from number 4, the normal RBS path for RBS accounts (IBK) and internal rize accounts (DGB).

- In pay to proxy, the payment service sends fees to the FIS gateway in the payment details field separated by "||||".

- In pay to QR, it is being sent in the recpient reference field also separated by "||||".

- there is no cooling-off screen now, it will always display as you exceeded the limit screen error.


-----------------------------------------------------------------------------------------------------------------------------------------------------

## FCY Account Enquiry & Monthly Statements


-----------------------------------------------------------------------------------------------------------------------------------------------------
