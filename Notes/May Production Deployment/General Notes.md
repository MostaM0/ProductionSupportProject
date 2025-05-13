
# Sprint 1


## Multi-Account Support (SA, TD, Payroll)

- Rize customer ID: the original used in: **internal microservice communication, Jumio, Feedzai, ComplyAdvantage, Twillio, Micorkiosk...etc.**

- Bank CIC: a new id used mainly with anything **related to money and cards in general: RBS, cards, communication with Finexus.**

- CASA stands for "Current Account and Savings Account".

- RCM = RBS + CRS.

- CRS contains: **addresses, profile data (phone number, martial status, nickname .. etc)**.

- Customer service: **preferences, email in rize (there is an email in RBs but they raised an issue for the email to be from Rize isntead) , Pingone data, scan reference for Jumio**.

- Spending balance is the balance without the money in the savings pot while Current balance is the total amount of money you have.

- The transaction list exclude savings pot transactions, the savings pot transactions are in the Savings Pots Inquiry page.

- Clicking the transaction list item itslef will show transaction recipet that an be printed.

- The transaction data (transaction history) gets fetched from **RBS directly, there might be differnece between Rize database and RBS**.

- The statements are also **generated from RBS directly not in Rize**.

- There is only savings pot in case of the **savings account**.

- You can set an account to be a default account from the settings, **this affects the first account to appear and the DuitNow proxy to bind**.

- According to Fathy, the accounts are fetched a single time on the dashboard and the accounts tab (to get the updated data, specifically the balance), and everytime there is a choice for the customer to choose a certain account in performing a certain action (choosing an account to deduct money from in the normal DuitNow transfers, for example), they pass it to this screen.


-----------------------------------------------------------------------------------------------------------------------------------------------------

## Financial Enquiry (PF, HF, AF)

- We don't apply for any financing application through Rize, **we only observe the details of an already existing application**.


-----------------------------------------------------------------------------------------------------------------------------------------------------

## Term Deposit-i

