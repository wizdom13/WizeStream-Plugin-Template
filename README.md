# Native WizeStream plugin template

This is a standalone Gradle project for **WizeStream plugin API 2.0**. It builds a signed
`.ppp` package and an `index.json` catalog. The sample registers **WizeStream Example** with
a reserved example URL; its extraction methods are intentionally unfinished.

For installation, compatibility, profile behavior, and publication guidance, read
[the plugin guide](../docs/plugins.md).

## Build in the WizeStream checkout

Use JDK 21 and the Android SDK versions in `gradle.properties`. The parent repository's
Gradle wrapper supplies the supported Gradle version.

```sh
# Run these commands from the WizeStream repository root.
./gradlew :app:pluginSdkJar
./gradlew -p plugin-template generateSigningKey repositoryIndex
```

The output is:

- `build/repo/wizestream-example-1.ppp`
- `build/repo/index.json`

`generateSigningKey` creates `signing-key.pk8` and `public-key.der` in this project on the
first run, and reuses them on later runs. Back up both files before publishing. They are
excluded from Git. The private key is PKCS#8 DER; the public file is X.509 DER. The index
contains the raw public key in the format the host expects.

To build a subsequent version with the existing key, update the manifest's `versionCode`
and `versionName`, then run:

```sh
./gradlew -p plugin-template repositoryIndex
```

The package is deterministic for the same source and build inputs; a repository's signatures
also depend on its persistent signing key. The build signs the raw SHA-256 digest bytes with
Ed25519 and verifies its own signature before writing the index.

## Use as a separate project

Copy this directory, preserving `LICENSE`. Supply a previously exported SDK JAR and use
Gradle 9.5.1, or copy the parent repository's Gradle wrapper files (`gradlew`, `gradlew.bat`,
and `gradle/wrapper/`) into the copied project.

```sh
gradle -PpluginSdkJar=/absolute/path/wizestream-plugin-api.jar \
  generateSigningKey repositoryIndex
```

Set `ANDROID_HOME`, or put `sdk.dir=/absolute/path/android-sdk` in a local, untracked
`local.properties`. Install the platform and build tools named in `gradle.properties`.
Use `-PpluginSigningKey=/private/path/key.pk8` and
`-PpluginPublicKey=/private/path/public.der` if keys are stored elsewhere.

## Replace the sample

1. Choose a permanent, unique `pluginId` and a service name that WizeStream does not already provide.
2. Rename `ExamplePlugin` and `ExampleService`, and update `entryClass` and `name` in `plugin.json`.
3. Implement the URL handlers and extractors marked `TODO`. Advertise only supported capabilities.
4. Set the actual author, description, minimum host version, and any optional manifest settings.
5. Test on the minimum Android version and against a release build of the host before publishing.

The manifest keeps `host: "wizestream"`, `apiVersion: "2.0"`, and `minAndroidSdk: 24`.
Its current `minHostVersionCode: 1051000` identifies the development baseline; replace it
with the first published WizeStream version that contains the API your plugin needs.
An earlier build does not gain a plugin runtime merely because its version code matches.

Native plugin DEX uses `pluginMinSdk=23` and the host's `desugar_jdk_libs_configuration_nio`
profile. This matches WizeStream's compiled SDK interfaces and date/time APIs. It is separate
from the manifest's Android-7 device requirement. Keep the compiler profile unchanged; if a
service needs a newer Android version, raise `minAndroidSdk` in its manifest instead.

The SDK, Kotlin standard library, nanojson, jsoup, and commons-lang3 are `compileOnly`.
The host provides them. Do not package replacement copies or import internal activities,
database classes, or player implementation classes. A plugin should use the public extractor
and plugin contracts.

## plugin.json

| Field | Rules |
| ----- | ----- |
| `pluginId` | Reverse domain, `[a-z0-9_.]`, at most 128 chars. Never change it after release. |
| `name` | Must equal the `StreamingService` name. |
| `versionCode` | Integer, increase with every release. |
| `versionName` | Shown to users. |
| `apiVersion` | `major.minor` of the plugin API you build against. Currently `1.0`. A host loads plugins with the same major and a minor not above its own. |
| `author` | Required. Shown in the plugin list. |
| `entryClass` | Class implementing `PipePipePlugin`. |
| `issueUrl` | Optional, https. Where users report bugs of your plugin. When a failure comes from your code, PipePipe sends the user here instead of its own tracker; without it they are told to reach you by the author name. |
| `description` | Optional. |
| `icon` | Optional, path of a PNG/WebP in the package. Square, single color, transparent background, at most 512 px. The app tints it. |
| `settings` | Optional list of `string` or `boolean` settings. The app renders them; read the values with `context.getPreferences()` (see `MyService.getBaseUrl()`). |
| `login` | Optional cookie login: `{"url": "https://...", "cookie": "<cookie name>", "key": "<preference key>"}`. The settings page gets Login and Log out buttons. Login opens `url` in a web view and, once the cookie is set, stores the cookie header under `key` (default `login_cookies`); log out stores an empty string. Read it with `context.getPreferences()`, `addChangeListener` reports changes. `url` must be https. |
| `theme` | Optional primary colors, applied by the app as an overlay on its light, dark and black themes while your service is selected: `{"light": "#F57C00", "dark": "#A35300"}`. Colors are `#RRGGBB`. At least one of `light` and `dark` is needed, a missing one is taken from the other; the black theme uses `dark`. The status bar uses the primary color; the accent color is black in the light theme and white in the dark theme, and cannot be configured. |

The `serviceId` passed to `createService` must be handed to the `StreamingService` constructor unchanged. It is derived from `pluginId` by the host, so it is the same on every device and never changes for a given `pluginId`.
