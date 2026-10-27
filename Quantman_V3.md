     Quantman App V3
     🔴 MUST DO THINGS BEFORE START 🔴
## Expo SDK Upgrade

- **Upgrade Expo SDK** to [Latest](https://docs.expo.dev/workflow/upgrading-expo-sdk-walkthrough/)
    - [x] Upgrade
    - [x] Audit dependencies that only support lower SDK versions.
    - [x] Install [ExpoUI](https://www.npmjs.com/package/@expo/ui) For more Cross Platform native handles.
    

##  Impl. `sentry` and `ENV`

- [ ] Implementation sentry
- [ ] Must Create Environment and handle the build <span style="color: red">[ Ex: PRODUCTION && DEVELOPMENT]</span>

##  Local Caching System

- **Implement local caching with [expo-sqlite](https://www.npmjs.com/package/expo-sqlite)**
        
    - [ ] Implement and local data storage and retrieval
    - [ ] Remove <span style="color: violet;">@react-native-async-storage/async-storage</span> from the package and move the existing storage to expo-sqLite
##  Theme 

- [ ] **Migrate existing screens to the new color palette**
- [ ] Make a dedicated component for handling `LinearGradient`


---
	Landing page changes

- [ ] Move <span style="color: green;">&lt;UserAuth /&gt; (Broker Login)</span> to [client-taproot](https://github.com/zeequant/client-taproot) from [zeequant-frontend]([https://github.com/zeequant/zeequant-frontend/](https://github.com/zeequant/zeequant-frontend/tree/master/app/javascript/v3/pages/LandingPage/Default/BannerSection/UserAuth))
	- [ ] Handling the Browser opening on the web-application and mobile-application
- [ ] Move [BrokerImages](https://github.com/zeequant/zeequant-frontend/tree/master/public/images/brokers) To [client-taproot](https://github.com/zeequant/client-taproot)
- [ ] Remove mobile auth Route on the zeequant-frontend `[If not needed]`
- [ ] keep the UserData and other important keys on the [local database](https://www.npmjs.com/package/expo-sqlite)

---

    Dashboard Fixes

- [ ] Must reload the positions when the app is in background more than 30 Seconds.

---

	Plan page fixes

- [ ] Integrate [Razor pay](https://razorpay.com/docs/payments/payment-gateway/react-native-integration/standard/) on mobile.