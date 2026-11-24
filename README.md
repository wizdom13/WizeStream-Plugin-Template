# WizeStream Plugin Template

A standalone Kotlin project for **native WizeStream plugin API 2.0**. It builds a signed
`.ppp` package and a JSON repository index that WizeStream can install.

The included **WizeStream Example** service is a developer skeleton. Its URL handlers and
extractors still need implementation; the example does not provide a working streaming service.

- [WizeStream application](https://github.com/wizdom13/WizeStream)
- [Plugin installation and API guide](https://github.com/wizdom13/WizeStream/blob/pipe/docs/plugins.md)
- [WizeStream Plugins catalog](https://github.com/wizdom13/WizeStream-Plugins)

## Requirements

- JDK 21.
- Gradle **9.5.1**, available through the WizeStream application's Gradle wrapper.
- Android SDK platform **37.0** and Build Tools **36.1.0**, as declared in
  [gradle.properties](gradle.properties).
- A `wizestream-plugin-api.jar` exported from a compatible WizeStream checkout.
- A WizeStream build that includes native plugin support, on Android **7/API 24 or newer**,
  for installation and testing.

This standalone repository does **not** include a Gradle wrapper. The commands below use the
wrapper from a sibling WizeStream checkout. Building the host SDK also requires the components
listed in WizeStream's [build guide](https://github.com/wizdom13/WizeStream/blob/pipe/BUILDING.md).

Set `ANDROID_HOME` to your Android SDK, or create an untracked `local.properties` in this
project containing `sdk.dir=/absolute/path/to/android-sdk`. Use forward slashes in a Windows
properties-file path, for example `sdk.dir=C:/Android/Sdk`. The host checkout needs its own SDK
configuration if you do not use the environment variable.

## Build the sample

Keep these repositories in the same parent directory:

- `WizeStream` — the application and SDK exporter, checked out on `pipe`.
- `WizeStream-Plugin-Template` — this standalone project.

Clone either repository if it is not already present:

```sh
git clone --branch pipe https://github.com/wizdom13/WizeStream.git
git clone https://github.com/wizdom13/WizeStream-Plugin-Template.git
```

### Windows PowerShell

From that parent directory:

```powershell
Set-Location .\WizeStream
.\gradlew.bat :app:pluginSdkJar
$pluginSdkJar = (Resolve-Path .\app\build\outputs\plugin-sdk\wizestream-plugin-api.jar).Path
.\gradlew.bat -p ..\WizeStream-Plugin-Template "-PpluginSdkJar=$pluginSdkJar" generateSigningKey repositoryIndex
```

### Linux or macOS

From that parent directory:

```sh
cd WizeStream
./gradlew :app:pluginSdkJar
plugin_sdk_jar="$PWD/app/build/outputs/plugin-sdk/wizestream-plugin-api.jar"
./gradlew -p ../WizeStream-Plugin-Template "-PpluginSdkJar=$plugin_sdk_jar" generateSigningKey repositoryIndex
```

The exported SDK is `WizeStream/app/build/outputs/plugin-sdk/wizestream-plugin-api.jar`.
The sample outputs are under **this template's** directory:

- `build/repo/wizestream-example-1.ppp`
- `build/repo/index.json`

The package filename uses `rootProject.name` from [settings.gradle.kts](settings.gradle.kts)
and the manifest's `versionCode`.

If you already have the SDK JAR and Gradle 9.5.1 installed, you can instead build directly
from this project's root:

```sh
gradle "-PpluginSdkJar=/absolute/path/wizestream-plugin-api.jar" generateSigningKey repositoryIndex
```

Always supply `pluginSdkJar` for a standalone checkout. The build's default relative SDK path
is intended for the template embedded inside the WizeStream application repository.

## Keep the signing identity

`generateSigningKey` creates `signing-key.pk8` and `public-key.der` in this project on the first
run. It reuses an existing complete pair and refuses to replace an incomplete pair. Both files
are excluded by [.gitignore](.gitignore); back them up before publishing.

The private key is PKCS#8 DER and the public file is X.509 DER. To keep them elsewhere, supply
both `-PpluginSigningKey=/private/path/key.pk8` and
`-PpluginPublicKey=/private/path/public.der` to the build commands.

For each update, retain the plugin ID and key pair, increase `versionCode`, update
`versionName`, and rerun the same packaging command with **`repositoryIndex` only**.
`repositoryIndex` builds the package, signs its raw 32-byte SHA-256 digest using Ed25519,
verifies that signature, and writes the digest, signature and raw public key into the index.

WizeStream pins the publisher key on first installation. Updates must keep that key and use a
higher version code. Uninstalling preserves the publisher pin; generating a new pair is not a
supported way to update an existing plugin.

## Implement your service

1. Choose a permanent, unique `pluginId` and a service name WizeStream does not already provide.
2. Rename the package/classes in [ExamplePlugin.kt](src/main/kotlin/org/example/wizestream/demo/ExamplePlugin.kt)
   and [ExampleService.kt](src/main/kotlin/org/example/wizestream/demo/ExampleService.kt).
3. Update `entryClass`, `name`, author and description in
   [plugin.json](src/main/resources/plugin.json). Update `rootProject.name` for your package filename.
4. Implement the marked URL handlers and extractors. Use the host downloader and extractor
   result types, and advertise only the capabilities your service implements.
5. Set the actual host/device requirements, then test the resulting package against compatible
   WizeStream builds before publishing it.

The public entry class must have a no-argument constructor and implement `WizeStreamPlugin`.
Its `createService(serviceId, context)` receives a `WizeStreamPluginContext`.
Pass `serviceId` unchanged to the `StreamingService` constructor and make the service name match
the manifest. The host derives the stable service identity from `pluginId`, so changing the ID
would separate the plugin from its existing subscriptions, history and playlists.

`WizeStreamPluginContext` exposes the plugin identity, host version, a captured profile UUID
string, and `getPreferences()`. Use those preferences for settings and optional credentials.
They belong to the profile that created the service. Installation, updates and profile changes
with loaded plugins require an app restart. Plugins run inside the app process; these APIs
provide profile-aware behavior, not a sandbox for untrusted code.

## Native manifest

The sample's core configuration is:

```json
{
  "pluginId": "org.example.wizestream.demo",
  "name": "WizeStream Example",
  "versionCode": 1,
  "versionName": "1.0.0",
  "host": "wizestream",
  "apiVersion": "2.0",
  "minHostVersionCode": 1051000,
  "minAndroidSdk": 24,
  "requiredCapabilities": ["profile-preferences"],
  "author": "Your name",
  "entryClass": "org.example.wizestream.demo.ExamplePlugin"
}
```

| Field | Native WizeStream behavior |
| --- | --- |
| `pluginId` | Stable reverse-domain ID, at most 128 characters. The parser accepts `^[a-z][a-z0-9_]*(\.[a-z0-9_]+)+$`. |
| `name` | Must match the returned `StreamingService` name. |
| `versionCode` | Positive integer for this template; increase it for every published update. |
| `versionName` | Human-readable release version. |
| `host` | Explicitly `"wizestream"`. Omitting it selects the separate PipePipe-compatible contract. |
| `apiVersion` | `"2.0"` for this template. Its entry interface is `WizeStreamPlugin`. |
| `minHostVersionCode` | Minimum WizeStream version code containing the API your plugin requires. |
| `minAndroidSdk` | Device requirement, at least 24 for external plugins. Raise it if your service needs newer Android APIs. |
| `requiredCapabilities` | Host features required before loading. Native API 2.0 currently exposes `profile-preferences`; unknown required capabilities are rejected. |
| `author` | Required publisher name. |
| `entryClass` | Fully qualified public class implementing `WizeStreamPlugin`, with a no-argument constructor. |
| `description` | Optional service description. |
| `issueUrl` | Optional HTTPS issue-tracker URL, at most 512 characters. |
| `icon` | Optional package-relative image path without `..`. Use a square PNG/WebP with a transparent background, at most 512 pixels per edge. |
| `settings` | Optional `string` or `boolean` preferences with a key, title, optional summary and default value. |
| `login` | Optional cookie metadata containing an HTTPS `url`, required cookie name and preference `key` (default `login_cookies`). See below for the host UI. |
| `theme` | Optional color metadata is parsed for format compatibility; WizeStream retains its own application theme. |

The sample's `minHostVersionCode: 1051000` identifies the development baseline. Before releasing
a plugin, set it to the first published host version that actually contains the API you use.
An older APK does not gain plugin support just because its version code matches that value.

### Settings and optional login cookies

Manifest settings are rendered by the host and stored through `context.getPreferences()`;
`addChangeListener` is available for changes. For cookie authentication, WizeStream exposes
**Login cookies** and **Clear login cookies**. Users supply a cookie header manually; the
settings screen shows the required login URL/cookie name and hides the saved value in its summary.
Changes require a restart because a service may cache its configuration.

This host does not open an automatic plugin login WebView or share its WebView cookie store.
Do not implement login-dependent behavior based on those assumptions. Cookie settings remain
separate per profile, and deleting a profile clears its plugin preferences.

## Compiler and dependency compatibility

Keep `pluginMinSdk=23` and the pinned `desugar_jdk_libs_configuration_nio` profile in
[build.gradle.kts](build.gradle.kts). They match the host's compiled SDK signatures and date/time
APIs. This compiler profile is separate from the manifest's Android 7/API 24 installation minimum.
If your plugin needs a newer device API, raise `minAndroidSdk` in its manifest.

The SDK, Kotlin standard library, nanojson, jsoup and commons-lang3 are `compileOnly` because
the host supplies them. Do not bundle replacement copies of these classes or depend on internal
WizeStream activities, database classes or player implementations. Additional packaged libraries
need deliberate dependency/namespace handling and their applicable license notices.

The `.ppp` is a ZIP containing the root `plugin.json`, JVM classes/resources and Android DEX.
The packaging task rejects host-class namespaces. Test on your minimum supported Android
version and a minified release host; a debug host alone may hide binary-compatibility problems.

## Publish to a repository

1. Replace all sample metadata and unfinished extraction methods.
2. Generate and test the exact `.ppp` you will distribute.
3. Upload `build/repo/index.json` and the generated `.ppp` together to an HTTPS location,
   such as assets in the same GitHub release.
4. Give users the direct JSON URL. A GitHub `/blob/` page is HTML, not a repository index.
5. To request inclusion in [WizeStream Plugins](https://github.com/wizdom13/WizeStream-Plugins),
   submit the generated plugin entry and link its source code. Follow that catalog's contribution rules.

The generated index contains a **relative** `package` filename. It works when the index and
package share the same URL directory. When copying an entry into a different catalog, replace
that field with the package's absolute HTTPS download URL while retaining the generated digest,
signature, public key and compatibility fields. Do not edit the signed package after generating
the index; rebuild and sign again if any package bytes change.

The WizeStream Plugins catalog's direct JSON URL is:

```text
https://raw.githubusercontent.com/wizdom13/WizeStream-Plugins/main/index.json
```

Its catalog can include native API 2.0 plugins and compatible PipePipe plugins. This template
builds the native contract; do not change only `host` or `apiVersion` to claim compatibility with
a different host. Check the [host guide](https://github.com/wizdom13/WizeStream/blob/pipe/docs/plugins.md)
for supported contracts and installation behavior.

## License and attribution

This template carries **GPL-3.0-or-later** source notices. Preserve [LICENSE](LICENSE) and the
source attribution when adapting it. The packaging format and service skeleton derive from the
[PipePipe Plugin Template](https://github.com/InfinityLoop1308/PipePipePluginTemplate/tree/91950887dfd173263ecacfc466437a33777d242d);
the native SDK contract and signing tasks are WizeStream additions.
