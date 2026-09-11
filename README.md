name: Build Cyber MyHD APK Clean

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

    - name: Setup Android SDK
      uses: android-actions/setup-android@v3

    - name: Unzip Project Files If Needed
      run: |
        sudo apt-get update && sudo apt-get install -y unzip
        for f in *.zip; do
          [ -e "$f" ] || continue
          unzip -o "$f" -d extracted_app || true
        done
        if [ -d "extracted_app" ]; then
          cp -rn extracted_app/*/. . 2>/dev/null || cp -rn extracted_app/* . 2>/dev/null || true
        fi

    - name: Fix Gradle Repositories & Inject MyHD
      run: |
        cat << 'EOF' > fix_and_patch.py
        import glob, os, re

        PIN = "2027"
        MYHD = "356288617436"

        # 1. Inject MyHD Code & Cyber Theme
        for p in glob.glob("**/*.java", recursive=True) + glob.glob("**/*.kt", recursive=True):
            if "build/" in p: continue
            try:
                with open(p, "r", encoding="utf-8", errors="ignore") as f:
                    txt = f.read()
                if "getText()" in txt or "toString()" in txt:
                    txt = re.sub(
                        r'(String\s+([a-zA-Z0-9_]+)\s*=\s*[^;]*getText\(\)\.toString\(\)[^;]*;)',
                        r'\1\n        if (\2 != null && (\2.trim().equals("' + PIN + '") || \2.trim().contains("' + PIN + '"))) { \2 = "' + MYHD + '"; }',
                        txt
                    )
                    with open(p, "w", encoding="utf-8") as f:
                        f.write(txt)
            except: pass

        colors = '<?xml version="1.0" encoding="utf-8"?><resources><color name="colorPrimary">#00F0FF</color><color name="colorPrimaryDark">#030508</color><color name="colorAccent">#00FF66</color><color name="backgroundColor">#020305</color><color name="cardBg">#0A0F1D</color><color name="textColorPrimary">#FFFFFF</color><color name="textColorSecondary">#00FF66</color></resources>'
        for c in glob.glob("**/res/values/colors.xml", recursive=True):
            if "build/" not in c:
                try: open(c, "w", encoding="utf-8").write(colors)
                except: pass

        # 2. Fix Gradle Plugins and Repositories
        for g in glob.glob("**/*.gradle*", recursive=True):
            if "build/" in g: continue
            try:
                with open(g, "r", encoding="utf-8", errors="ignore") as f:
                    content = f.read()
                if "repositories {" in content and "google()" not in content:
                    content = content.replace("repositories {", "repositories {\n        google()\n        mavenCentral()\n        gradlePluginPortal()\n")
                    with open(g, "w", encoding="utf-8") as f:
                        f.write(content)
            except: pass
        EOF
        python3 fix_and_patch.py

    - name: Grant Permission
      run: |
        find . -name "gradlew" -exec chmod +x {} \;

    - name: Build Debug APK
      run: |
        GRADLE_BIN=$(find . -name "gradlew" | head -n 1)
        if [ -n "$GRADLE_BIN" ]; then
          cd $(dirname "$GRADLE_BIN")
          ./gradlew assembleDebug --no-daemon --stacktrace
        else
          gradle assembleDebug --no-daemon --stacktrace
        fi

    - name: Upload APK
      uses: actions/upload-artifact@v4
      with:
        name: DHIQAR-TV-FINAL
        path: "**/build/outputs/apk/debug/*.apk"
