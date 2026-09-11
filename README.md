name: Build Cyber Pro APK Clean

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

    - name: Unzip & Find Project Root
      run: |
        sudo apt-get update && sudo apt-get install -y unzip
        for f in *.zip; do
          [ -e "$f" ] || continue
          unzip -o "$f" -d extracted_app || true
        done
        
        PROJECT_DIR=$(find . -name "build.gradle" -o -name "build.gradle.kts" | head -n 1 | xargs dirname)
        if [ -z "$PROJECT_DIR" ]; then
          PROJECT_DIR="."
        fi
        echo "PROJECT_DIR=$PROJECT_DIR" >> $GITHUB_ENV

    - name: Apply Safe Cyber Customizations
      run: |
        python3 -c "
        import glob, re, os

        proj_dir = os.environ.get('PROJECT_DIR', '.')

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

        for c in glob.glob(f'{proj_dir}/**/res/values/colors.xml', recursive=True):
            if 'build/' not in c:
                try: open(c, 'w', encoding='utf-8').write(cyber_colors)
                except: pass

        for m in glob.glob(f'{proj_dir}/**/AndroidManifest.xml', recursive=True):
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

    - name: Build APK with Auto-Repair
      run: |
        cd $PROJECT_DIR
        if [ -f "./gradlew" ]; then
          chmod +x gradlew
          ./gradlew assembleDebug --no-daemon
        else
          gradle wrapper
          chmod +x gradlew
          ./gradlew assembleDebug --no-daemon
        fi

    - name: Upload APK Artifact
      uses: actions/upload-artifact@v4
      with:
        name: DHIQAR-TV-CYBER-FINAL
        path: "**/build/outputs/apk/debug/*.apk"
