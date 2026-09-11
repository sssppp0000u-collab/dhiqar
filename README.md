name: Build APK

on:
  push:
    branches:
      - main
      - master
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
          distribution: temurin
          java-version: "17"

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4

      # خطوة إضافة ثيم النيون وربط اشتراك MyHD بالرمز 2027
      - name: Inject MyHD Subscription & Cyber Theme
        run: |
          python3 -c "
          import glob, re

          PIN_CODE = '2027'
          MYHD_CODE = '356288617436'

          # 1. ربط الرمز 2027 بكود MyHD
          for path in glob.glob('**/*.java', recursive=True) + glob.glob('**/*.kt', recursive=True):
              if 'build/' in path: continue
              try:
                  with open(path, 'r', encoding='utf-8', errors='ignore') as f:
                      code = f.read()
                  if 'getText()' in code or 'toString()' in code:
                      new_code = re.sub(
                          r'(String\s+([a-zA-Z0-9_]+)\s*=\s*[^;]*getText\(\)\.toString\(\)[^;]*;)',
                          r'\1\n        if (\2 != null && (\2.trim().equals(\"' + PIN_CODE + '\") || \2.trim().contains(\"' + PIN_CODE + '\"))) { \2 = \"' + MYHD_CODE + '\"; }',
                          code
                      )
                      if new_code != code:
                          with open(path, 'w', encoding='utf-8') as f:
                              f.write(new_code)
              except Exception:
                  pass

          # 2. تطبيق ألوان النيون Cyber
          cyber_colors = '''<?xml version=\"1.0\" encoding=\"utf-8\"?>
          <resources>
              <color name=\"colorPrimary\">#00F0FF</color>
              <color name=\"colorPrimaryDark\">#030508</color>
              <color name=\"colorAccent\">#00FF66</color>
              <color name=\"backgroundColor\">#020305</color>
              <color name=\"cardBg\">#0A0F1D</color>
              <color name=\"cardBgFocused\">#002B3D</color>
              <color name=\"textColorPrimary\">#FFFFFF</color>
              <color name=\"textColorSecondary\">#00FF66</color>
              <color name=\"neonCyan\">#00F0FF</color>
              <color name=\"neonGreen\">#00FF66</color>
          </resources>'''

          for c in glob.glob('**/res/values/colors.xml', recursive=True):
              if 'build/' not in c:
                  try: open(c, 'w', encoding='utf-8').write(cyber_colors)
                  except: pass

          # 3. ضبط الشاشة أفقي مريح للتلفزيون
          for m in glob.glob('**/AndroidManifest.xml', recursive=True):
              if 'build/' not in m:
                  try:
                      txt = open(m, 'r', encoding='utf-8', errors='ignore').read()
                      if '<application' in txt and 'supportsRtl' not in txt:
                          txt = txt.replace('<application', '<application android:supportsRtl=\"true\"', 1)
                      if '<activity' in txt and 'screenOrientation' not in txt:
                          txt = txt.replace('<activity', '<activity android:screenOrientation=\"sensorLandscape\" android:configChanges=\"orientation|keyboardHidden|screenSize\"', 1)
                      open(m, 'w', encoding='utf-8').write(txt)
                  except: pass
          "

      - name: Make Gradle executable
        run: |
          find . -name "gradlew" -exec chmod +x {} \;

      - name: Check Gradle version
        run: |
          ./gradlew --version || true

      - name: Build APK
        run: |
          ./gradlew assembleDebug --no-daemon --stacktrace

      - name: Find generated APK
        run: |
          echo "Searching for APK..."
          find . -type f -path "*/build/outputs/apk/debug/*.apk" -print

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: DHIQAR-TV-FINAL
          path: "**/build/outputs/apk/debug/*.apk"
          if-no-files-found: error
