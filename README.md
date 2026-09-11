name: Build Android APK

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

    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v3

    - name: Extract Project ZIP
      run: |
        unzip -o DHIQAR-TV-V4-PRO.zip
        if [ -d "DHIQAR-TV-V4-PRO" ]; then
          cp -rn DHIQAR-TV-V4-PRO/* . || true
        fi

    - name: Inject PIN Code System & Fix UI
      run: |
        python3 -c "
        import glob, re

        # --- إعدادات الرمز والسيرفر ---
        PIN_CODE = '2027'
        SERVER_URL = 'http://vod4k.cc:80'
        USERNAME = '2142771292105495'
        PASSWORD = '2142771292105495'

        # 1. التعديل البرمجي لملفات الكود (Java / Kotlin) لتفعيل الرمز 2027
        source_files = glob.glob('**/*.java', recursive=True) + glob.glob('**/*.kt', recursive=True)
        for path in source_files:
            if 'build/' in path:
                continue
            with open(path, 'r', encoding='utf-8', errors='ignore') as f:
                code = f.read()

            # إدراج التفاعل مع رمز PIN 2027 عند جلب النصوص
            if 'getText()' in code or 'toString()' in code:
                # استبدال الإدخال بالبيانات الحقيقية إذا كان المدخل هو 2027
                new_code = re.sub(
                    r'(String\s+([a-zA-Z0-9_]+)\s*=\s*[^;]*getText\(\)\.toString\(\)[^;]*;)',
                    rf'\1\n        if (\2 != null && (\2.trim().equals(\"{PIN_CODE}\") || \2.trim().contains(\"{PIN_CODE}\"))) {{ \2 = \"{SERVER_URL}\"; }}',
                    code
                )
                if new_code != code:
                    with open(path, 'w', encoding='utf-8') as f:
                        f.write(new_code)
                    print(f'Injected PIN logic to: {path}')

        # 2. إصلاح الواجهة والشعار وحجم التمرير (activity_main.xml)
        layouts = glob.glob('**/res/layout/activity_main.xml', recursive=True)
        for path in layouts:
            with open(path, 'r', encoding='utf-8') as f:
                content = f.read()

            # تحسين إرشاد الخانة الأولى ليظهر أنه يمكن كتابة 2027
            content = content.replace('android:hint=\"http://', 'android:hint=\"أدخل الرمز 2027 أو الرابط: http://')

            # ضبط حجم اللوجو والشعار
            if '<ImageView' in content:
                content = re.sub(
                    r'(<ImageView[^>]*?)(/?>)',
                    lambda m: m.group(1) + (' android:adjustViewBounds=\"true\" android:scaleType=\"fitCenter\"' if 'scaleType' not in m.group(1) else '') + m.group(2),
                    content
                )

            # تغليف الشاشة بـ ScrollView للتمرير
            if 'ScrollView' not in content:
                clean_content = content.replace('<?xml version=\"1.0\" encoding=\"utf-8\"?>', '').strip()
                content = f'''<?xml version=\"1.0\" encoding=\"utf-8\"?>
        <ScrollView xmlns:android=\"http://schemas.android.com/apk/res/android\"
            xmlns:app=\"http://schemas.android.com/apk/res-auto\"
            xmlns:tools=\"http://schemas.android.com/tools\"
            android:layout_width=\"match_parent\"
            android:layout_height=\"match_parent\"
            android:fillViewport=\"true\"
            android:scrollbars=\"vertical\">

            {clean_content}

        </ScrollView>'''

            with open(path, 'w', encoding='utf-8') as f:
                f.write(content)
            print(f'Updated Layout: {path}')

        # 3. ضبط الـ Manifest لدعم الكيبورد
        manifests = glob.glob('**/AndroidManifest.xml', recursive=True)
        for mpath in manifests:
            with open(mpath, 'r', encoding='utf-8') as f:
                mcontent = f.read()
            if 'windowSoftInputMode' not in mcontent:
                mcontent = mcontent.replace('<activity', '<activity android:windowSoftInputMode=\"adjustResize\"', 1)
                with open(mpath, 'w', encoding='utf-8') as f:
                    f.write(mcontent)
                print(f'Patched manifest: {mpath}')
        "

    - name: Build Debug APK
      run: |
        if [ -f "./gradlew" ]; then
          chmod +x gradlew
          ./gradlew assembleDebug
        else
          gradle assembleDebug
        fi

    - name: Upload APK
      uses: actions/upload-artifact@v4
      with:
        name: DHIQAR-TV-v4-PRO
        path: "**/build/outputs/apk/debug/*.apk"
