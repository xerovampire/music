# GitHub Actions APK Building Guide

This document explains how to build the Metrolist APK using GitHub Actions.

## Quick Start

The GitHub Actions workflow automatically builds the APK on:
- Push to `main` or `develop` branches
- Pull requests to `main` or `develop` branches  
- Manual workflow dispatch (via GitHub UI)

## Available Builds

### 1. **Unsigned Release APK** (Automatic)
- Built on every push and PR
- FOSS variant (no Google Play Services)
- GMS variant (with Google Play Services) - optional
- Artifacts available in Actions tab for 90 days

### 2. **Signed Release APK** (With Keystore)
- Built when pushing to `main` branch (if keystore configured)
- Requires GitHub Secrets setup
- Ready for direct installation or Play Store submission

## Setting Up Signed Builds

### Step 1: Create or Use Existing Keystore

If you don't have a keystore:

```bash
keytool -genkey -v -keystore release.keystore \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias metrolist-key
```

### Step 2: Encode Keystore to Base64

```bash
base64 release.keystore > keystore.base64
```

On Windows PowerShell:
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("release.keystore")) | Out-File keystore.base64 -NoNewline
```

### Step 3: Add GitHub Secrets

Go to your GitHub repository:
1. Settings → Secrets and variables → Actions
2. Click "New repository secret" and add:

| Secret Name | Value |
|------------|-------|
| `KEYSTORE_BASE64` | Content of keystore.base64 file |
| `KEYSTORE_PASSWORD` | Your keystore password |
| `KEY_ALIAS` | The alias used when creating keystore |
| `KEY_PASSWORD` | The key password (usually same as keystore) |

### Step 4: Update Gradle Build Config

Edit [Metrolist/app/build.gradle.kts](../app/build.gradle.kts) to add signing config:

```kotlin
signingConfigs {
    create("release") {
        storeFile = file(System.getenv("KEYSTORE_PATH") ?: "release.keystore")
        storePassword = System.getenv("KEYSTORE_PASSWORD")
        keyAlias = System.getenv("KEY_ALIAS")
        keyPassword = System.getenv("KEY_PASSWORD")
    }
}

buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
        minifyEnabled = true
        shrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
    }
}
```

## Workflow File

The GitHub Actions workflow is defined in [.github/workflows/build-apk.yml](.github/workflows/build-apk.yml)

### Key Features:

- **Automatic builds** on push and PR
- **Gradle caching** for faster builds (~2-3 min)
- **Multiple variants** support (FOSS and GMS)
- **Artifact uploads** for testing
- **Tag-based releases** (automatic release creation)
- **Error handling** with continue-on-error options

## Build Variants

The app supports multiple build variants:

1. **FOSS Variant** (Free and Open Source Software)
   - No Google Play Services
   - F-Droid compatible
   - Command: `./gradlew assembleFossRelease`

2. **GMS Variant** (Google Mobile Services)
   - Includes Google Play Services
   - Crash reporting and analytics
   - Command: `./gradlew assembleGmsRelease`

## Local Building

### Without Signing
```bash
cd Metrolist
./gradlew assembleRelease
```

### With Signing (local)
```bash
# Set environment variables
export KEYSTORE_PATH=./release.keystore
export KEYSTORE_PASSWORD=your_password
export KEY_ALIAS=metrolist-key
export KEY_PASSWORD=your_key_password

./gradlew assembleRelease
```

### Debug Build
```bash
./gradlew assembleDebug
```

## Artifacts Location

After workflow completes:
1. Go to Actions tab in your repository
2. Click the workflow run
3. Scroll to "Artifacts" section
4. Download `apk-builds` or `signed-apk`

## Release Management

### Creating a Release

1. Tag a commit:
```bash
git tag v1.0.0
git push origin v1.0.0
```

2. GitHub Actions automatically:
   - Builds the APK
   - Creates a GitHub Release
   - Attaches built APKs

3. The release is visible at: `Releases` tab

### Version Management

Update version in [Metrolist/app/build.gradle.kts](../app/build.gradle.kts):

```kotlin
defaultConfig {
    versionCode = 140      // Increment for each release
    versionName = "12.12.5" // Semantic versioning
}
```

## Troubleshooting

### Build Fails with Java Error
- Ensure Java 17+ is available
- Check `org.gradle.jvmargs` in [gradle.properties](../gradle.properties)
- Current: `-Xmx4096M` (adjust if low memory)

### APK Not Found
- Check the variant name matches your build
- Verify `assembleRelease` task completed
- Look in `app/build/outputs/apk/*/release/`

### Signing Fails
- Verify all 4 secrets are set correctly
- Check keystore password is correct
- Ensure key alias matches

### Slow Builds
- First build is slower (downloads dependencies)
- Gradle cache helps subsequent builds
- Local SSD storage improves performance
- Consider increasing `org.gradle.jvmargs` on CI

## Best Practices

1. **Never commit keystore** - use GitHub Secrets
2. **Tag releases** - use semantic versioning
3. **Monitor build times** - adjust cache settings if needed
4. **Test locally first** - before pushing to main
5. **Keep dependencies updated** - use Renovate bot for PRs

## Additional Resources

- [Gradle Documentation](https://gradle.org/docs/)
- [Android Build Documentation](https://developer.android.com/build)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Android Signing Guide](https://developer.android.com/studio/publish/app-signing)

---

For more information, check [development_guide.md](../development_guide.md)
