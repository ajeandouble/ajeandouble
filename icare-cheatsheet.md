# Icare cheatsheet

## Table of contents

1. [API](#apinotes)
   1. [CRUD Operation example](#api-crud)
   2. [Cron jobs](#api-cron)
2. [Icare Clients](#icareclients)
   1. [Notes](#icareclients-notes)
   2. [Registration process](#icareclients-registration)
   3. [Login process](#icareclients-login)
   4. [Logout process](#icareclients-logout)
   5. [Orders process](#icareclients-orders)
   6. [App version check](#icareclients-appversion)

## API  <a name="apinotes"></a>

### CRUD Operation Example <a name="api-crud"></a>

#### Client-side

dispatch someUserAction(payload)

&emsp;**call**apiCall(payload)

&emsp;&emsp;get JWT from local storage

&emsp;&emsp;add JWT token to HTTP headers

&emsp;&emsp;send request to server

&emsp;&emsp;**if response has an error status** axios throws error

&emsp;&emsp;**else**	

&emsp;&emsp;&emsp;return response

&emsp;&emsp;**catch err**

&emsp;&emsp;&emsp;**if 401 Unauthorized** (session expired/invalid token etc.)

&emsp;&emsp;&emsp;&emsp;toast Session Expired

&emsp;&emsp;&emsp;&emsp;clear localStoragen

&emsp;&emsp;&emsp;&emsp;go to /login

&emsp;&emsp;&emsp;**else**

&emsp;&emsp;&emsp;&emsp;**throw err**	

​	&emsp;&emsp;**// ..** **DO THINGS** .. (and dispatch things)

&emsp;**catch err**

&emsp;&emsp;**call** handleApiError

&emsp;&emsp;&emsp;**if** error.response.data.body.message || error.message

&emsp;&emsp;&emsp;&emsp;toast message

&emsp;&emsp;&emsp;**else**

&emsp;&emsp;&emsp;&emsp;toast "Something went wrong"



### API Side

&nbsp;clientRoutes.js

&emsp;validate token, check JOI Schema ( [JOI](https://joi.dev/api/))

&emsp;&emsp;→clientController.js (parses body and url params, check for error and set HTTP status accordingly)

&emsp;&emsp;&emsp;→clientService.js



### Cron jobs <a name="api-cron"></a>

[Node-cron](https://www.npmjs.com/package/node-cron) module is used in *src/helper/cronHelper.js* to trigger scheduled operations such as:

- orderReminder: send reminder to the client for services yet to be provided
- checkExpiredOrders: check for expired orders in orders collection. Cancel stripe payment intent. Set point to 0 for the order. Send email to client.
- serviceStartSMSBeforeOneHour: twilio SMS reminder one hour before service sent to provider
- checkProcessingOrdersOfDeactiveProvider: fully delete deactivated accounts in providers/clients collections if all services performed after 15 days.
- insuranceValidityCheck: send reminder for renewing assurance documents if they expire in less than 35 days



## Icare-Clients <a name="icareclients"></a>

### Notes <a name="icareclients-notes"></a>

The front uses two reducers combined in *src/store/rootReducer* : userReducer and orderReducer. 

It would be interresting to review the selection process. Complicated logic and conditions there. Would be cool to find a way to describe impacts of fields and service options on others.

The processes usually use *__START* and *_END*  action types to save loading state to the store and render the forms accordingly.

**Geolocation** App.jsx includes GeolocationPermission.jsx which calls the ionic native geolocation which triggers the browser/phone geolocation menu.



#### Local Storage

- JWT Token ( [Storing jwt](https://stormpath.com/blog/where-to-store-your-jwts-cookies-vs-html5-web-storage))

  Registering process

  user and order store



### Registration process <a name="icareclients-registration"></a>

#### /register-phone	appRoutes.REGISTER_PHONE

On **send OTP:**

&emsp;**signupPhone**

&emsp;&emsp;userActionType.CLEAR_STATE			(clear local storage, remove token from state, isLoggedIn = false, logoutLoading = false)

&emsp;&emsp;orderActionType.CLEAR_STATE

&emsp;&emsp;**apiCall** /client/signup/phone

&emsp;&emsp;(API) format number to international format with a preceding '+'

&emsp;&emsp;&emsp;check if non-deleted user exists

&emsp;&emsp;&emsp;generate new OTP

&emsp;&emsp;&emsp;if user exists and is in registration phase: resend OTP via twilio

&emsp;&emsp;&emsp;else send first OTP message



#### /verify-phone	appRoutes.VERIFY_PHONE



On **verify otp**

&emsp;UPDATE_SIGNUP_STATE (userState.signup['phoneOTP'] == otp)

On **resending otp**

&emsp;apiCall /client/signupPhone



**Notes**

phoneNumberVerified useSelector is greyed out in the front.

The field doesn't exist in the signup state but it still gets updated.





#### /register-mail	appRoutes.REGISTER_MAIL

Update userState.signupState['email'] and ['password'] fields

**API**:

&emsp;format mail

&emsp;save mail to db, ignore deleted users

&emsp;generate hash with bcrypt and save it to DB

&emsp;send an OTP via mail



#### /verify-email	appRoutes.VERIFY_EMAIL



#### /register-name	appRoutes.REGISTER_NAME

userActionType.SIGNUP_STATE (update userState.signup fields →image, firstname, lastname etc.)

&emsp;**apiCall** (/signup/profile)

&emsp;&emsp;**signupProfileUpdate**

&emsp;&emsp;&emsp;Create **stripe customer** → stripeHelper.createCustomer

&emsp;&emsp;&emsp;generate user  token and save it to db

&emsp;&emsp;&emsp;save infos to db

&emsp;&emsp;&emsp;send success mail

​	

#### /signup-succes appRoutes.SIGNUP_SUCCESS



### Login process <a name="icareclients-login"></a>

####  /login	appRoutes.LOGIN		LoginPage.jsx

&emsp;On **login**:

&emsp;&emsp;userActionType.CLEAR_STATE			  (clear localStorage and JWT)

&emsp;&emsp;orderActionType.CLEAR_STATE			(clear localStorage and JWT)

&emsp;&emsp;userActionType.LOGIN_BEGINS

&emsp;&emsp;**apiCall** /client/login 

&emsp;&emsp;(API) **if** client.accountStatus == "deleted"

&emsp;&emsp;&emsp;throw Error if password invalid **throw** Error

&emsp;&emsp;&emsp;userActionType.LOGIN_FAILURE

&emsp;&emsp;&emsp;toast

&emsp;&emsp;**else**

&emsp;&emsp;&emsp;return JWT

&emsp;&emsp;...

&emsp;	**on success**

​	&emsp;&emsp;	userActionType.LOGIN_SUCCESS	(save token, loginLoading = false, isLoggedIn = true)

​	&emsp;&emsp;	save JWT to local storage (used in AuthRoute.jsx to display authorized urls or go back to /home if not logged in)

&emsp;**on failure**

​	&emsp;&emsp;userActionType.LOGIN_FAILURE	(token = null, loginLoading = false, isLoggedIn = false)



### Logout process <a name="icareclients-logout"></a>

#### logout()

​	clear local storage

​	**API** /logout

​		delete UserToken model

​	

### Orders process <a name="icareclients-orders"></a>

/property-list →/property-renting-date →

​	switch case →go to detail page (e.g. appRoutes.CLEANING_HR_DETAIL (cleaningByHrDetail.jsx page))

​		→addItemToFinalCart orderActionType.ADD_FINALCART_ITEM

/cart →/payment →orderSuccess→*/alerts*

/payment

​	**API** /order-item

​		**create stripe payment intention** stripeHelper.createPayment

​			set capture_method to manual and set curency

​		**DB**  orderStatus.orderItem.status = 'NEW'

​		goto /orderSuccess



### App version check

​	App.jsx

​		getDeviceInfo()

​			if latest app version is not the app version running on device

​				type: userActionType.SET_SHOW_APP_VERSION_ALERT, setShowAppVersion == true

RouteDetect is included in App.jsx

​	RouteDetect.jsx

​		if not latest app version

​			redirects to appRoutes.APP_VERSION_ALERT.url

​		else

​			switch route

​				case ... (show bottom tab menu accordingly)



### Bottom tab menu

RouteDetect.jsx

​	Display bottom tab menu and dispatch(setShowBottomTab()) depending on the route

​		displays if route in ORDERS, LIST_ORDERS, ALERTS, MENU, RATING_TODO?, HOME



***Note:*** is still a /menu url. Is it legacy or is it used somewhere? All links seem to redirect to /alerts.



### Evolution

Evolution progresses are saved in *client_evolutions* and *provider_evolutions* collections.  The API entry points are implemented in their own *evolutionService.js* file.

The points and rewards are distributed and saved in the DB in *providerService.s* and *clientService.js* API points after some actions are performed (e.g. an order is created).

Some API points (e.g. *providerService.createOrder*) add the current points in *user* and *provider* collections with the new points and save the result.

They also add a bonus on level up. Levels are defined in *commonHelper.getClientLevel*.



#### /evolution

&emsp;(API) call /evolution

&emsp;&emsp;fetch evolution level details sorted by date

&emsp;&emsp;

***Note:*** the total XP is saved in *user* and *provider* collections. *getClientEvolution* and *getProviderEvolution* return sorted-by-date list of the different XPs progression milestones. When an API function adds an evolution document, it adds up the XP to the user/provider total point count??



### Tips



## Providers

### Orders process

/offers

​	onClick

​	show offerDetails.jsx as modal

​			**API call** /provider/accept-offer providerService.acceptOffer

​				check if there is a providerFullfillment document corresponding to the order Id and throws error in that case

​				check if there is such an offer with that order Id if not throws an 'Invalid order'

​				create a payment intent with stripe

​				stripe.paymentIntents.capturePayment → proces the order

​		

Processing the order triggers the update of the client/provider points and updates of levels.

 

## DBs

entity names AKA collections are stored in 

### client

emailVerified, phoneVerified, hasOrdered,

points, level,

emailAlert, smsAlert,

phoneNumber,

otp,

registrationPending,

 - properties
 - billingAddresses

email,

emailToken,

password,

firstName,

lastName,

stripeCustomerId

firebaseNotificationWebToken,

type



- What triggers payment intention to real payment from the clients credit card?

### Questions

- ~~What is the role of the signup state in the signup process? It is only used a couple of time in the userReducer as in CLEAR_APP_STATE and CLEAR_SIGNUP_STATE. Is it legacy?~~
- What is the user token saved in register-name exactly for? Probably not the JWT tochen used in AuthRoute.jsx
- ~~Order payment is where the payment happens? Anyway to cancel a payment intent? Where is~~ 





