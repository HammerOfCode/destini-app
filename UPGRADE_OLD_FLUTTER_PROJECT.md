# Upgrading an Old Flutter Course Project to Run on a Modern Toolchain

These steps fix the "SDK / JDK mismatch" errors you get when opening an old (2019-era) Flutter starter project with a current Flutter install (e.g. Flutter 3.41 / Dart 3.11).

## Why it fails

Two separate problems stack on top of each other:

1. **The `android/` folder is outdated.** It uses Gradle 5.x, Android Gradle Plugin (AGP) 3.x and the old v1 Flutter embedding (`io.flutter.app.FlutterActivity`). Current Flutter no longer supports any of that.
2. **The JDK doesn't match Gradle.** Flutter uses the JDK that `flutter doctor -v` reports, which is usually Android Studio's bundled one, not your system `JAVA_HOME`.
   - Modern AGP (8.x) needs **JDK 17 or newer**.
   - Gradle 8.14 can only *run* on **JDK 24 or older**. Running on JDK 25 fails with an error that just says `25.0.3`.

Changing the JDK alone can't fix problem 1. You have to do both parts below.

---

## Step 1: Diagnose

From the project root, run:

```powershell
flutter --version
flutter doctor -v
```

In the **Android toolchain** section, find the `Java binary at:` and `Java version` lines. That is the JDK Flutter actually uses.

Then check the project's Android versions:

```powershell
Get-Content android/gradle/wrapper/gradle-wrapper.properties   # Gradle version
Get-Content android/build.gradle                                # AGP version (classpath 'com.android.tools.build:gradle:X')
```

If you see Gradle 5.x/6.x or AGP 3.x/4.x, continue with the steps below.

## Step 2: Back up the old `android/` folder

```powershell
Copy-Item android ..\android_backup -Recurse
```

This matters if you customized app icons or other resources.

## Step 3: Delete and regenerate `android/`

```powershell
Remove-Item android -Recurse -Force
flutter create --platforms=android --org co.appbrewery --project-name destini_challenge_starting .
```

- `--org` plus the project name produce the package ID. Choose them so it matches the old `applicationId` (here: `co.appbrewery.destinichallengestarting`).
- The `.` at the end means "the current folder". `flutter create` only adds the missing files and leaves your `lib/` code alone.
- This may end with a `pub get` error about null safety. That's expected; Step 5 fixes it.

## Step 4: Restore custom app icons

Copy the old icon folders back over the generated ones:

```powershell
Copy-Item ..\android_backup\app\src\main\res\mipmap-* android\app\src\main\res\ -Recurse -Force
```

If the old project had `mipmap-anydpi-v26` (adaptive icons), also restore whatever drawable it references, typically:

```powershell
Copy-Item ..\android_backup\app\src\main\res\drawable\ic_launcher_background.xml android\app\src\main\res\drawable\
```

Otherwise the build fails with a "resource not found" error.

## Step 5: Update `pubspec.yaml` for Dart 3

Dart 3 only supports null-safe code, so the old SDK constraint blocks `pub get`. Change:

```yaml
environment:
  sdk: ">=2.1.0 <3.0.0"
```

to:

```yaml
environment:
  sdk: ">=3.0.0 <4.0.0"
```

Also bump old packages, for example:

```yaml
cupertino_icons: ^1.0.8
```

Then run:

```powershell
flutter pub get
```

## Step 6: Clean up generated extras (optional)

`flutter create` may add files the original project didn't have:

- `test/widget_test.dart` refers to a `MyApp` class that doesn't exist in this project. Delete it or fix it.
- `analysis_options.yaml` expects the `flutter_lints` package. Delete it, or add `flutter_lints` under `dev_dependencies`.

## Step 7: Replace removed widgets in Dart code

`FlatButton`, `RaisedButton` and `OutlineButton` were removed from Flutter. Replace them:

| Old             | New              |
|-----------------|------------------|
| `FlatButton`    | `TextButton`     |
| `RaisedButton`  | `ElevatedButton` |
| `OutlineButton` | `OutlinedButton` |

`color:` is no longer a direct property; it moves into `style:`:

```dart
// Before
FlatButton(
  onPressed: () {},
  color: Colors.red,
  child: Text('Choice 1'),
)

// After
TextButton(
  onPressed: () {},
  style: TextButton.styleFrom(
    backgroundColor: Colors.red,
    foregroundColor: Colors.white,
  ),
  child: Text('Choice 1'),
)
```

Check for remaining problems with:

```powershell
flutter analyze
```

## Step 8: Fix the JDK version

See which Gradle version was generated:

```powershell
Get-Content android/gradle/wrapper/gradle-wrapper.properties
```

Then make sure Gradle runs on a compatible JDK. For Gradle 8.14 with AGP 8.x, that means **JDK 17 to 24**. JDK 17 or 21 (LTS releases) are the safest choices.

List the JDKs you have installed:

```powershell
Get-ChildItem "C:\Program Files\Java", "C:\Program Files\Microsoft", "C:\Program Files\Eclipse Adoptium" -Directory -ErrorAction SilentlyContinue
```

If you don't have one, install JDK 17 or 21 (e.g. Eclipse Temurin or Microsoft OpenJDK).

Then choose **one** of these options:

**Option A: this project only (recommended).** Add this line to `android/gradle.properties`, using forward slashes:

```properties
org.gradle.java.home=C:/Program Files/Java/jdk-20
```

**Option B: all Flutter projects.**

```powershell
flutter config --jdk-dir="C:\Program Files\Java\jdk-21"
```

**Option C: newer Gradle instead of an older JDK.** Keep JDK 25 and raise `distributionUrl` in `gradle-wrapper.properties` to Gradle **9.1 or newer**. AGP / Kotlin plugin compatibility with Gradle 9 may need extra version bumps in `android/settings.gradle.kts`.

## Step 9: Build and run

```powershell
flutter clean
flutter pub get
flutter build apk --debug    # verifies Gradle works; the first run downloads Gradle and can take several minutes
```

When you see `✓ Built build\app\outputs\flutter-apk\app-debug.apk`, start the emulator and run:

```powershell
flutter devices
flutter run
```

Warnings like `A restricted method in java.lang.System has been called` are harmless.

---

## Extra: null safety while following the course

Old courses define classes without types or `required`. In Dart 3 you need:

```dart
class Story {
  String storyTitle;
  String choice1;
  String choice2;

  Story({required this.storyTitle, required this.choice1, required this.choice2});
}
```

## Quick reference: compatible versions

| Component      | Version used here |
|----------------|-------------------|
| Flutter / Dart | 3.41.5 / 3.11.3   |
| Gradle         | 8.14              |
| AGP            | 8.11.1            |
| Kotlin plugin  | 2.2.20            |
| JDK for Gradle | 17–24 (used 20)   |
