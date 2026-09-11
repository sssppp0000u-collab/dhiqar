name: Build Cyber MyHD Root Fixed

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

    - name: Extract & Move Project to Root Directory
      run: |
        sudo apt-get update && sudo apt-get install -y unzip
        
        # 1. فك ضغط الملفات إن وجدت
        for f in *.zip; do
          [ -e "$f" ] || continue
          unzip -o "$f" -d extracted_app || true
        done

        # 2. البحث عن المجلد الحقيقي للمشروع ونقله للجذر
        REAL_ROOT=$(find . -maxdepth 4 -name "settings.gradle" -o -name "settings.gradle.kts" -o -name "build.gradle" | head -n 1 | xargs dirname)

        if [ -n "$REAL_ROOT" ] && [ "$REAL_ROOT" != "." ]; then
          echo "Moving files from $REAL_ROOT to root directory..."
          cp -r "$REAL_ROOT"/* . 2>/dev/null || true
        fi

        # 3. إنشاء ملف settings.gradle في الجذر إذا كان مفقوداً
        if [ ! -f "settings.gradle" ] && [ ! -f "settings.gradle.kts" ]; then
          echo "include ':app'" > settings.gradle
        fi

    - name: Inject MyHD Subscription & Cyber Theme
      run: |
        cat << 'EOF' > patch.py
        import glob, os, re

        PIN = "2027"
        MYHD = "356288617436"

        # ربط الرمز 2027 باشتراك MyHD
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
                    with open(p, "w", encoding="utf-8") as f: f.write(txt)
            except: pass

        # تطبيق ثيم النيون Cyber
        colors = '<?xml version="1.0" encoding="utf-8"?><resources><color name="colorPrimary">#00F0FF</color><color name="colorPrimaryDark">#030508</color><color name="colorAccent">#00FF66</color><color name="backgroundColor">#020305</color><color name="cardBg">#0A0F1D</color><color name="textColorPrimary">#FFFFFF</color><color name="textColorSecondary">#00FF66</color></resources>'
        for c in glob.glob("**/res/values/colors.xml", recursive=True):
            if "build/" not in c:
                try: open(c, "w", encoding="utf-8").write(colors)
                except: pass
        EOF
        python3 patch.py

    - name: Ensure Gradle Wrapper Exists
      run: |
        if [ ! -f "./gradlew" ]; then
          gradle wrapper --gradle-version 8.5
        fi
        chmod +x gradlew

    - name: Build Debug APK
      run: |
        ./gradlew assembleDebug --no-daemon --stacktrace

    - name: Upload APK Artifact
      uses: actions/upload-artifact@v4
      with:
        name: DHIQAR-TV-FINAL
        path: "**/build/outputs/apk/debug/*.apk"
