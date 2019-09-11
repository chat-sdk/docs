# Supporting Android X

Update your multidex version in the app level `build.gradle`:

```
implementation 'androidx.multidex:multidex:2.0.0'
```

Update your `gradle.properties` file:

```
android.useAndroidX=true
android.enableJetifier=true
```