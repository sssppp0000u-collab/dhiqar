name: Build APK Direct Clean

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Unzip Project Files If Zip Present
      run: |
        sudo apt-get update && sudo apt-get install -y unzip
        for f in *.zip; do
          [ -e "$f" ] || continue
          unzip -o "$f" -d extracted_app || true
        done
        if [ -d "extracted_app" ]; then
          cp -rn extracted_app/*/. . 2>/dev/null || cp -rn extracted_app/* . 2>/dev/null || true
        fi

    - name: Grant Permission for Gradlew
      run: |
        find . -name "gradlew" -exec chmod +x {} \;

    - name: Build Debug APK
      run: |
        GRADLE_BIN=$(find . -name "gradlew" | head -n 1)
        if [ -n "$GRADLE_BIN" ]; then
          DIR=$(dirname "$GRADLE_BIN")
          cd "$DIR"
          ./gradlew assembleDebug --no-daemon
        else
          gradle assembleDebug --no-daemon
        fi

    - name: Upload APK
      uses: actions/upload-artifact@v4
      with:
        name: DHIQAR-TV-FINAL
        path: "**/build/outputs/apk/debug/*.apk"
