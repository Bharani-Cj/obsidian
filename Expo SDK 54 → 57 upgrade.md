  
Branch: `sdk-57-upgrade` · Package manager: npm · Node: **22.13.1** (current `expo-doctor` needs 22.13+; the project

pins in `.nvmrc`, `package.json` engines and `eas.json` are already moved to it). Switch your shell with `nvm use 22.13.1`.

  

Do it as three hops (55 → 56 → 57). Commit after each hop. Don't skip the build at a hop: a failure is much easier to

diagnose before the next SDK is stacked on top.

  

How to use this file: tick the boxes as you go. When something fails, paste the output into that hop's **Error log**

and send me the file.

  

| SDK | React Native | React | Notes |

| -------- | ------------ | ------ | ------------------------------------------------------------------- |

| 54 (now) | 0.81.5 | 19.1.0 | |

| 55 | 0.83 | 19.2 | Legacy architecture removed (you're already on New Arch) |

| 56 | 0.85 | 19.2 | expo-router drops react-navigation, Hermes v1 default, iOS min 16.4 |

| 57 | 0.86.3 | 19.2.3 | Target. Use `expo@57.0.17` or later (Hermes memory/startup fixes) |

  

---

  

## 0. Before you start

  

- [x] `git switch -c sdk-57-upgrade`

- [x] `npm run ios` builds on 54 (baseline — so any later breakage is the upgrade)

- [x] Save the typecheck baseline:

```bash

npx tsc --noEmit --ignoreDeprecations 5.0 2>&1 | grep 'error TS' | sort > ~/tsc-before.txt

```

- [x] Leave the GitHub token in `package.json` alone for now (it's how `npm install` fetches `zeequant-constants`).

Rotate it as a separate task: it is also in `package-lock.json` and git history (commit `f11e177`).

  

---

  

## 1. Hop 1 — SDK 55

  

- [x] `npx expo install expo@^55.0.0 --fix` — **expect it to exit 1** (see the error log below): it can't edit

`app.config.ts`. The packages still install.

- [x] `app.config.ts`: add `"expo-font"` and `"expo-image"` to `plugins` (SDK 55 gave both a config plugin)

- [x] `npx expo install --fix` again — now exits 0 and installs the dev dependencies the first run skipped

(`@types/react` → ~19.2.10, `eslint-config-expo`, `jest-expo`). No separate `@types/react` step needed.

- [x] `package.json`: change `"expo": "^55.0.0"` to `"~55.0.32"` (the CLI writes a caret; everything else uses `~`)

- [x] Switch to Node 22 first: `nvm use 22.13.1` (check with `node -v`)

- [x] Project pins moved to Node 22.13.1: `.nvmrc`, `package.json` engines (node 22.13.1, npm 10.9.2), `eas.json`

(`uat`, `apk`, `production`)

- [x] After switching Node, reinstall once so `node_modules` is consistent: `rm -rf node_modules && npm install`

- [x] Doctor is clean, or every warning understood. Copy only the command, nothing after it: `npx expo-doctor@latest`

- [x] `app.config.ts`: delete `newArchEnabled: true,` (removed from schema in 55)

- [x] `app.config.ts`: delete `edgeToEdgeEnabled: true,` (removed from schema in 55)

- [x] `rm -rf ios android && npx expo prebuild --clean`

- [x] JitPack plugin still applied. Expect the `content { includeGroupByRegex ... }` block in `build.gradle` and

`react.includeJitpackRepository=false` in `gradle.properties` (the `-i` matters: the property has a capital J,

and `-A4` shows the filter lines, which don't contain the word "jitpack"):

`grep -n -i -A4 jitpack android/build.gradle android/gradle.properties`

- [x] `npm run ios` builds

- [x] `npm run android` builds

- [x] App launches and the smoke test (section 5) passes

- [x] Commit: `SDK 55`

  

If the iOS build fails in Firebase (the `forceStaticLinking` workaround was written for RN 0.81), try first:

`npm i @react-native-firebase/app@latest @react-native-firebase/analytics@latest`

  

### Error log — hop 1

  

**1. `npx expo install expo@^55.0.0 --fix` exited with code 1** — not a real install failure.

  

```

Cannot automatically write to dynamic config at: app.config.ts

Add the following to your Expo config

{ "plugins": ["expo-font", "expo-image"] }

Error: npx expo install --fix exited with non-zero code: 1

```

  

Cause: SDK 55's `expo-font` and `expo-image` ship config plugins. The CLI tries to append them to the app config,

but `app.config.ts` is a function so it can't, and it exits 1. Side effect: it stopped before the dev-dependency pass,

so `@types/react`, `eslint-config-expo` and `jest-expo` were left on their old versions.

Fix: add both plugins to `app.config.ts`, then re-run `npx expo install --fix`. Status: ☐ fixed

  

**2. `npx expo-doctor@latest — clean` failed** — two separate problems.

  

```

npm error arg Argument starts with non-ascii dash, this is probably invalid: —

Node.js (v20.20.0) is outdated and unsupported. Please update to a newer Node.js LTS version (required: >=22.13.0)

Project directory /Users/bharani/Desktop/zeequant/quantman-lite/— does not exist

```

  

Cause A (typo): the `— clean` from this checklist was copied along with the command, so the em dash was passed as the

project directory. Run just `npx expo-doctor@latest`.

Cause B (real): `expo-doctor` 1.21.0 and later require Node 22.13+. 1.20.4 and earlier accept Node 20.19.4+.

Fix: `nvm use 22.13.1` (already installed), or on Node 20 run `npx expo-doctor@1.20.4`. Status: ☑ fixed

  

**3. Doctor ran on Node 22: 18/20 passed, 2 failed — both network, not the project.**

  

```

Unexpected error while running 'Check Expo config (app.json/ app.config.js) schema' check:

TypeError: fetch failed

ConnectTimeoutError: Connect Timeout Error (attempted address: exp.host:443, timeout: 10000ms)

✖ Check Expo config (app.json/ app.config.js) schema

✖ Validate packages against React Native Directory package metadata

Directory check failed with unexpected server response

```

  

Cause: both checks call Expo / React Native Directory servers. The first timed out connecting to `exp.host`; the second

got a bad response. Neither inspects your code. From another shell all of `exp.host`, `api.expo.dev` and

`reactnative.directory` answered 200 in under a second, so this looks transient.

Fix: re-run `npx expo-doctor@latest`. If it repeats: turn off VPN/proxy, or try

`NODE_OPTIONS=--dns-result-order=ipv4first npx expo-doctor@latest`. The schema check is the one that would catch a bad

`app.config.ts` key, so don't tick the doctor box until it passes. Status: ☐ re-run

  

**Next error:**

  

```

Command:

Platform:

Error output:

  

What I tried:

```

  

---

  

## 2. Hop 2 — SDK 56

  

- [x] `npx expo install expo@^56.0.0 --fix`

- [x] `npx expo-doctor@latest`

- [x] Run the router codemod on each source dir:

```bash

for d in app components globals pages src; do

npx expo-codemod sdk-56-expo-router-react-navigation-replace $d

done

```

- [x] `git diff` reviewed. Expo's migration guide sends `ThemeProvider` to `expo-router/react-navigation`, but the

codemod README maps `@react-navigation/native` to `expo-router`. If the codemod wrote the wrong one, fix by hand.

- [x] `grep -rn "@react-navigation" app components globals pages src` returns nothing (see the file tracker below)

- [x] `npm uninstall @react-navigation/native @react-navigation/bottom-tabs @react-navigation/elements`

- [x] Read https://docs.expo.dev/router/migrate/sdk-55-to-56/ for `<Tabs tabBar={...}>` in

`app/(protected)/_layout.tsx` — confirm the custom tab bar still works unchanged

- [x] `rm -rf ios android && npx expo prebuild --clean`

- [x] `npm run ios` builds

- [x] `npm run android` builds

- [x] Smoke test (section 5) passes — especially light/dark theming, tabs, stack navigation, OAuth return

(`app/auth/callback.tsx`)

- [x] Commit: `SDK 56`

  

Notes for this hop:

  

- Hermes v1 is now the default. Its reanimated/worklets memory regression is fixed in 57, so **don't ship from here**.

Opt-out if it blocks you: `useHermesV1` in `expo-build-properties`.

- iOS minimum rises to 16.4 (drops iPhone 7 / SE 1st gen).

- `expo/fetch` becomes the global `fetch`. Axios uses XHR so you're probably unaffected; if networking acts strangely

set `EXPO_PUBLIC_USE_RN_FETCH=1`.

  

### `@react-navigation` import tracker (37 files)

  

Each file below imported from `@react-navigation/*` before the upgrade. Tick when its import is moved to

`expo-router` / `expo-router/react-navigation` and the file compiles.

  

Routes / layouts

  

- [ ] `app/_layout.tsx` (`ThemeProvider` — most important)

- [ ] `app/(protected)/_layout.tsx` (`Tabs` + custom `tabBar`)

- [ ] `app/(protected)/genie/_layout.tsx`

- [ ] `app/(protected)/home/_layout.tsx`

- [ ] `app/(protected)/marketplace/_layout.tsx`

- [ ] `app/(protected)/profile/_layout.tsx`

  

Theme / navigation core

  

- [ ] `src/theme/colors.ts` (`CustomTheme` extends the navigation theme type)

- [ ] `src/hooks/useHideTabBarOnScroll.ts`

- [ ] `globals/components/CustomTabBar/index.tsx` (`BottomTabBarProps`)

- [ ] `globals/components/Hoc/WithStyleSheet.tsx` (`useTheme` — used by most components)

  

`globals/components/`

  

- [ ] `AppToolTip/index.tsx`

- [ ] `CustomText/index.tsx`

- [ ] `HtmlText/index.tsx`

- [ ] `LoadingSpinner/index.tsx`

- [ ] `MarkLogo/index.tsx`

- [ ] `Modal/BackgroundScaleHost.tsx`

- [ ] `MoneyDisplay/index.tsx`

- [ ] `RunPnLViewer/index.tsx`

- [ ] `RunStatus/index.tsx`

- [ ] `StatusBar/index.tsx`

  

`components/`

  

- [ ] `CustomHeaders/Header/index.tsx`

- [ ] `DashboardComponents/FlatTransactions/Card/WithStatus.tsx`

- [ ] `DashboardComponents/FlatTransactions/OrderStatus/index.tsx`

- [ ] `DashboardComponents/MarketData/TickAndChangeDisplay.tsx`

- [ ] `DashboardComponents/MarketData/index.tsx`

- [ ] `DashboardComponents/RecentRuns/SwipeActions/index.tsx`

- [ ] `DashboardComponents/RecentRuns/SwipeHintBanner/index.tsx`

- [ ] `DashboardComponents/RunDetailContent/index.tsx`

- [ ] `DashboardComponents/RunHero/RunHeroOverlay.tsx`

- [ ] `MarketPlaceComponents/StrategyDetails/DeploymentInformation/index.tsx`

- [ ] `MarketPlaceComponents/StrategyLists/StrategyCard/StrategyCardHeader.tsx`

- [ ] `MarketPlaceComponents/StrategyLists/StrategyCard/index.tsx`

- [ ] `ProfileComponents/PlansQuota/index.tsx`

- [ ] `ResultComponents/ResultStackLayout.tsx`

- [ ] `ResultComponents/RunshowDetails/FilterField.tsx`

- [ ] `TransactionDetails/index.tsx`

  

### Error log — hop 2

  

```

Command:

Platform:

Error output:

  

What I tried:

```

  

---

  

## 3. Hop 3 — SDK 57

  

- [x] `npx expo install expo@^57.0.0 --fix` (should resolve to 57.0.31)

- [x] `npx expo-doctor@latest`

- [x] `rm -rf ios android && npx expo prebuild --clean`

- [x] `npm run ios` builds

- [x] `npm run android` builds

- [x] Smoke test (section 5) passes

- [x] Commit: `SDK 57`

  

Expect little breakage here (57 has no breaking changes over 56).

  

### Error log — hop 3

  

```

Command:

Platform:

Error output:

  

What I tried:

```

  

---

  

## 4. Final checks

  

- [ ] Typecheck — only new errors:

```bash

npx tsc --noEmit --ignoreDeprecations 5.0 2>&1 | grep 'error TS' | sort > ~/tsc-after.txt

comm -13 ~/tsc-before.txt ~/tsc-after.txt

```

- [ ] `npm run lint`

- [ ] `eas build -p android --profile apk` (release-mode build)

- [ ] iOS EAS build (or archive)

- [ ] Merge

  

### Error log — final checks

  

```

Command:

Platform:

Error output:

  

What I tried:

```

  

---

  

## 5. Smoke test

  

- [x] Login (including OAuth return to `app/auth/callback.tsx`)

- [x] Home and runs screens

- [x] Live ticker websocket (prices update)

- [ ] Push notification tap routing

- [ ] Offline modal (netinfo goes 11 → 12 in SDK 56)

- [ ] Bottom sheets / action sheets (`react-native-actions-sheet`)

- [ ] NativeWind styling in light and dark

- [ ] Tab bar and stack navigation

- [ ] Lottie, WebView, in-app browser

- [ ] Walkthrough tooltips (`react-native-walkthrough-tooltip`)

  

---

  

## Reference

  

### What `expo install --fix` changes (33 packages)

  

- Core: `react` 19.1.0 → 19.2.3, `react-native` 0.81.5 → 0.86.3

- Animation / gesture: `react-native-reanimated` 4.1 → 4.5.1, `react-native-worklets` 0.5.1 → 0.10.1,

`react-native-gesture-handler` 2.28 → 2.32, `react-native-screens` 4.16 → 4.26,

`react-native-safe-area-context` 5.6 → 5.7

- Renumbered Expo packages: `expo-router` 6 → 57, `expo-notifications` 0.32 → 57, all other `expo-*` → 57.x

- Majors: `@react-native-community/netinfo` 11 → 12, `react-native-keyboard-controller` 1.18.5 → 1.21.9

- Dev: `eslint-config-expo` → ~57.0.2, `jest-expo` → ~57.0.6

  

### Not managed by Expo — check these yourself if they misbehave

  

`nativewind`, `react-native-actions-sheet` (0.9.8), `react-native-reanimated-carousel`,

`react-native-walkthrough-tooltip`, `@react-native-firebase/*`, `react-native-inappbrowser-reborn`,

`react-native-skeleton-placeholder`, `react-native-calendars`, `react-native-render-html`

  

### Deferred to SDK 58 (not part of this upgrade)

  

- `InteractionManager` is removed in RN 0.88: used in `components/NotificationComponents/useNotification.ts`, and

internally by `react-native-walkthrough-tooltip` (`useInteractionManager` prop in `globals/components/AppToolTip`)

- Node 22.13+ required

- Android R8 on by default in release builds (may need ProGuard keep rules)

- Strict TypeScript API: refs need instance types (`TextInputInstance`, `ViewInstance`)

- iOS scene lifecycle / Xcode 27

  

### Rollback

  

```bash

git reset --hard HEAD~1 && rm -rf node_modules && npm install

```

  

`ios/` and `android/` are gitignored and regenerated by prebuild, so there is nothing to restore there.