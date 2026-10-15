# Use Expo commands for Android development

*Last updated: 15 October 2026*

Use this guide to choose the right command when setting up and developing an Expo-based React Native app for Android. Run every command from the project root, where `package.json` is located.

## What you need

- **Node.js and npm**.
- **An existing Expo project** with the `expo` package in its dependencies.
- **Expo Go or a development build** on the Android device if you only need to run the Metro development server.
- **The Android development environment**—including a JDK and the Android SDK—for a local native build. Connect a physical device with USB debugging enabled, or start an Android emulator.

## Which command should I use?

| Situation | Command |
|---|---|
| Set up a checkout that has `package-lock.json` | `npm ci` |
| Set up a project that does not have `package-lock.json` | `npm install` |
| Check that package versions match the installed Expo SDK | `npx expo install --check` |
| Start normal JavaScript or TypeScript development | `npx expo start` |
| Restart Metro with an empty bundler cache | `npx expo start -c` |
| Let a device connect when LAN access does not work | `npx expo start --tunnel` |
| Compile, install and run the native Android app on a chosen device | `npx expo run:android --device` |
| Generate or update only the Android native project | `npx expo prebuild --platform android` |
| Delete and regenerate the Android native project | `npx expo prebuild --platform android --clean` |

## Step 1: Install and check the dependencies

If the project has `package-lock.json`, install the exact dependency tree recorded in it:

```bash
npm ci
```

`npm ci` is best for a fresh checkout and for reproducible installs. It removes the existing `node_modules` directory before installing, never changes `package.json` or `package-lock.json`, and fails if those two files disagree. Do not delete the lockfile merely to get around that failure; find out why the files are out of sync.

If the project does not have `package-lock.json`, resolve the dependencies from `package.json` and create the lockfile:

```bash
npm install
```

Use `npm install` rather than `npm ci` when intentionally adding, removing or updating dependencies because the lockfile may need to change.

After installation, ask Expo to compare the installed package versions with the versions expected by the project's Expo SDK:

```bash
npx expo install --check
```

The command reports incompatible versions and, in an interactive terminal, offers to install compatible versions. Review any proposed changes before accepting them because they can update the dependency files.

!!! note "Use `npx`, not `npm`"
    The Expo CLI belongs to the project's `expo` package. `npx expo ...` runs that CLI; `npm expo ...` is not the correct command.

## Step 2: Start the development server

For the normal JavaScript or TypeScript development loop, start Metro:

```bash
npx expo start
```

Scan the QR code with Expo Go or a development build. If an Android emulator is running, press `a` in the Expo terminal to open the app. Keep the server running while developing, and stop it with `Ctrl+C`.

This command serves the JavaScript bundle; it does not compile a new native Android binary. Most JavaScript, TypeScript and asset changes only need this command and Fast Refresh.

If Metro appears to serve stale code or assets, stop it and restart with an empty bundler cache:

```bash
npx expo start -c
```

Use `-c` for cache-related symptoms, not for every start—the next startup is slower because Metro must rebuild the cache.

If a physical device cannot reach the development server over the local network, use a tunnel:

```bash
npx expo start --tunnel
```

Try the normal LAN connection first. A tunnel is useful on restrictive Wi-Fi, across different networks or when a firewall blocks LAN access, but reloads are slower and both devices need internet access. Expo's tunnel support also requires `@expo/ngrok` to be installed.

If Expo reports that the tunnel dependency is missing, install it once on the development computer:

```bash
npm install --global @expo/ngrok
```

## Step 3: Build and run the native Android app

To choose a connected physical device or emulator, compile the native Android project, install its debug app and start Metro, run:

```bash
npx expo run:android --device
```

Use a native build for the first local development build and after a change that affects the native runtime, such as adding a native library or changing native app configuration. For JavaScript-only changes after the app is installed, return to `npx expo start` instead of recompiling it.

If `android/` does not exist, `run:android` runs prebuild automatically to generate it. If the directory already exists, Expo skips that automatic generation; regenerate it yourself when a native dependency or app configuration change needs to be applied.

## Step 4: Regenerate the Android project

To generate `android/`, or layer the current Expo configuration and config-plugin changes over an existing directory, run:

```bash
npx expo prebuild --platform android
```

This is faster than a clean prebuild, but repeated changes can leave stale output, and some config plugins do not produce the same result when applied more than once.

For projects that use Continuous Native Generation and treat `android/` as generated output, the safer way to apply native configuration changes is:

```bash
npx expo prebuild --platform android --clean
```

`--clean` deletes the existing `android/` directory and generates it again from the Expo app configuration and config plugins. Save or commit the project first. Any manual edits inside `android/` that are not represented in the app configuration or a config plugin will be lost.

After prebuild, run `npx expo run:android --device` to compile and install the regenerated project.

!!! warning "Do not clean a manually maintained native project"
    If the project keeps intentional hand-written changes in `android/`, do not use `--clean` unless those changes have first been moved into config plugins or you are prepared to reapply them.

## References

- [npm: `npm ci`](https://docs.npmjs.com/cli/commands/npm-ci/)
- [npm: `npm install`](https://docs.npmjs.com/cli/commands/npm-install/)
- [Expo CLI: start, install and compile commands](https://docs.expo.dev/more/expo-cli/)
- [Expo: Start developing](https://docs.expo.dev/get-started/start-developing/)
- [Expo: Create a debug build locally](https://docs.expo.dev/guides/local-app-development/)
- [Expo: Continuous Native Generation](https://docs.expo.dev/workflow/continuous-native-generation/)