- You choose a term (e.g., 1 month, 6 months, 1 year, 5 years, etc. You cannot withdraw the money before the term ends without a penalty. The bank pays you interest on your deposit — usually higher than a regular savings account. At the end of the term (called maturity), you get back your original deposit plus the interest earned.

- You can withdraw the term deposit money from the applciation directly.


-----------------------------------------------------------------------------------------------------------------------------------------------------


# Sprint 2


## JomPAY

- The favourites list gets displayed by default, if no favourites are there the default three-field page gets displayed.

- You don't specify the biller amount when you add a favourite, **instead the biller amount is taken from the user in the transaction details screen for each biller**.

- The "Schedule These Payments" switch is not there at the moment with the whole functionality, will be up with Sprint 3.

- When you choose to pay to multiple billers, **you show the result for each payment in the results screen and the receipt shows the same screen**.

- A notification and an email is sent when a successful JomPAY transfer succeeds.

- The JomPAY transactions appear separately on the trasnaction history page.

- You can edit a favourite and you can delete a favourite.

- We currently call the inquire API two times for Jompay, one with the account screen and on the amount screen, there is a debate on the bank currently for a change request to make a single inquiry after the amount screen.



-----------------------------------------------------------------------------------------------------------------------------------------------------


## Limits and Fees Management

- These are limits for the following:

	1- Transaction Type Limit

	2- Transaction Group Limit 

	3- User Segment Limit

- Transaction Type Limit = Account, Proxy or QR.

- Transaction Type Limit = Any number of transaction types bundled together, it can be configured from the Backoffice as well.

- User Segment Limit = for each customer based on their income and job, they are inserted into a certain segment.

- At the time of writing, there is two types of payments on the mobile UI:

	1- DuitNow to Account/Proxy, DuitNow QR

	2- JomPAY/JomPAY

	For IBG and Within the bank transfers, these ones will be up and live by sprint 3.

- The "DuitNow to Account/Proxy, DuitNow QR" **is a transaction limit group**, while the JomPAY/JomPAY **is a transaction type** that gets fetched by it's database id from the frontend.

- Fees are now applied over every transaction type, we will investigate from where we fetch these fees and VAT in the code notes.

- **Very Important: At the time of writing we learned that the fees and VAT info are not actually being displayed to the users and are reducted automatically in the transactions, and the info itself does not get displayed to the user.**

- Transaction groups can be created dynamically at the back office, but they won't reflect on the frontend, dynamic from the backend static from the frontend.

- **Increasing the limits now applies a cooling-off on the increased limit amount**, for example: If my current limit is RM 1000, and I increased it to RM 2000, I will be able to only use the original RM 1000 for 12 hours, I will then be able to use the RM 2000 after the 12-hour cooling-off.

- Removing the cooling-off now will result on the ability of the customer to use the full amount (RM 2000 in the previous example) of their limit.

- An important issue: There is a value called default limit for each payment group, including the "DuitNow to Account/Proxy, DuitNow QR" group, it takes this value from one of the payment type limits in the group, this happens only for the first time. The problem arises when you change the limit of one of the transaction types that won't reflect on the default limit. If the user changed the limit one (transaction group limit), then there will be no issue.

- There is no cooling-off screen now, it will always display as you exceeded the limit screen error.

-----------------------------------------------------------------------------------------------------------------------------------------------------

## FCY Account Enquiry & Monthly Statements

- The foreign currencies accounts are located in a separated tab inside the Account bottom tab.

- Like the normal account transactions, clicking on a transaction record in the transaction history list will show you the details page allowing you to print it.

- Currency exchange is out of scope, currently we can inquire about a foreign currency account, see its details, see transaction history with filtering and statements.

- The statements can be accessed from the account details page, customers can then switch the account to view various statements for each account including the FC account.

- Statements are fully generated from RBS side.


-----------------------------------------------------------------------------------------------------------------------------------------------------

## Savings Pot

- The savings pot behaviour will be as per production, only changes are: Editing Savings pot name and image.

- Savings pots are now managed by RBS, it just a normal savings account tied to the main Rize savings account with a special label attached.

- At the time of writing, as the savings pot is a saving account, it comes with -10 Ringgits by default, they are still discussing this issue.

-----------------------------------------------------------------------------------------------------------------------------------------------------

## Onboarding (Branch ETB with NRIC)

- All customers that are currently prospect will not be able to continue their onboarding journey after the production deployment because the steps now are differnet.

- ETB = Existing To Bank customer means they already have an account on Alrajhi bank, NTB = Non-existing To Bank customer.

- For the NTB flow, it's the same from the frontend side, for ETB it exists only on the new app.

- ّ**Inside the onboarding microservice there is an excel file showing all the different scenarios for the onboarding, this is very important**.

- The back office does not decide if a customer is Rize or RBS or anything. They only check if the data extracted from the MyKad scan is equivalent to MyKad data in the MyKad image.

- Notes for every onboarding scenario (names according to excel sheet, for every scenario, try to amp it to figma):

	1- **ETB_Mod_ETB**: 

	- ETB_ONBOARDING step is where the syncing happens between the onboarding and customer service.
	- FINISHED_JOURNEY step in this scenario and any other scenario is to singal for the frontend that the onboarding journey is over.
	
	2- **ETB-Mod-MobileMismatch**:

	- ETB_MOBILE_MISMATCH step gets INITIATED when the result come from back office.
	- The customer calls the customer support, they change the number in CRS directly, when they login again the frontend sends to backend to perform the same comparison and will now find both numbers match. It will then make the ETB_MOBILE_MISMATCH_OTP_VERIFICATION and ETB_MOBILE_MISMATCH_LIVENESS_VERIFICATION step initiated.
	
	3- **ETB-Mod-NTB**:

	- The customer is simply lying and we will continue making an account for them as if they don't exist.
	
	4- **ETB-Mod-RIZE**:

	- They exist but not RBS customer, they are Rize customer; therfore there current onboarding must be archived.
	
	5- **NTB-Mod-NTB**:

	- This is just the normal onboarding, the only difference is that the back office is now involved.
	- FINISHED_JOURNEY is there also.
	
	6- **NTB-Mod-ETB**:

	- They chose new but they already had an RBS account, allow them to continue onboarding but with less steps.
	
	7- **NTB-Mod-MobileMismatch**:

	- Basically the same as ETB-Mod-MobileMismatch, the only difference is that the customer chose wrongly at the start.
	
	8- **NTB-Mod-RIZE**: 

	- Basically the same as ETB-Mod-RIZE, the only difference is that the customer chose wrongly at the start.
	
	9- **NTB-NotMod**:

	- This is just the pure normal Rize onboarding currently on production.
	
	10- **RIZE_CUSTOMER**: 

	- Basically the same as ETB-Mod-RIZE, but the only difference is that the customer modified their Jumio data.

	11- **ETB_CUSTOMER**:

	- The ETB elligibility found the prospect existing in the bank.
	
- The steps are generated dynamically, for example when we tried to bypass jumio on the onboarding we needed to delete steps for them to be generated gain.

- There was an issue in case of ETB mobile mismatsh, when they fail Jumio liveness check it never changes to FAIL, keep an eye for that (this was solved, now it displays as FAILED).


	
-----------------------------------------------------------------------------------------------------------------------------------------------------

## Online Banking e-Wallet (OBW-CA)

- We display a list of banks the user can use to top up their account, and depending on the chosen bank, it opens in-app browser for this bank where you enter the data required to proceed with debiting the customer account in that bank.

- For non-personal (top-up account name is different, different KAD .. etc), e-wallet and joint account top-up, the transaction is reviewed and has to be approved in order for the custoemr to proceed.

- Top-up unsuccessful if: User clicks on the back button, debiting agent has system error or insufficient account balance in the customer's bank.

- When user finishes topping-up using their other account portal, a special screen of "We're processing yor payment" gets displayed, it's time out only redirects customers to the home page, but it does not fail the topping-up.

- Most probably won't go to production.


-----------------------------------------------------------------------------------------------------------------------------------------------------
































































